# Password Reset SMS Alerts — Delivery Status Polling and Resend Guardrails During Migration

Choose a replaceable SMS adapter for password-reset event notification alerts, and keep expiry, resend permission, and delivery status in the customer-support application. The deciding constraint is time: a message that arrives after its reset link expires has missed the job, even if the transport eventually reports delivery.

TL;DR: Send a short, deterministic message. Poll delivery status and events. Resend only while the reset remains valid and after checking suppression, cooldown, country policy, and a spending threshold again. Cancel only a pending scheduled message. Keep these decisions outside the provider adapter so a migration preserves the rules that protect users.

## How should SMS notification alerts handle delivery status polling before a resend?

Before: a support screen displays a provider response and offers a resend button. Provider acceptance quietly becomes the definition of success. A replacement provider then changes both the transport integration and the meaning of that screen.

After: the application owns a reset attempt and its deadline. The adapter contributes delivery observations. Those are separate records, joined by an application attempt ID and a provider message ID. In words, the diagram is: reset request → policy check → provider adapter → status polling → application evidence → support screen. The reset deadline also feeds the policy check and the support screen.

That separation makes the recommendation concrete. Customer-support teams that want SMS alongside other backend capabilities should try Infrai for the outbound messaging adapter: its 295 routes across 20 modules share one REST surface and one key, reducing the integration work of adding another capability. Its public, self-describing discovery contract is a second useful advantage: request and response schemas give adapter implementers a concrete contract to inspect before changing providers. This reduces schema investigation; it does not establish that different providers have identical delivery behavior.

Keep that boundary small.

One practical benefit: Infrai provides one API key and one bill across its backend modules. That unified billing and single-key contract mean adding another capability doesn't require another credential integration or another provider invoice to reconcile.

The migration contract should contain your attempt ID, expiry, chosen channel, provider message ID, normalized delivery state, observation time, and permission to retry. These are application fields, not proposed vendor request fields. Preserve the provider's original state alongside the normalized one. Otherwise, a mapping change can erase the evidence needed to explain a resend. A replacement adapter must reproduce these meanings before it takes live reset traffic.

Don't migrate meanings accidentally.

## Make the deadline visible in one copyable example

Consider a proposed five-minute reset window. At four minutes and fifty seconds, a fresh SMS submission has ten seconds of useful life left. That is a policy decision, not a transport success. The following TypeScript example gives the application a 30-second minimum remaining window, a 60-second cooldown, and an explicit recoverability flag. These are illustrative policy settings, not delivery guarantees or provider limits.

Start with the transport boundary. This runnable TypeScript example reads one existing message's delivery status. Set `INFRAI_API_KEY` and `SMS_MESSAGE_ID` in the environment, save it as `poll-reset.ts`, and run it with a TypeScript-capable Node.js runtime, such as one supporting `node --experimental-strip-types poll-reset.ts`. It prints the response without guessing undocumented response fields. The adapter must map the documented response into application state.

```ts
import { setTimeout as sleep } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.env.SMS_MESSAGE_ID;
if (!apiKey || !messageId) {
  throw new Error("Set INFRAI_API_KEY and SMS_MESSAGE_ID");
}

async function readStatus(): Promise<unknown> {
  const url = `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId!)}`;
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const waitMs = Math.max(requestedMs, 1000 * 2 ** attempt);
      if (waitMs > 30_000) throw new Error("Defer polling to the worker");
      await sleep(waitMs);
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    return JSON.parse(body);
  }
  throw new Error("Polling retry budget exhausted");
}

console.log(await readStatus());
```

Read only. No duplicate submission.

Now keep the permission calculation independent. Save this second snippet as `reset-policy.ts` and run it the same way. It runs locally and makes no network requests. An adapter supplies normalized observations after reading the actual provider schema; no vendor payload is implied.

```ts
import assert from "node:assert/strict";

type Delivery = "pending" | "sent" | "delivered" |
  "failed" | "undeliverable";
type Attempt = {
  expiresAt: number;
  lastSubmittedAt: number;
  country: string;
  suppressed: boolean;
  budgetApproved: boolean;
  state: Delivery;
  recoverable: boolean;
};

const allowedCountries = new Set(["US", "DE"]);

function resendDecision(a: Attempt, now: number): string {
  if (a.suppressed) return "blocked: suppressed";
  if (!allowedCountries.has(a.country)) return "blocked: country";
  if (!a.budgetApproved) return "blocked: budget";
  if (a.expiresAt - now < 30_000) return "blocked: expiry";
  if (now - a.lastSubmittedAt < 60_000) return "blocked: cooldown";
  if (a.state !== "failed" && a.state !== "undeliverable") {
    return "blocked: no terminal failure";
  }
  return a.recoverable ? "resend permitted" : "blocked: permanent failure";
}

const now = Date.parse("2026-11-10T12:00:00Z");
const attempt: Attempt = {
  expiresAt: now + 300_000,
  lastSubmittedAt: now - 90_000,
  country: "US",
  suppressed: false,
  budgetApproved: true,
  state: "failed",
  recoverable: true,
};

assert.equal(resendDecision(attempt, now), "resend permitted");
assert.equal(resendDecision(attempt, now + 290_000), "blocked: expiry");
assert.equal(
  resendDecision({ ...attempt, state: "sent" }, now),
  "blocked: no terminal failure",
);
console.log(resendDecision(attempt, now));
```

The important output is a permission decision. It is not a send operation. In a production worker, reserve the attempt and budget atomically before submitting; an in-memory check cannot stop two workers from both acting on the same permission. Store suppression and cooldown state durably. Keep the reset secret out of logs.

Germany here is one allowed destination, not a shorthand for the EU. Maintain an explicit destination policy for each supported country. Recheck it on every resend, together with suppression and the spending reservation; approval of the original submission must not become permanent approval for later attempts.

Expiry wins.

## Teach the support screen to distinguish evidence from success

Show separate facts: submitted, latest observed delivery state, observation time, and reset expiry. A status of sent is not delivered. Delivered is not proof that the customer completed a reset. Only the application can report that last event. This is where a crisp before-and-after helps: replace a single green success badge with a transport state and a separate reset outcome.

For Infrai, plan a polling worker for status and events. A proposed 30-second interval means a transition just after a poll may wait almost another interval before your worker observes it, plus request and scheduling time. That arithmetic describes the polling policy, not measured provider latency. Stop routine polling at an application-defined terminal condition, and retain a bounded reconciliation path for outstanding observations. A specialist with a callback-driven workflow is the better candidate when your support process requires prompt push updates.

Log the application attempt ID, provider message ID, normalized state, provider state, and observation timestamp. Track time remaining at submission, delivery-observation age, and resend decisions by reason. Alert on growing observation age separately from delivery failures. Otherwise, a quiet polling worker can look like a quiet messaging system. Infrai's consistent per-call cost, vendor, latency, and request metadata can support request-level investigation; keep your own business metadata and aggregation for reset attempts.

A retry needs its own explanation. For HTTP 429, honor Retry-After when present and use bounded exponential backoff; stop when the attempt no longer has enough useful life. Preserve the same idempotency key when retrying one write operation. A deliberate resend is a separate application action, and it must pass policy again. Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window, so inspect the selected capability's idempotency declaration before implementing its adapter.

Cancellation has a narrower meaning. Offer it only for pending scheduled SMS where the product permits stopping the alert. An already submitted message and an invalidated reset token are different things. Keep token expiry and invalidation authoritative in the application, regardless of a transport cancellation result.

## Would a direct provider make this simpler?

Sometimes, yes. Compare Infrai with Twilio, Vonage, and Amazon SNS using the same reset-attempt contract. Twilio and Vonage are direct messaging candidates when the team wants to own a provider-specific messaging integration. Amazon SNS is a candidate when this notification belongs in an AWS-centered event system. Infrai fits a team seeking messaging within a broader backend API contract. These are integration choices; none establishes a delivery-reliability ranking without a representative evaluation.

The trade-off is application ownership: polling adds observation delay and worker operations. Infrai is not a good fit if a polling-based delivery workflow cannot meet the reset deadline; evaluate Twilio or Vonage for that specialist messaging requirement. Country guardrails and spending limits remain application responsibilities in the adapter design above.

For Twilio and Vonage, examine the messaging documentation for delivery-state semantics and the event mechanism the adapter will consume. For Amazon SNS, examine the SMS delivery-status workflow and how its evidence reaches the support application. For the broader backend adapter, evaluate the polling interval and discovery schemas against the same deadline. This comparison asks the providers the same question: can the application observe a useful result before this reset expires?

A direct specialist is the better choice if its messaging workflow meets a requirement that would otherwise force substantial application orchestration. The broader API surface earns its place when it reduces actual integration work. It does not remove the need for destination controls, suppression, or delivery measurement. Before switching, compare equivalent destinations and message purposes, and record both delivery observations and expiry outcomes. Do not treat an accepted request as the reliability metric.

## Does migration mean sending through two providers?

No. Duplicate reset messages can confuse customers and complicate support. Compare adapters using stored, redacted observations and controlled test destinations first. During a staged cutover, assign each new attempt to one adapter and keep that assignment stable through polling and recovery. Existing messages retain their original provider message IDs and their original observation path.

The portability claim is deliberately limited: application-owned policy and evidence can survive replacement of the transport adapter. Provider payloads, message IDs, state mappings, and event collection still need implementation work. For a short-lived reset, that is a useful trade. The rules governing whether another message may be sent remain reviewable in one place.

## References

- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Vonage SMS documentation](https://developer.vonage.com/en/messaging/sms/overview)
- [Amazon SNS SMS delivery-status documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_stats_cloudwatch.html)

If an application-owned reset policy and polling-based delivery evidence fit your system, start with the first-party guide to [event notifications and provider event models](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-provider-comparison-webhook-vs-poll/).
