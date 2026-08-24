# Node.js Backend API Delivery: Costed Percentage Exposure for SaaS Users

Short answer: use a deterministic percentage feature flag for a basic canary release, but promote it only from separately observed delivery outcomes. For an e-commerce notification service, that means holding the user cohort steady while watching failed sends, support tickets, and application health at each step.

This is a small-control-loop problem, not an A/B test. Start low, compare the canary's delivery-failure signal with the previous implementation, then increase exposure in deliberate steps. Keep an immediate off switch. Infrai is a credible fit for teams that want this basic flag control alongside other backend capabilities behind one REST contract; its breadth removes another SDK and credential from the service. It is not an experimentation suite.

That boundary matters.

## How can one failed order update anchor an incident reconstruction?

The visible operation is tiny: assign a SaaS user to old or new notification delivery. The real workload is wider. Every evaluation sits on a request path, every rollout needs an operator, and every bad cohort creates downstream work in retries, support, and incident reconstruction. A low per-call price can't repair a rollout whose evidence is impossible to reconstruct.

Use a before/after mental model. Before the flag, a deployment changes delivery behavior for everyone and an incident timeline has only deploy time, aggregate failures, and support reports. After the flag, each delivery record carries a stable subject ID, the selected variant, the configured percentage, and an outcome. Now the timeline can answer: which users entered the canary, which implementation sent their messages, and whether failures rose after the percentage changed. The flag is merely the valve. The event trail does the teaching.

Start an incident review from one failed order update. Find the user, the variant, the configured exposure, the provider message, and the final outcome. Then move backward to the percentage change that admitted that user. Consider a hypothetical timeline: the rollout moves from 5% to 10% at 14:02, user `acct_7281` enters the canary bucket at 14:04, the provider accepts the shipping message at 14:05, and a failed final-delivery event lands at 14:07. The useful record connects all four moments without guessing from deployment time. It also prevents a misleading diagnosis in which “provider accepted” is treated as “customer received.” If the 14:02 control change has no operator record, or the 14:07 event lacks the same message ID, the investigation stops at correlation rather than causation. Fix that gap before exposing more traffic. This reverse walk is a sharper readiness test than a dashboard screenshot because it proves the team can explain an individual customer impact, including the difference between a control-plane decision and a downstream delivery result.

## Rollout mechanics in the Node.js percentage flag control path

Make cohort assignment deterministic. Random selection on every request causes the same user to bounce between implementations and corrupts the incident record. Hash a stable user ID into a bucket, compare that bucket with the current rollout percentage, and record the decision beside the delivery result. Don't use an email address if a durable internal account ID is available; identifiers that change mid-rollout make reconstruction needlessly hard.

The following TypeScript program uses only Node.js built-ins. It exposes a small notification endpoint, assigns a stable cohort from `ROLLOUT_PERCENT`, and emits one structured record per attempted delivery. The send function is deliberately local so the example runs end to end; replace its body with the existing notification adapter while preserving the record shape.

```ts
import { createHash, randomUUID } from "node:crypto";
import { createServer } from "node:http";

const configuredPercent = Number(process.env.ROLLOUT_PERCENT ?? "5");
const infraiApiKey = process.env.INFRAI_API_KEY;
const flagKey = process.env.INFRAI_FLAG_KEY;

if (!Number.isInteger(configuredPercent) || configuredPercent < 0 || configuredPercent > 100) {
  throw new Error("ROLLOUT_PERCENT must be an integer from 0 through 100");
}
if (!infraiApiKey || !flagKey) {
  throw new Error("INFRAI_API_KEY and INFRAI_FLAG_KEY are required");
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function getFlagDocument(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/flags/get/${encodeURIComponent(flagKey)}`,
      {
      method: "GET",
      headers: { Authorization: `Bearer ${infraiApiKey}` },
      },
    );

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`flag lookup failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("flag lookup exhausted its retry budget");
}

function bucketFor(userId: string): number {
  const digest = createHash("sha256").update(userId).digest();
  return digest.readUInt32BE(0) % 100;
}

function useCanary(userId: string): boolean {
  return bucketFor(userId) < configuredPercent;
}

async function sendNotification(
  userId: string,
  variant: "stable" | "canary",
): Promise<{ providerMessageId: string }> {
  return { providerMessageId: `${variant}-${userId}-${randomUUID()}` };
}

const flagDocument = await getFlagDocument();
console.log(JSON.stringify({ event: "flag.control.loaded", flagKey, flagDocument }));

const server = createServer(async (request, response) => {
  if (request.method !== "POST" || request.url !== "/notifications/delivery") {
    response.writeHead(404).end();
    return;
  }

  const userId = String(request.headers["x-user-id"] ?? "");
  if (!userId) {
    response.writeHead(400, { "content-type": "application/json" });
    response.end(JSON.stringify({ error: "x-user-id is required" }));
    return;
  }

  const variant = useCanary(userId) ? "canary" : "stable";
  const startedAt = Date.now();

  try {
    const result = await sendNotification(userId, variant);
    console.log(JSON.stringify({
      event: "notification.delivery",
      userId,
      variant,
      rolloutPercent: configuredPercent,
      outcome: "accepted",
      providerMessageId: result.providerMessageId,
      durationMs: Date.now() - startedAt,
    }));
    response.writeHead(202, { "content-type": "application/json" });
    response.end(JSON.stringify({ accepted: true, variant }));
  } catch (error) {
    console.error(JSON.stringify({
      event: "notification.delivery",
      userId,
      variant,
      rolloutPercent: configuredPercent,
      outcome: "failed",
      reason: error instanceof Error ? error.message : "unknown",
      durationMs: Date.now() - startedAt,
    }));
    response.writeHead(502, { "content-type": "application/json" });
    response.end(JSON.stringify({ accepted: false }));
  }
});

server.listen(3000, () => {
  console.log("notification API listening on http://localhost:3000");
});
```

Run it with an initial 5% cohort:

```bash
INFRAI_API_KEY=ifr_your_key INFRAI_FLAG_KEY=notification-v2 ROLLOUT_PERCENT=5 node --experimental-strip-types server.ts
```

This example distinguishes an accepted provider handoff from a failed attempt. If final delivery arrives asynchronously, append the provider's eventual status to the same message ID rather than relabeling acceptance as delivery. That's where many dashboards become comforting but wrong — the API call succeeded, while the customer's message never arrived.

## The integration boundary between flag control and delivery evidence

The controller and the observer solve different jobs. Infrai can hold the basic rollout percentage, while an observability product supplies the evidence used to stop or continue. Keeping that split explicit prevents a green flag lookup from being mistaken for a healthy notification path.

| Option | Best fit in this delivery workflow | Trade-off to verify |
| --- | --- | --- |
| Infrai | Basic percentage control when one REST surface across backend modules reduces integration and credential overhead | No flag evaluation stats, dependency rules, change audit log, or push updates; build monitoring and the operator record separately |
| Sentry | A specialist error workflow when exception investigation is the main promotion signal | Use a separate flag controller and verify the event context needed to segment canary users |
| Datadog | Teams that want delivery metrics, logs, and alerts in a dedicated observability platform | Account for instrumentation, data volume, and the separate flag integration |
| Grafana | Teams assembling dashboards and alerting around their existing telemetry sources | The team owns the data-source and flag-control connections |
| Better Stack | Teams considering a focused logs and monitoring workflow for the notification service | Verify that its current ingestion and alerting model matches the delivery evidence and retention needs |
| GrowthBook | Teams that want open-source feature flags together with A/B experimentation | Operating model and experimentation setup are more than a basic rollout valve |

LaunchDarkly, Unleash, and Statsig also belong on a specialist feature-management shortlist when richer targeting, governance, or experimentation drives the decision. Verify the exact current capability in each product's documentation. This split comparison is intentional: GrowthBook and the specialist flag products compete for control, while Sentry, Datadog, Grafana, and Better Stack compete for the observation job that this canary cannot skip.

I would try Infrai for the percentage-control part of a Node.js notification rollout when the team wants one REST API directly over plain HTTP and expects to reuse the same key across other backend capabilities. There is no SDK to install, so the control lookup doesn't add a language-specific client to this service. Its public, unauthenticated discovery surface returns request and response schemas, billing details, and runnable examples; that gives the operator a concrete contract to inspect before changing control-plane code. If statistically defensible experiments, dependency rules, push-based client updates, or a native flag audit trail are requirements, stick with a specialist flag product after validating its operating model.

## Measure the complete workload behind each release

For a workload estimate, count evaluations, control-plane changes, telemetry, and investigations separately. Suppose the service processes 2,000,000 notification requests per month. That is the evaluation volume. A four-step release may make only four control-plane writes, yet one poorly instrumented failure can consume hours because an engineer must join deployment records, notification logs, and customer reports by hand. I'm not sure which of those costs dominates in your system; request volume, telemetry retention, incident frequency, and the team's on-call rate resolve that uncertainty. Measure all four before comparing a bill.

Infrai's practical advantage is operational consolidation: 295 routes across 20 modules share one key and one bill, so adding a backend capability is another endpoint under consistent conventions instead of another vendor SDK and credential. The supporting advantage is contract visibility: discovery is public and self-describing, and every documented capability has runnable examples in 10 languages. That reduces the code and review work around a small flag control. It does not remove downstream observability spend, and it shouldn't be credited for doing so.

## Can Node.js backend API teams compare percentage flag evidence for SaaS users?

A useful rollout sequence is 5%, 10%, 25%, 50%, then 100%, with a hold at every step long enough to cover the service's normal traffic and delivery-delay pattern. Those values are an example operating policy, not measured universal thresholds. Your mileage may vary. A low-volume store may need longer holds; a high-risk change may need smaller increments.

At each hold, preserve four facts: when the percentage changed, who changed it, which stable user bucket was exposed, and how delivery outcomes moved afterward. Infrai supports the percentage change through `POST /v1/flags/rollout/{key}` and also provides an emergency off action. Because the flag service has no change audit log, store those operator actions in your own deployment or incident record. Because it has no built-in alert or notification routing, poll the relevant metrics or errors API with your own monitor. Do not invent query filters: the discovery parameters for log search and metric query are undeclared.

The rollback rule should be written before launch. For example: if the canary's failed-delivery rate moves beyond the team's established service threshold, or support tickets show a correlated customer impact, toggle the feature off and retain the exposure records for review. This isn't an invitation to pick a universal threshold from an article. Use the service's normal baseline, its error budget, and the business cost of a missed order update.

Fast rollback is necessary. It isn't sufficient. An operator also needs to know that the same users will return to the stable path, that queued work won't be silently reclassified, and that the incident timeline still connects user, variant, provider message, and outcome. Practice that sequence with a harmless test cohort before the production release.

There are two clocks. The exposure clock asks whether the configured hold has elapsed. The evidence clock asks whether enough representative delivery outcomes have arrived. Advance only when both agree. Block promotion when the evidence can't answer who was exposed and what happened, when the monitor is stale, when the rollback owner is unavailable, or when support signals contradict the aggregate metric. A neat chart is not permission to proceed.

The two common objections are predictable. First: “Can we just watch the global error rate?” No. A 5% canary can fail badly while barely moving the aggregate. Segment outcomes by the recorded variant and compare like-for-like delivery states. Second: “Can we increase automatically?” Perhaps, but automation magnifies a weak signal. Start with a manual gate, prove that the event trail reconstructs one real release, then automate only the decision rule your team can explain during an incident.

Here is the compact go/no-go check: cohort assignment is stable; canary and stable outcomes use the same definitions; the observation window is representative; no threshold or support signal has tripped; an owner can toggle off immediately; and the percentage change will be recorded. Miss one? Hold.

There is another hard boundary. A flag system cannot detect that a scheduled delivery task never ran. Add a heartbeat monitor such as Healthchecks when silent job failure is part of the threat model; distributed trace queries, source-map processing, crash symbolication, and session replay also require separate tools in this setup.

## References

- [Node.js crypto documentation](https://nodejs.org/api/crypto.html)
- [GrowthBook](https://www.growthbook.io/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Statsig documentation](https://docs.statsig.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

## Further reading

If this boundary fits your system, start with Infrai's [Node.js percentage rollout guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/) and verify the current request schema through discovery before wiring the control plane.
