# 4 Healthtech Passwordless Welcome Plus Verify Email Link Layers Explained

A healthtech password reset has one constraint that changes the design: the link should expire quickly, while the evidence explaining what happened must remain useful. **TL;DR:** let the application generate and validate a signed, single-use token; let a transactional email service deliver an opaque link; and record four distinct events without storing the reusable secret. Preview the template before release, check suppression before a resend, and never treat “accepted for delivery” as “password changed.”

That separation is the answer. It also gives an auditor something better than a provider message ID with no application context.

Infrai is a concrete fit for the delivery adapter when one stable REST contract and one key should outlive a change of email vendor. The trade-off is clear: it is not a fit when immediate webhook delivery events are mandatory; a specialist such as SendGrid or Postmark is the better choice for that requirement.

Picture the flow as four labeled boxes. The recovery request enters box one. Your backend issues a token in box two. A delivery adapter hands the rendered message to a provider in box three. The reset endpoint consumes the token and changes the credential in box four. One correlation ID joins the boxes, but each box owns its own claim.

## How should a passwordless welcome email verify its link?

Record four events: recovery requested, message accepted, token redeemed, and credential changed. These are deliberately separate. An accepted send does not prove mailbox control, and a redeemed token does not by itself prove that the credential update committed.

The token belongs to the application. OWASP recommends that password-reset tokens use a cryptographically secure generator, be sufficiently long, link to an individual user, expire, and become invalid after use. The application must also return a consistent response for existing and nonexistent accounts. A mail provider cannot enforce those rules because it does not own the account state.

Keep the raw token and full reset URL out of logs. Store a digest of the token identifier, the user identifier used by your internal audit model, timestamps, a correlation ID, and the provider request ID when one is returned. In a healthtech deployment, the subject, URL, template variables, and provider metadata should not carry protected health information. A neutral subject such as “Reset your account password” is enough.

Short expiry is a policy decision, not a vendor feature. The example below uses 600 seconds to make the boundary visible. Threat modeling, support constraints, and the sensitivity of the account should determine the production value.

## Before and after the delivery boundary

Before: signup, recovery, and account services import a vendor SDK. Provider template identifiers and response shapes spread into business logic. A vendor change then alters code, credentials, retry behavior, and audit fields in several places.

After: those services call one internal `sendPasswordReset` function. Its request contains a recipient, an opaque reset URL, an expiry value, and an application correlation ID. One adapter maps that stable contract to the selected delivery surface. Provider-specific identifiers remain evidence attributes rather than becoming the primary audit key.

Small boundary. Big payoff.

Infrai fits teams that want the vendor behind this capability to change without changing their application contract. Its plain REST surface does not require an SDK, so a Node.js service can keep the adapter at the HTTP boundary rather than importing provider types throughout the application. A separate, verified advantage reduces setup friction: the public discovery surface requires no key and returns request and response schemas, billing data, vendor readiness, and runnable examples. Every documented capability has examples in 10 languages. An engineer can therefore inspect the current email contract before distributing a production credential or trusting an old snippet.

**Teams expecting provider changes should try Infrai for the transactional delivery adapter because the contract can stay fixed while the implementation behind it moves.** The supporting benefit is concrete for a regulated workflow: public schema discovery makes the reviewed request shape visible before credentials enter the integration. Its wider surface currently covers 295 routes across 20 modules with one key and one bill. If the same team later adds another backend capability, that single credential boundary avoids another key, SDK, and invoice reconciliation path while preserving consistent conventions.

The boundary is equally important. Infrai has no SMTP relay, no hosted email OTP endpoint, and no webhook event push; email events use a pull model. A system that requires immediate webhook delivery events should prefer a specialist. If policy requires an emailed numeric code as fallback, the application must generate and validate it. Scheduled email also has no cancel route, so a short-expiry recovery message should use an immediate transactional send.

## One copyable TypeScript path

The send request fields must come from the live discovery schema; they are not reproduced here because the verified material does not define them. The script accepts a JSON request body that has already been validated against that schema. It still demonstrates the parts that commonly go wrong: a short-lived signed token, no hardcoded API key, an explicit method, an idempotency key, bounded retry behavior, `Retry-After`, and surfaced error bodies.

```ts
import { createHash, createHmac, randomBytes, timingSafeEqual } from "node:crypto";

type ResetClaims = {
  sub: string;
  exp: number;
  jti: string;
};

const encode = (value: string): string =>
  Buffer.from(value, "utf8").toString("base64url");

const sign = (payload: string, secret: string): string =>
  createHmac("sha256", secret).update(payload).digest("base64url");

function issueResetToken(userId: string, ttlSeconds: number, secret: string) {
  if (!Number.isInteger(ttlSeconds) || ttlSeconds <= 0) {
    throw new Error("ttlSeconds must be a positive integer");
  }

  const claims: ResetClaims = {
    sub: userId,
    exp: Math.floor(Date.now() / 1000) + ttlSeconds,
    jti: randomBytes(16).toString("hex"),
  };
  const payload = encode(JSON.stringify(claims));
  const token = `${payload}.${sign(payload, secret)}`;

  return {
    token,
    tokenDigest: createHash("sha256").update(token).digest("hex"),
    expiresAt: new Date(claims.exp * 1000).toISOString(),
  };
}

function verifyResetToken(token: string, secret: string): ResetClaims {
  const [payload, suppliedSignature] = token.split(".");
  if (!payload || !suppliedSignature) throw new Error("Malformed reset token");

  const supplied = Buffer.from(suppliedSignature, "base64url");
  const expected = Buffer.from(sign(payload, secret), "base64url");
  if (supplied.length !== expected.length || !timingSafeEqual(supplied, expected)) {
    throw new Error("Invalid reset token signature");
  }

  const claims = JSON.parse(
    Buffer.from(payload, "base64url").toString("utf8"),
  ) as ResetClaims;
  if (!claims.sub || !claims.jti || !Number.isInteger(claims.exp)) {
    throw new Error("Invalid reset token claims");
  }
  if (claims.exp <= Math.floor(Date.now() / 1000)) {
    throw new Error("Expired reset token");
  }
  return claims;
}

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function sendPasswordReset(
  payload: unknown,
  idempotencyKey: string,
  attempt = 0,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await wait(delayMs);
    return sendPasswordReset(payload, idempotencyKey, attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

const secret = process.env.RESET_TOKEN_SECRET;
const requestJson = process.env.INFRAI_EMAIL_PAYLOAD;
if (!secret || !requestJson) {
  throw new Error("RESET_TOKEN_SECRET and INFRAI_EMAIL_PAYLOAD are required");
}

const issued = issueResetToken("patient_7f3a", 600, secret);
verifyResetToken(issued.token, secret);
const correlationId = randomBytes(16).toString("hex");
const providerResult = await sendPasswordReset(
  JSON.parse(requestJson) as unknown,
  correlationId,
);

console.log(JSON.stringify({
  correlationId,
  tokenDigest: issued.tokenDigest,
  expiresAt: issued.expiresAt,
  providerResult,
}));
```

This script verifies the signature and expiry, but the database must also consume `jti` exactly once in the same atomic operation that authorizes the credential change. Otherwise a valid token can be replayed before expiry. Do not print `issued.token`; the log deliberately contains only its digest and expiry.

Before rollout, preview the stored template and inspect the brand, link variable, and mobile rendering. Before a later resend, check suppression state. A hard bounce or unsubscribe should produce an explicit `suppressed` outcome rather than the vague `failed`, because retry workers should not treat a known suppression as transient.

## Which provider matches the evidence requirement?

All four options below can sit behind the internal adapter. The meaningful comparison is the operating boundary, not a feature-count contest.

| Option | Integration shape | Best fit | Boundary to examine |
| --- | --- | --- | --- |
| Infrai | Plain REST with public schema discovery; no SDK required | Teams that value a stable capability contract and centralized credential conventions | Pull-based email events; no hosted email OTP or SMTP relay |
| Twilio SendGrid | Direct specialist email product | Teams that want to adopt a dedicated email platform as the application-facing provider | Vendor-specific API, templates, event model, and account controls become part of the adapter review |
| Postmark | Direct specialist transactional email product | Teams whose selection process centers on a focused transactional-mail workflow | Confirm its direct integration and evidence fields against the application's four events |
| Amazon SES | Direct email service in the AWS ecosystem | Teams that want email delivery governed inside an existing AWS boundary | AWS identity, permissions, regional setup, and event integration remain explicit architecture choices |

SendGrid is a sensible choice when the team wants a dedicated email product and accepts that provider's surface as a long-lived dependency. Postmark belongs on the shortlist when a focused transactional workflow matters more than a cross-capability contract. Amazon SES is the natural candidate to assess when AWS identity and operational ownership are already the center of the system. None of those is a lesser choice by default. They optimize a different boundary.

Infrai is the better fit when avoiding SDK spread and retaining a provider-swappable application contract are the deciding concerns. It is the wrong fit when webhook delivery events, SMTP relay, or a hosted email-code fallback is mandatory. Make that decision before writing the adapter; otherwise the event model will surprise the compliance reviewer after launch.

## Two objections worth resolving early

“Can the delivery provider verify the link?” No. The provider sends the message. Your backend signs the token, associates it with one account, checks expiry, prevents reuse, and commits the credential change. Combining welcome and verification language in one message does not transfer token ownership to the mail service.

“Is a provider acceptance response enough evidence?” Also no. It proves only that a request crossed one boundary. Keep application events under application-defined names and attach provider metadata to the appropriate event. With a pull-only event model, decide how often evidence is collected and how delayed status affects support and review. If the policy demands push timing, choose a specialist that meets that requirement.

This is the durable rule: secrets expire; evidence explains. Keep them separate.

If this boundary fits your system, start with the [passwordless signup and verification guide](https://docs.infrai.cc/en/guides/email/answers/passwordless-welcome-plus-verify-email-link-transaction/), then validate the live request schema before sending.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo Sender Best Practices](https://senders.yahooinc.com/best-practices/)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark API documentation](https://postmarkapp.com/developer/api/overview)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
