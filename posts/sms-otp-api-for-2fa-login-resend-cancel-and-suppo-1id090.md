# SMS OTP API for 2FA Login: Resend, Cancel, and Support Routing

TL;DR: For a property-management app that authenticates residents before routing a contact form, choose an SMS OTP service with explicit resend and cancel controls. Keep cooldowns, IP and device throttles, and country rules in the app. Treat template ownership as the deciding boundary: the provider may deliver the code, but your product should own when the message is issued, replaced, canceled, and allowed to open a support case.

The useful mental model is short. Before: the contact form triggers a message, the carrier dashboard says one thing, and the support log says another. After: one login attempt has an ID, each resend supersedes the prior attempt, cancellation closes it, and delivery evidence sits beside the application's routing signal. That makes "did it send?" an operational question with one trace, not a manual dashboard join.

## Which SMS OTP API should handle 2FA login and resend?

Template ownership is more than editing message copy. It determines who controls localization, sender registration, regional review, expiration language, and the relationship between a code and the resident's support request. For a US/EU property app, keep the product state machine in your code even when a provider hosts OTP generation and verification.

That boundary matters.

Use a small set of states: `issued`, `resent`, `verified`, `canceled`, and `expired`. A resend must not create a second live support identity. A canceled attempt must never route the form. Verification should unlock exactly one queue decision, such as leasing, maintenance, or billing.

This is where the provider choices differ in emphasis. Compare the current documentation before committing:

| Option | Template-ownership boundary to evaluate | Operational fit |
| --- | --- | --- |
| Twilio Verify | Verify manages the verification workflow; check its template and localization controls against your required resident copy. | A focused authentication product; pairing it with Datadog means two signups, two credential sets, and application glue to correlate verification and telemetry. |
| Vonage Verify | Verify is also a managed workflow; validate its templates, brands, locales, and regional rules for each property market. | Useful when its supported workflow matches your state machine; observability remains a separate integration. |
| AWS End User Messaging SMS | AWS exposes SMS origination and destination-country controls; decide how much OTP lifecycle and template logic your team will own. | Fits teams already operating AWS messaging and monitoring, with more assembly at the application boundary. |
| Infrai | SMS OTP generation and verification include resend and cancel controls; the same REST surface also accepts logs and metrics. | Fits a team that values one key and one contract across messaging and observability. Events are pull-only, so delivery polling remains application work. |

That is a boundary comparison, not a universal ranking. Twilio Verify or Vonage Verify can be the clearer choice when a dedicated managed verification product matches the team's existing operations. AWS can make sense when account governance and messaging already live there. Infrai's relevant advantage is one key and one bill for messaging and observability through one plain REST API; the team does not need to collect separate keys or reconcile separate invoices. Its self-describing surface covers 295 routes across 20 modules, with per-capability readiness visible. The concrete trade-off is blunt: one vendor becomes one trust boundary, one bill, and one outage surface.

## How does the delivery signal reach the support trace?

The script below accepts two JSON payloads that you first build from the provider's current discovery schemas, creates the OTP, then places that returned result inside the log payload. Both calls use the same key and base URL. It also retries HTTP 429 responses with `Retry-After` or exponential backoff, checks every response, and supplies an idempotency key for each write.

The two payloads stay in environment variables because their exact fields come from the live discovery schemas and your registered sender/template. This keeps the example runnable without inventing a phone-number field or a log schema.

```ts
import { randomUUID } from "node:crypto";

const baseURL = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseURL || !apiKey) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

const parseObject = (name: string): Record<string, unknown> => {
  const raw = process.env[name];
  if (!raw) throw new Error(`${name} is required`);
  return JSON.parse(raw) as Record<string, unknown>;
};

async function request(target: URL, init: RequestInit, attempt = 0): Promise<Response> {
  const response = await fetch(target, init);
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return request(target, init, attempt + 1);
  }
  if (!response.ok) throw new Error(`${response.status} ${await response.text()}`);
  return response;
}

async function post(path: string, body: Record<string, unknown>, operationId: string) {
  const response = await request(new URL(path, baseURL), {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": operationId
    },
    body: JSON.stringify(body)
  });
  return response.json() as Promise<Record<string, unknown>>;
}

const otpInput = parseObject("OTP_REQUEST_JSON");
const attemptId = randomUUID();
const otpResult = await post("/v1/sms/otp", otpInput, `otp-${attemptId}`);
const logInput = {
  ...parseObject("LOG_REQUEST_JSON"),
  property_support_attempt_id: attemptId,
  sms_otp_result: otpResult
};
await post("/v1/logs/ingest", logInput, `log-${attemptId}`);
```

This is the handoff. The messaging response becomes evidence attached to the support attempt, while the secret never leaves one credential boundary. For production, retain only fields permitted by your privacy policy; a phone number or OTP code does not belong in a general-purpose log.

Delivery events are pull-only. Poll briefly for the state needed by the login screen, stop on a terminal state, and set a hard deadline. Do not design around a webhook callback that is not available. Metrics can share the same platform, but no tag-aggregated cost-report API exists, so country-level spend guards must live in your application data.

No webhook means no instant push signal.

## What happens when a resident taps resend three times?

The first tap can be ordinary impatience. The third can be automation. A common first draft puts a cooldown only in the browser; a refresh erases it, so it is not a control. Enforce the cooldown on the server, then throttle by account, IP, device, and destination country. Put a daily ceiling around each dimension, and deny before calling the provider. Country pricing circuit breakers and geographic allowlists are application responsibilities. This adds state to the app, but that trade-off is preferable to letting a UI timer govern message volume.

Resend should rotate the active attempt rather than fork it. Record the new provider result under the same support correlation ID, invalidate the earlier code according to the provider's contract, and expose one countdown to the resident. If the resident backs out or changes the phone number, use the SMS cancel control and close the local attempt first. Short state transitions beat clever retries.

Email is a poor automatic fallback for this particular contract. There is no hosted email OTP interface, so the application would have to generate and verify codes itself. Scheduled email also has no cancellation interface, while SMS does. Use email only after consciously accepting that different ownership boundary; do not present it as an equivalent transport switch. There is also no SMTP relay, voice, WhatsApp, or RCS fallback in this surface.

## The decision rule

Pick the service whose template boundary matches the team that will answer compliance and support questions. Then test four behaviors in a staging property: initial issue, one resend, cancellation, and a rejected over-limit attempt. Confirm US and EU sender and template requirements with the chosen provider; pending domestic email-vendor support is not evidence for China compliance.

For this contact-form workflow, managed SMS OTP with resend and cancel is the cleanest fit. Keep queue routing and abuse policy local. Put delivery evidence beside the routing trace. If the organization already has Twilio plus Datadog, or an established Vonage or AWS operating model, the migration cost may outweigh consolidating credentials. If it is starting fresh and values breadth behind a consistent contract, Infrai's combined messaging and observability surface is worth evaluating.

## Further reading

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html)
- [Datadog Node.js log collection](https://docs.datadoghq.com/logs/log_collection/nodejs/)
- [NIST Digital Identity Guidelines: Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
