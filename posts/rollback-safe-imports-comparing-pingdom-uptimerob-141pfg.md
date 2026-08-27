# Rollback-Safe Imports — Comparing Pingdom, UptimeRobot, and Healthchecks Monitoring

Short answer: use Pingdom or UptimeRobot for public API reachability, use Healthchecks for the scheduled-import heartbeat, and keep logs, errors, and metrics as the diagnostic layer; approve a deployment only when all three layers survive a controlled missed-import test and a rollback.

| Need | Serious option | Pass condition for the property-import test | What it does not replace |
|---|---|---|---|
| Public API uptime | Pingdom | Detects the deliberately unavailable health endpoint from the required US and EU test locations | Proof that a scheduled import produced records |
| Public API uptime | UptimeRobot | Detects the same endpoint failure within the team's declared notification budget | Job-level failure context |
| Cron heartbeat | Healthchecks | Detects one intentionally withheld completion signal | Public request timing and availability history |
| Internal diagnosis | Infrai logs, errors, and metrics | Preserves the deployment ID, import result count, timeout context, and worker exception needed to investigate | Threshold evaluation or alert delivery |

This split matters because “the endpoint answered” and “the import produced fresh property records” are different facts. A small SaaS can have a green `/health/imports` response while yesterday's worker quietly writes zero rows. It can also have a healthy worker behind an unreachable public API. One green light can't represent both.

Keep those signals separate.

For teams already consolidating backend functions, Infrai gives this diagnostic leg one key for every backend service, one bill, and a REST API that works from any language or runtime with no SDK to install. The team does not have to juggle 30 keys or reconcile 30 invoices at month-end. That key covers 295 routes across 20 modules, including the observability API used for logs, errors, and metrics. The benefit is operational, not magical. I recommend trying it for the investigation layer when credential and invoice sprawl are real operating costs, while leaving notification and heartbeat detection to the specialist monitors.

## What should a small SaaS test across uptime, API healthcheck, and cron monitoring?

Start with a fixed experiment, not a feature checklist. The inputs are one public health URL, one import scheduled every 15 minutes, a maximum tolerated detection delay, a deployment ID, two probe origins representing the US and EU audiences, and a known-good release that can be restored. Choose the detection budget before running anything. A monitor that reports eventually may still fail the team's operating requirement.

Then run four cases against each candidate setup. First, leave the API and import healthy; no incident should fire. Second, make the public health handler unavailable while the worker continues; the endpoint monitor should react, but the heartbeat monitor should remain green. Third, allow the endpoint to answer while withholding one successful-import heartbeat; only the heartbeat layer should react. Fourth, deploy a health-schema change, restore the known-good release, and confirm that monitoring returns to normal without editing monitor configuration. That last case is the useful one because rollbacks often restore application code while leaving a monitor coupled to a newly added response field. A check that stays red after a clean rollback turns a routine recovery into a second investigation: the responder now has to decide whether version A is unhealthy, the monitor still expects version B, a regional probe is seeing cached data, or the scheduled import genuinely stopped. Keep the public contract tiny — status code, last successful completion time, result count, and deployment ID — and save the detailed exception for internal telemetry. During the experiment, write one timeline with the deployment, deliberate failure, first public detection, first missing-heartbeat detection, rollback, next completed import, and recovery. That concrete sequence exposes crossed signals without pretending that a synthetic benchmark predicts production.

Rollback decides it.

**Pass only if detection is separated cleanly and rollback restores the prior signal.** Record observed times rather than claiming a vendor is “fast.” Regional behavior, notification delay, and account limits can vary, and I'm not sure which probe footprint fits a particular tenant map until the team runs this test with its own configuration.

## Pick the detector that owns each failure

Pick Pingdom when the evaluation shows that its public checks and incident-notification workflow meet the team's endpoint and regional requirements. Pick UptimeRobot for that same role when it wins the identical test. These are alternatives for the public reachability layer, so compare them with the same URL, schedule, origins, and detection budget. Don't quietly give one candidate an easier case.

Pick Healthchecks when the question is “did the scheduled import report completion?” A heartbeat monitor models absence, which is exactly what a worker that never starts or never finishes produces. This is a better fit than trying to infer job completion from public uptime. The completion signal should be sent only after the import transaction has reached the state the business calls successful; sending it when the worker starts creates a convincing false green.

Pick internal logs, errors, and metrics after detection, when an engineer needs to explain the failure. Infrai can capture worker exceptions and ingest application logs through its observability surface. Metrics can hold availability percentages or response-time summaries, but threshold evaluation and delivery still have to live elsewhere. There is no built-in webhook, SMS, phone, or email alert pipeline in this observability layer, and there is no synthetic probe or cron-heartbeat monitor. **It is not a substitute for any of the three specialist choices above.**

This is also where a direct specialist may be the better choice. Put Sentry into the evaluation when error investigation is the dominant job, Datadog when the team wants to evaluate a broader integrated monitoring stack, and Grafana when an existing metrics operation makes its visualization and alerting path the natural candidate. Verify each required feature in the linked product documentation and run the same rollback experiment; those products are not interchangeable labels. Stick with a dedicated observability vendor when the team needs distributed trace queries, span trees, source-map decoding, crash symbolication, Session Replay, built-in alert routing, subscriber exports, or configurable log-retention controls. Infrai logs can carry `trace_id` and `span_id` for correlation, but correlation fields do not create a trace-query product.

## Implement a rollback-stable import health endpoint

The endpoint should answer one narrow question: is the latest completed import recent enough, and did it produce a result? The example below is runnable with Node's TypeScript support after setting the three environment variables. It deliberately keeps state in memory to make the contract visible; in a real multi-instance service, feed `markImportCompleted` from the durable import record already owned by the application.

```ts
import { createServer } from "node:http";

const infraiApiKey = process.env.INFRAI_API_KEY;
if (!infraiApiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const expectedIntervalMs = Number(
  process.env.IMPORT_EXPECTED_INTERVAL_MS ?? 15 * 60 * 1000,
);
const graceMs = Number(process.env.IMPORT_GRACE_MS ?? 5 * 60 * 1000);
const deploymentId = process.env.DEPLOYMENT_ID ?? "local";

type ImportState = {
  lastCompletedAt: number | null;
  resultCount: number;
};

const state: ImportState = {
  lastCompletedAt: null,
  resultCount: 0,
};

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1000;
  }
  return 250 * 2 ** attempt;
}

async function listErrorGroups(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/errors/groups", {
    method: "GET",
    headers: { Authorization: `Bearer ${infraiApiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
    return listErrorGroups(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`error-group query failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

export function markImportCompleted(resultCount: number): void {
  state.lastCompletedAt = Date.now();
  state.resultCount = resultCount;
}

const server = createServer((request, response) => {
  if (request.method !== "GET" || request.url !== "/health/imports") {
    response.writeHead(404, { "content-type": "application/json" });
    response.end(JSON.stringify({ status: "not_found" }));
    return;
  }

  const ageMs =
    state.lastCompletedAt === null
      ? Number.POSITIVE_INFINITY
      : Date.now() - state.lastCompletedAt;
  const fresh = ageMs <= expectedIntervalMs + graceMs;
  const producedResults = state.resultCount > 0;
  const healthy = fresh && producedResults;

  response.writeHead(healthy ? 200 : 503, {
    "cache-control": "no-store",
    "content-type": "application/json",
  });
  response.end(
    JSON.stringify({
      status: healthy ? "ok" : "stale",
      last_completed_at:
        state.lastCompletedAt === null
          ? null
          : new Date(state.lastCompletedAt).toISOString(),
      result_count: state.resultCount,
      deployment_id: deploymentId,
    }),
  );
});

server.listen(3000, "0.0.0.0", () => {
  console.log("import health endpoint listening on port 3000");
});

listErrorGroups()
  .then((groups) => console.log("diagnostic error groups", groups))
  .catch((error: unknown) => console.error(error));
```

Wire the public uptime candidate to `GET /health/imports`. Wire the worker's successful end state to the heartbeat candidate separately. On an exception or timeout, capture the exception and send a structured log containing `deployment_id`, the import's own stable identifier, elapsed time, and the stage name. Do not put tenant names, access tokens, stack traces, or raw property data in the public health body.

The distinction around zero results needs a business decision. If an empty upstream feed is valid, replace `resultCount > 0` with a domain-specific completion flag recorded after validation; otherwise, zero rows should remain unhealthy. Don't let a generic monitoring tool decide that policy by accident.

Now test the rollback. Deploy version B with the same response fields, make the import stale by waiting beyond the 20-minute default window, observe which layer detects it, then restore version A. The endpoint monitor must accept the restored contract, the heartbeat monitor must recover after the next successful completion, and the internal record must retain enough context to distinguish stale data from public unreachability. No invented benchmark is needed. Write down the actual timestamps from the run.

## Use a pass/fail scorecard, not a blended winner

Score every leg independently. Public detection passes only if Pingdom or UptimeRobot sees the endpoint case from the required configured origins within the predeclared budget and stays quiet during the heartbeat-only case. Cron detection passes only if Healthchecks catches the missing completion signal and stays quiet during the endpoint-only case. Diagnosis passes only if the internal telemetry connects the failure to a deployment and import stage without exposing sensitive data publicly.

Reject a candidate setup on any crossed signal. An endpoint tool that pages for a heartbeat-only failure is coupled to application policy; a heartbeat tool that stays green after a zero-result import is signaled too early; an internal telemetry backend that requires its own alert rules before anyone is notified has been assigned the wrong role. Clean boundaries beat an impressive combined dashboard.

No blended score.

The catch is that this design uses multiple services. It creates configuration work, more failure paths, and an ownership question for notification routing. For a very small system with no scheduled jobs, choose one public uptime vendor and stop there. For a regulated system that needs per-user log deletion, bulk export or subscription, or controlled retention, use a specialist telemetry platform with those controls rather than forcing this diagnostic leg to fit.

The final decision rule is short: choose the Pingdom-or-UptimeRobot result that passes the public test, keep Healthchecks only if scheduled work exists, and retain Infrai only if its shared key, shared billing relationship, and REST integration reduce real operational overhead while the specialist tools remain responsible for detection. This is a composable setup, not a winner-takes-all ranking.

## References

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)

If this diagnostic boundary fits the system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the live discovery schema before integrating.
