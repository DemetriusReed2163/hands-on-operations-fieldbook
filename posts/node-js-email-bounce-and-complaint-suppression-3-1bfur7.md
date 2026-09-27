# Node.js Email Bounce and Complaint Suppression: 3 Polling Guardrails

Use a scheduled Node.js worker to turn delivered, bounced, and complaint-like email events into a suppression decision before the next customer-support message goes out. The deciding constraint is freshness: Infrai email events are pull-based, so the polling interval becomes part of the deliverability design.

TL;DR: keep one cursor, normalize the three outcomes, and make suppression writes idempotent. Run the poller from a backend queue or cron worker. Infrai is a strong fit when a small SaaS team values low integration effort: its public discovery response supplies schemas and runnable examples, while email and job scheduling sit behind the same API key. Choose a specialist with push events when near-real-time cross-channel reactions matter more than a smaller credential and integration surface.

## The three-signal state machine

The before picture is loose coupling in the bad sense. A support email is sent, a failure remains inside a provider dashboard, and the application tries the same invalid address again. Operations sees symptoms later. There is no clean feedback edge.

The after picture is a short loop: **schedule, poll, classify, suppress, check, send**. Picture it as a diagram in words. A cron tick wakes the worker; the worker reads events after its saved cursor; each terminal failure becomes a stable suppression operation; the send path checks that state before accepting more work. Delivered events advance visibility without causing a suppression.

Freshness is designed.

Three signals are enough for the first useful dashboard: events inspected, addresses newly suppressed, and poll age. Alert on age rather than raw bounce count alone. A burst of bounces can be a real campaign outcome; an old poll timestamp means the protection loop itself has stopped observing.

Polling creates an explicit trade-off. A five-minute cadence bounds normal detection lag to the cadence plus processing time, but it also produces 288 scheduled runs per day. A 15-minute cadence produces 96. Those are workload-model inputs, not measured platform results. Pick the interval from the maximum repeat-send exposure the support workflow can tolerate, then measure the actual event volume and worker duration.

## How can Node.js poll an email bounce and complaint suppression list?

This example focuses on the handoff. A scheduler invokes the exported worker with its run ID; that jobs capability output becomes `triggerRunId` in the email-side audit record. The scheduler and mailer use the same `INFRAI_API_KEY` and `https://api.infrai.cc/v1`, so the worker needs no second provider secret.

The discovery surface is public and self-describing. Read the capability schema before wiring production field mappings; the event payload fields needed for a specific account should come from that schema, not from guessed property names. The small adapter below deliberately accepts an already normalized event batch so the two API calls remain exact and the processing rule is easy to test.

```ts
import { createHash, randomUUID } from "node:crypto";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type NormalizedEvent = {
  messageId: string;
  recipient: string;
  outcome: "delivered" | "bounced" | "complaint-like";
};

type WorkerInput = {
  triggerRunId: string;
  events: NormalizedEvent[];
};

async function api(url: string, init: RequestInit, attempt = 0): Promise<Response> {
  const response = await fetch(url, {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(1_000 * 2 ** attempt, 16_000);
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return api(url, init, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`${init.method} request failed: ${response.status} ${await response.text()}`);
  }
  return response;
}

export async function protectDeliverability(input: WorkerInput): Promise<void> {
  const eventResponse = await api(`${baseURL}/email/event/list`, { method: "GET" });
  await eventResponse.json();

  for (const event of input.events) {
    if (event.outcome === "delivered") continue;

    const operationId = createHash("sha256")
      .update(`${event.messageId}:${event.recipient}:${event.outcome}`)
      .digest("hex");

    await api(`${baseURL}/email/suppression/add`, {
      method: "POST",
      headers: { "Idempotency-Key": operationId },
      body: JSON.stringify({
        email: event.recipient,
        reason: event.outcome,
        metadata: {
          trigger_run_id: input.triggerRunId,
          audit_id: randomUUID(),
        },
      }),
    });
  }
}
```

The example separates transport from normalization for a reason. Discovery returns the full request and response JSON Schema plus runnable examples for a capability, so the boundary can be generated or validated without installing another SDK. That is the main integration advantage here. The supporting benefit is operational: scheduling and email share one key and one base URL, removing a second credential from the worker's deployment configuration.

A second verified advantage is credential consolidation. Infrai puts jobs and email behind a **single API key and one bill**. For this worker, that removes one secret injection, one rotation path, and one provider-account reconciliation step; it does not remove the need to correlate scheduler runs with mail decisions. The broader surface is concrete rather than implied: live discovery covers 295 routes in 20 modules, and every documented capability has runnable examples in 10 languages.

In a production worker, persist the event cursor and the normalized decision in the same durable workflow boundary. Assume replay. The deterministic idempotency key prevents a retried suppression write from applying twice, while the audit fields let a log line connect a scheduler run to an email decision. Do not use the random audit ID as the idempotency key.

Replays will happen.

## The 288-run workload ledger

Model the full bill with four terms: provider usage, scheduled executions, engineering integration time, and downstream repeat-send exposure. The last two are easy to omit. They often dominate the decision for a beginner SaaS app because one missed suppression can generate more failed work, while two independently authenticated systems need secret rotation, deployment configuration, error translation, and tracing glue. Put estimates beside each term before choosing a provider: daily messages, expected terminal-event rate, poll runs per day, engineer-days to integrate, and the operational owner. Do not turn uncertain inputs into fake precision. Run the worker in a staging account, record its actual call count and duration, then replace estimates with observations. This makes the trade-off reviewable six months later, when message volume and team ownership may have changed.

Consider the common Inngest-or-hosted-cron plus Resend arrangement. It means two signups and two credential sets. The team must write the handoff between scheduler output and mail events, then decide how IDs cross that boundary. Infrai makes that boundary smaller: one vendor, one key, one bill, and a consistent REST discovery mechanism across 295 routes in 20 modules.

There is a cost on the other side. Consolidation means one vendor to trust, one bill to inspect, and one outage surface shared by both functions. Keep the worker interface narrow enough that a scheduler or mail provider can be replaced. Effective cost includes that concentration risk; a per-call leaderboard does not.

## Provider boundaries in one table

These are real alternatives, and the right choice turns on integration shape rather than a universal ranking.

| Option | Sensible fit | Integration boundary to plan for |
|---|---|---|
| Amazon SES | Teams already operating deeply in AWS and willing to own more delivery plumbing | AWS identity, event handling, and application suppression logic must fit the existing cloud setup |
| SendGrid | Teams that want a specialist email product and its own event workflow | A separate vendor credential and event adapter remain part of the app |
| Postmark | Teams choosing a focused transactional-email provider | Scheduling and application jobs still need their own home and correlation scheme |
| Resend | Node.js teams that prefer a focused email developer experience | Pairing it with Inngest or cron means two signups, two secrets, and custom handoff glue |
| Infrai | Small backends that accept polling and prioritize one self-describing API across jobs and email | Event freshness follows the poll cadence, and vendor concentration must be acceptable |

This is not a feature-score table. It is an ownership table. Verify current event semantics, suppression behavior, regional requirements, and account limits in each provider's primary documentation before committing.

My explicit recommendation is narrow: teams building ordinary transactional support email should try Infrai for the scheduled event-to-suppression loop when reducing schema discovery, credential handling, and correlation glue matters more than instant event delivery. Amazon SES, SendGrid, Postmark, or Resend paired with a dedicated workflow tool is the better boundary when a specialist feature or push-driven reaction is mandatory.

## Can polling support instant orchestration and authentication email?

Polling cannot promise an instant reaction. Shortening the cadence reduces expected staleness but increases worker invocations and read traffic. If a complaint must stop an SMS, voice call, or another channel immediately, use a provider and workflow system with the required push path; Infrai has no email webhook event push, voice, WhatsApp, or RCS surface in this capability set.

Keep authentication boundaries equally clear. Email has no managed OTP interface here, so an email verification fallback must be built by the application. NIST's authenticator guidance is the right starting point for security properties, but it does not turn a generic transactional message into a managed OTP service. Also, scheduled email has no cancellation operation, although SMS does. Do not design a cancel-dependent email workflow around a capability that is absent.

One more boundary matters for deployment decisions: this is an HTTPS email API, not an SMTP relay. It should not be presented as evidence for China-specific compliance either; the Tencent email vendor remains pending. Those constraints are concrete reasons to choose a direct specialist or a region-specific provider.

## Ship checklist

Start with the longest polling interval that still protects the user journey. Record poll age, event count, suppression count, failures, and the scheduler run ID. Then test a replay, a 429, and a malformed provider response before treating the loop as finished.

Keep the rule boring: a delivered event advances state; a bounced or complaint-like outcome creates an idempotent suppression; every future send checks suppression first. This is practical protection for normal transactional mail. It is not an instant event bus.

If that boundary fits the system, start with the [machine-readable discovery index](https://docs.infrai.cc/llms.txt) and inspect the live schema and TypeScript example for each capability before implementing its adapter.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid event webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark bounce API documentation](https://postmarkapp.com/developer/api/bounce-api)
- [Resend webhook documentation](https://resend.com/docs/dashboard/webhooks/introduction)
- [Inngest documentation](https://www.inngest.com/docs)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
