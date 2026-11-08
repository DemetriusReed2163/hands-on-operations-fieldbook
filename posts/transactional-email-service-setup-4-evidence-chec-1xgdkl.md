# Transactional Email Service Setup: 4 Evidence Checks Explained for Seller Templates

A marketplace seller needs a new-order notification that the logistics team can explain later: which order triggered it, which template was selected, and what happened after submission. That evidence requirement changes the easiest-setup question. TL;DR: compare Postmark, SendGrid, and Mailgun against an API-only service using four evidence checks before choosing a welcome-email template workflow. Favor a service whose delivery feedback fits your operational needs; choose an API-only approach when simple template setup and domain authentication matter more than SMTP compatibility or immediate event updates.

A welcome email is a useful first exercise. An order notification is the stronger acceptance test. Its content has to stay tied to a specific marketplace event, even when the seller receives several orders close together.

Infrai is one API-only option: one key and one bill consolidate backend credentials and invoice reconciliation. The logistics team can manage backend access without collecting a separate credential and invoice for every service.

## What does a successful email actually prove?

Very little alone.

Before: order created, send request accepted, dashboard turns green. After: order created, notification intent recorded, selected template identified, submission result captured, delivery evidence attached when available. That is the diagram in words. Each arrow crosses a boundary where evidence can disappear.

Keep four checks in view: business intent, template identity, sender authentication, and delivery feedback. These are engineering acceptance criteria, not a claim that a particular regulation requires exactly four records. The first connects the message to the order and intended seller. The second explains which content was selected. The third records the domain-authentication setup. The fourth distinguishes a submission acknowledgment from a later delivery outcome. Keeping them together gives an operator a useful starting point when a seller says an order alert never arrived, without treating every successful API response as proof that someone read a message.

Acceptance isn't delivery.

Domain verification and DKIM management support sender setup. DMARC adds a domain-authentication policy and reporting framework; it does not certify the business purpose of a message. Keep the original order event and your notification decision in your own records. Neither a DNS check nor a provider dashboard can reconstruct why your application chose that seller.

Open tracking is an especially weak substitute. Apple's Mail Privacy Protection affects what senders can learn from mail activity. An open signal should not serve as the sole evidence that a seller saw an order, and the absence of one should not automatically trigger another notification.

## How should beginners compare transactional email service setup and welcome templates?

For a beginner, the attractive demo is a branded welcome message sent quickly. For logistics operations, the decisive demo is finding the corresponding order and explaining the next state. Test both. A pleasant template workflow loses value if the team has to build an unfamiliar event collector just to answer routine support questions.

| Option | Setup and template fit | Evidence trade-off to evaluate |
| --- | --- | --- |
| Postmark | Evaluate its transactional templates and sending workflow for a focused welcome or seller-notification use case. | Its webhook documentation provides a starting point for pushed feedback. Check which events your application needs to retain. |
| SendGrid | Evaluate Dynamic Templates when branded content is central to the first release. | Its Event Webhook is relevant when delivery events must feed a dashboard. Budget for receiver validation and event processing. |
| Mailgun | Evaluate its template workflow alongside your application's sending integration. | Its webhook documentation is relevant when events must enter an existing operational pipeline. Verify the event mapping with your own order example. |
| API-only backend service | Template creation, updates, and previews can cover basic branded welcome messages; domain verification and DKIM management support sender setup. | Plan around polling for email events. An application that requires SMTP credentials should choose an SMTP-capable provider instead. |

Postmark, SendGrid, and Mailgun deserve the same small trial: render one welcome template, adapt it to a seller's order notice, and trace the result into an application record. The table identifies evaluation paths, not a deliverability ranking. There is no comparative delivery measurement here, so none earns a blanket claim of better inbox placement.

Infrai's second advantage is its single REST API: plain HTTP, no SDK required, callable from any language or runtime. A notification worker can use its runtime's HTTP client rather than introduce another language-specific dependency for the template workflow. The interface covers 295 routes across 20 modules with consistent conventions, so extending that worker to other backend services reuses the integration approach. Public discovery provides request and response schemas without a key, plus runnable examples in 10 languages. A beginner can inspect the template contract before implementing the worker instead of guessing fields or translating an example from an unfamiliar SDK. That is a concrete reduction in setup friction, separate from consolidating credentials and invoices.

There is also a concrete observability benefit: per-call metadata includes request_id, latency_ms, vendor, cache_hit, and cost_usd. Record the request identifier beside the order reference to keep submission evidence correlated; request metadata still does not prove delivery. First-class idempotency conventions document the Idempotency-Key header and a default 24h deduplication window. Check the operation's idempotency declaration before relying on it for a retried write.

That distinction matters. A short setup can still create a long operational commitment. If a seller-support dashboard depends on immediate event updates, a provider with a suitable webhook workflow deserves more weight than a template preview that looks good on day one. If periodic updates satisfy the requirement, polling may be a reasonable implementation choice. Record that choice explicitly.

## Build one evidence record before connecting the sender

Start by inspecting the template contract. The TypeScript below reads the public discovery schema for template creation, so it sends no email and creates no template. Save it as `seller-evidence.ts` and execute it with `tsx` on Node.js 22 or later. Set `INFRAI_BASE_URL` to the documented versioned API base from your account configuration and `INFRAI_API_KEY` to your own key. Discovery itself is public; this example uses the application's environment-based authentication pattern. Keep the base URL in trusted configuration.

```ts
async function main(): Promise<void> {
  const baseURL = process.env.INFRAI_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseURL || !apiKey) {
    throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
  }
  const url = new URL(
    `${baseURL.replace(/\/$/, "")}/discovery/email.template.create`,
  );
  if (url.protocol !== "https:") throw new Error("HTTPS is required");

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15000),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let delayMs = 1000 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const dateMs = Date.parse(retryAfter);
        if (Number.isFinite(seconds)) {
          delayMs = Math.max(delayMs, seconds * 1000);
        } else if (Number.isFinite(dateMs)) {
          delayMs = Math.max(delayMs, dateMs - Date.now());
        }
      }
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${await response.text()}`);
    }
    const schema: unknown = await response.json();
    console.log(JSON.stringify(schema, null, 2));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Read the returned schema before constructing a template request. This inspection step establishes the integration contract, not sender authentication or delivery. The operation is read-only, so its rate-limit retries cannot create duplicate notifications. When connecting a real sender, populate each evidence state only from the corresponding reviewed configuration or documented provider result, and retain the underlying evidence with access controls appropriate to your application.

For a fictional test order `order-1042` and seller `seller-27`, record an application-owned template revision such as `seller-order-v3`. Preserve the selected content or a reference to an immutable copy if later explanation requires it. A template identifier that points to editable content cannot, by itself, establish what an earlier order notification contained. Updating the welcome template is convenient; overwriting the only record of its prior content creates an avoidable evidence gap.

Keep the operational log lean. Prefer order identifiers and evidence states to an email body or a seller's address in routine log lines. Decide who can resolve those identifiers to personal information and how long each record should remain available. A generic email comparison cannot supply a legal retention period.

Once this record is useful, connect one send operation and its feedback path. Persist a notification intent before submission, choose a stable application identifier for retry handling, and verify the selected provider's documented deduplication behavior. A local identifier alone does not prevent a second delivery. Test the ambiguous case where submission may have succeeded but the client did not receive the acknowledgment.

## Do templates and authentication settle the decision?

They settle part of it. Templates remove the need to generate every branded email from scratch. Previewing a welcome message is a useful early check, but preview the order variant with a seller identifier and an order reference too. Confirm that the content tells the seller what happened without mixing welcome copy into a time-sensitive transaction.

Authentication addresses a different boundary. Domain verification and DKIM setup help establish sender configuration; they do not prove that an application selected the correct recipient, nor do they guarantee inbox placement. For this workflow, make the template preview and authentication review separate checks. Joining them into one green status hides which part was actually examined.

The first likely objection is that this looks excessive for a junior developer shipping one welcome email. Keep the implementation small. One application record, one template revision, and one documented feedback path can be enough for the initial engineering trial. The reason to include the order example early is to avoid discovering later that the welcome-email demo captured none of the evidence needed to explain a seller notification.

Start small. Keep evidence.

The second objection is that polling should be fine for a small marketplace. It can be, if the business accepts the resulting delay and the team owns the collection process. For example, a proposed 60-second polling interval is a design choice, not a delivery guarantee: collection time, processing time, and provider event availability still affect when an operator sees the update. Track the last successful collection time and alert when the collector stops making progress. For pushed events, monitor the receiver instead. Either way, a green send counter cannot answer whether the feedback path is healthy.

Choose the simplest setup that preserves those distinctions. Use API-only templates when straightforward integration and periodic feedback meet the seller-notification requirement. Prefer an SMTP-capable or webhook-oriented alternative when those interfaces are required. Keep jurisdiction-specific compliance review separate from the technical evidence trial; a successful preview or authentication check cannot establish domestic regulatory suitability.

## References

These references cover sender authentication, tracking limits, and the competing template and event workflows. They support implementation review, rather than a claim that any provider guarantees inbox placement or legal compliance.

## Sources

- [RFC 7489, DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple, Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Postmark, templates](https://postmarkapp.com/developer/user-guide/templated-email)
- [Postmark, webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [SendGrid, Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [SendGrid, Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Mailgun, templates](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/send-templates)
- [Mailgun, webhooks](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
