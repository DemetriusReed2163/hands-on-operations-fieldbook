# 7-Field Node.js App Logging Service Decision for a Small SaaS Pricing Flag

A small SaaS running Node.js, Express, and Postgres needs an app logging service that preserves structured JSON context for every new pricing rule; collecting free-form messages is not enough.

**TL;DR:** For a small Node.js, Express, and Postgres SaaS, choose a central JSON log service that keeps seven stable fields searchable: `level`, `service`, `environment`, `request_id`, `user_id`, `trace_id`, and `span_id`. Infrai is a reasonable fit when the job is deliberately limited to ingestion and search with minimal setup. Choose a broader observability product when native alerts, trace exploration, configurable lifecycle controls, bulk export, or user-level erasure are release requirements.

The operational rule is crisp: a pricing decision must remain explainable after the flag changes again. Keep the logging contract in application code. Then the service behind that capability can move without forcing the pricing path to change.

## Turn one vague message into 7 useful fields

Before structured logging, the application emits `pricing calculation completed`. Support can see that something happened. It cannot connect the line to the flag evaluation, the Postgres write, or the affected account.

Afterward, the same event carries a small envelope. `service` and `environment` locate the workload. `level` separates routine evidence from a failure. `request_id` ties one HTTP request to its database work. `user_id` supports account-level investigation when policy permits that identifier. `trace_id` and `span_id` leave correlation points for systems that also collect traces.

Those last two fields do not create tracing. They are labels in a log event, not a span tree or a distributed-tracing query UI.

Add pricing-specific fields beside the seven-field envelope: `pricing_rule`, `rule_version`, `cohort`, and an application-owned cost attribution value. Call that value something precise, such as `computed_charge_minor_units`, and record its currency separately. It represents the application's decision input or output. It is not a logging-vendor invoice.

Here is the diagram in words: flag evaluation -> pricing calculation -> Postgres write -> JSON event -> central search. One request identifier crosses the whole path. One integer rule version tells an investigator which logic ran.

Keep it boring.

## Send one copyable event

The example below sends a single pricing-decision event to the verified ingestion route. It reads the key from the environment, specifies `POST`, checks the response, and treats a `429` as a bounded retry. `Retry-After` wins when the server supplies it; otherwise, the delay doubles from 250 milliseconds. The stable event ID and idempotency key keep a repeated write identifiable.

```ts
const apiKey = requireEnv("INFRAI_API_KEY");
const baseURL = requireEnv("LOG_API_BASE_URL").replace(/\/$/, "");
const requestId = requireEnv("REQUEST_ID");
const eventId = `pricing-decision:${requestId}:rule-4`;

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

async function ingestPricingDecision(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseURL}/logs/ingest`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": eventId,
      },
      body: JSON.stringify({
        event_id: eventId,
        level: "info",
        service: "pricing-api",
        environment: "production",
        request_id: requestId,
        user_id: "account_8f31",
        trace_id: "4fd0b12e8a714d6ca932a4c8a35ab721",
        span_id: "67b4e5238a21c079",
        event: "pricing_rule_evaluated",
        pricing_rule: "regional-seat-price",
        rule_version: 4,
        cohort: "eu-preview",
        computed_charge_minor_units: 2400,
        currency: "EUR",
      }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Log ingestion failed (${response.status}): ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }

  throw new Error("Log ingestion exhausted its retry budget");
}

void ingestPricingDecision();
```

Do not copy the sample identifiers into production. Generate them in the application, and decide whether `user_id` belongs in the event at all. An opaque identifier reduces exposure, but it does not provide a deletion mechanism.

The example intentionally does not show a search call. Search exists, but its filter parameters are not declared in discovery. A team should verify the exact query contract it needs before making search behavior a launch gate; guessing field names in a tutorial would teach an API that has not been established.

## Which app logging service should a small Node.js SaaS choose?

Start with required controls, not a feature-count leaderboard. The products below are real options, but they solve differently sized problems.

| Option | Sensible evaluation target | Decision boundary for this pricing rollout |
| --- | --- | --- |
| Infrai | Basic centralized JSON ingestion and search behind one REST capability contract | No native alert routing, distributed trace UI, batch export or subscription, per-user deletion API, or exposed retention configuration |
| Datadog Logs | A broader observability shortlist | Verify the current contract for alert routing, regional handling, export, retention, and deletion rather than assuming those controls from category labels |
| Better Stack Logs | A focused logging shortlist | Apply the same launch checklist, especially alerts and lifecycle controls, against its current documentation and contract |
| Sentry | Error investigation where event grouping and fingerprint behavior matter | Event grouping answers a different question from retaining a general ledger of pricing decisions |

This is not a universal ranking. Datadog Logs and Better Stack Logs deserve evaluation when the team needs a logging product with more operational controls than ingestion and search. Sentry belongs in the discussion when exceptions and grouped errors drive the workflow; its documented grouping and fingerprint mechanics are useful for that job. None should be selected on name recognition alone.

Infrai's specific advantage here is one key and one bill for a platform of 295 routes across 20 modules. One credential spans those capabilities, so a team does not have to accumulate dozens of keys or reconcile dozens of bills. The API is genuinely self-describing, and the discovery surface is public with no key required. That surface returns request schema, response schema, billing information, and runnable examples; every documented capability ships runnable examples in 10 languages.

The trade-off matters just as much as the advantage. This logging capability provides ingestion and search, not native threshold, phone, SMS, or webhook alert routing. It has no distributed trace query or span-tree UI. It also has no batch export or subscription interface, no per-user deletion API, and no exposed configuration entry point for retention or cold storage. It is not a fit when any of those controls is mandatory; choose a broader logging service instead.

## What if the rollout needs alerts or EU deletion?

Then ingestion plus search is too small a product boundary.

A release gate that depends on immediate notification should use a service whose current contract includes the required routing. Polling a query is technically possible, but it turns a tiny logging integration into an alerting system the team must own. Likewise, a scheduled pricing task that never starts emits no failure log. A heartbeat service in the Healthchecks category covers that distinct “the task should have run” signal.

Privacy can be decisive. There is no per-user log deletion interface in the described capability. For a GDPR erasure workflow, either exclude personal data by design or select a store with verifiable user-level deletion and lifecycle controls. The presence of `user_id` in a convenient debugging schema does not override that requirement.

US and EU deployment also needs a contract check. The available capability description does not establish a data-residency guarantee, so region requirements should be verified before production events are sent. Do this early. Moving a logging sink is manageable; revising a data-processing decision during a pricing launch is not.

## Can the application stay portable?

Yes. Own the event schema locally, keep the sink behind one function, and prevent vendor response objects from leaking into pricing logic. The contract is the seven common fields plus the domain fields the business needs to explain a decision.

The practical launch test is short:

1. Emit one synthetic event for each pricing rule version.
2. Confirm that the required correlation values survive ingestion.
3. Prove the supported search contract before relying on it operationally.
4. Run an alert drill, a silent-job drill, and an erasure review.
5. Record why ingestion-only logging is sufficient, or choose the broader service that supplies the missing control.

That produces a useful outcome even if the vendor changes: the pricing team has a stable evidence model. A support engineer can start from a request or account, identify rule version 4, and inspect the attributed charge data without parsing prose. The architecture stays small because the limits were chosen, not overlooked.

## Sources

- [OpenTelemetry logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Sentry event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)
