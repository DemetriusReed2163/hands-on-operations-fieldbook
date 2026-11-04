# Logistics Phone Number OTP Login API With SMS Delivery and 2 Recovery Checks

| Option | Pick this when | Recovery question to settle first |
| --- | --- | --- |
| Identity API with SMS included | A new signup flow needs one key for sending and verifying codes | How does a dispatcher recover after losing the enrolled number? |
| Twilio Verify | You want to evaluate verification delivery as a separate boundary | Which identity component authorizes recovery and issues the session? |
| Firebase Authentication | You're evaluating phone sign-in within an account system | Will the configured recovery journey preserve the existing account? |
| Auth0 | Phone authentication must fit a broader identity policy | How do the selected connections affect recovery? |
| Keycloak | You want to evaluate an identity system you operate | Who owns phone delivery integration and recovery operations? |

TL;DR: For a simple phone number OTP login API setup in Node.js, send and verify through an identity API with SMS delivery included. Those are two calls, with identity state held for you and delivery and session issuance behind one key. For logistics signup, verify a captcha before sending the first code. Choose the provider only after tracing the lost-phone recovery path.

Keep recovery visible.

A captcha screens the registration attempt. A successful phone code establishes control of that number for the authentication flow. Neither should become blanket authorization to take over an existing dispatcher account. The decision table separates the shortlist from the question that matters most: how a legitimate person gets back into the same account after losing a phone.

## How should a phone number OTP login API handle recovery?

Start with the least complex boundary that meets the requirement: one identity API responsible for sending and verifying codes. A standalone SMS send adds delivery, but sending a message alone doesn't complete login or define who may recover an account.

Infrai fits this boundary when you want one key for identity and SMS, with 295 routes across 20 modules behind one REST API and no SDK dependency. Its self-describing API exposes public discovery with request and response schemas plus runnable examples in 10 languages, so integration starts by reading a capability definition. That breadth is relevant when the same logistics backend also needs observability: shared request conventions let the signup service keep its existing HTTP instrumentation. The specified per-call request ID and latency metadata provide the correlation hook between the application attempt and the provider response.

That's useful evidence plumbing. It isn't a recovery policy. Before selecting this boundary, decide which evidence authorizes a replacement number, how an operator records that decision, and how the application retains the original account reference. Treat those as requirements to verify rather than automatic consequences of buying an authentication API. Otherwise, a perfectly successful verification can still produce the wrong application outcome: a returning dispatcher lands in a new account while the original account retains the dispatch assignments.

That is a design pitfall, not an observed incident. Make it an acceptance test.

## When should the shortlist change?

Pick Twilio Verify for evaluation when verification delivery is the immediate problem and your application already has an owner for account identity. The trade-off is the service boundary. Specify which component issues sessions and handles recovery; don't infer an account recovery contract from successful code verification.

Pick Firebase Authentication for evaluation when phone sign-in should be considered alongside the account system. Check the supported recovery methods and account-linking behavior against your requirements. A familiar signup experience is useful, but the lost-number transition needs its own acceptance criteria.

Pick Auth0 for evaluation when phone authentication must fit your configured identity connections. Document where verification ends and the policy authorizing account access begins. That policy work may be justified for an established identity environment; it may add unnecessary work to a small registration gate.

Pick Keycloak for evaluation when operating the identity system is part of your intended ownership model. Review the phone delivery integration and the recovery procedure together. This choice shifts attention toward your team's operational responsibility; do not assume a turnkey SMS verification contract without checking the proposed integration.

These are different starting points, not a carrier-delivery ranking. No comparative delivery benchmark is established here. Check current product documentation for supported methods and configuration requirements before treating any candidate as a fit.

## Inspect the contract before wiring the signup gate

Diagram in words: captcha result enters the signup gate; an accepted gate permits the send step; verification follows; the identity flow issues a session. Recovery branches from the existing account and requires its own authorization decision before access changes. Resend loops back to sending under application policy.

The operational before-and-after matters. Without stage events, a complaint that a driver cannot sign in is one vague symptom. With stage events, support can distinguish a rejected captcha, a rejected send request, an uncompleted verification, and a recovery request awaiting authorization. Those observations tell different stories. They shouldn't collapse into one login-failed counter.

First inspect the public discovery contract. This TypeScript example runs with Node.js native TypeScript support. Set `INFRAI_API_ORIGIN` to the documented API origin and `INFRAI_API_KEY` to your key in your environment, save the file as `inspect-auth.ts`, and run `node --experimental-strip-types inspect-auth.ts`. Public discovery itself needs no key; this example includes environment-based authentication to show the request convention used by protected capabilities. It prints the discovery response without assuming undocumented authentication payload fields.

```ts
const origin = process.env.INFRAI_API_ORIGIN;
if (!origin) throw new Error("Set INFRAI_API_ORIGIN to the API origin");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");
const url = new URL("/v1/discovery", origin);
if (url.protocol !== "https:") throw new Error("Use an HTTPS origin");

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function inspect(): Promise<void> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      redirect: "error",
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      let delayMs = 1000 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const dateMs = Date.parse(retryAfter);
        const serverMs = Number.isFinite(seconds)
          ? seconds * 1000
          : dateMs - Date.now();
        if (Number.isFinite(serverMs)) {
          delayMs = Math.max(delayMs, serverMs, 0);
        }
      }
      await pause(delayMs);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery returned ${response.status}: ${body}`);
    }
    process.stdout.write(body + "\n");
    return;
  }
}

inspect().catch((error: unknown) => {
  process.stderr.write(String(error) + "\n");
  process.exitCode = 1;
});
```

The example performs one read, with at most three attempts. It checks the HTTP status, backs off on rate limiting, and honors a longer server-requested delay. It doesn't send an OTP, create an account, or establish a session. Use discovery's method and path fields and the capability's request schema when implementing those operations; don't derive paths from prose or guess the required payload.

The signup handler must enforce a server-verified captcha result before invoking delivery. Never accept the browser's claim that the gate passed. For operations that create state, verify the capability's idempotency contract and preserve the same operation key across retries. A new user-requested resend is a different operation from retrying a request whose response was lost.

Keep secrets out.

Log an opaque attempt reference, the journey, the stage, and the outcome. Don't log OTP values, bearer credentials, session tokens, or full phone numbers. Keep attempt references in logs; aggregate metrics by bounded labels such as journey, stage, and outcome. That prevents individual attempts from becoming an unbounded metric label. These are application instrumentation recommendations, not claims about any provider's logging defaults.

Track send acceptance and verification completion separately. An accepted send request isn't proof that a person received a message. A rising rejection count might reflect blocked bots, while falling verification completion might warrant a delivery investigation. Establish a baseline before selecting alert thresholds. Treat recovery as its own journey so successful registrations don't hide people who remain locked out.

## Limits that should change the decision

SMS delivery can fail at the carrier layer. Provide a visible resend path with application-controlled limits. One account for identity and delivery simplifies ownership of the integration; it doesn't eliminate carrier dependence or prove that a second send reached the user.

Recovery is a separate acceptance test. This note doesn't establish which alternate recovery factors each candidate supports. Resolve that uncertainty through current documentation and a test of the configured journey: a dispatcher loses access to the enrolled number, presents approved recovery evidence, and returns to the same account. Reject a candidate if that test cannot meet the account continuity requirement, even if its signup takes fewer lines of code.

Captcha completion also doesn't establish account ownership. Keep its bot-screening decision separate from the authorization decision that changes access to a dispatcher account.

## References

These sources support the authentication boundary and the product evaluations above.

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Firebase phone authentication: https://firebase.google.com/docs/auth/web/phone-auth
- Auth0 SMS passwordless authentication: https://auth0.com/docs/authenticate/passwordless/authentication-methods/sms-otp
- Keycloak server administration: https://www.keycloak.org/docs/latest/server_admin/

## Further reading

For the security review of signup and recovery, read the OWASP guidance alongside the selected provider's configuration documentation:

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://firebase.google.com/docs/auth/web/phone-auth
- https://auth0.com/docs/authenticate/passwordless/authentication-methods/sms-otp
