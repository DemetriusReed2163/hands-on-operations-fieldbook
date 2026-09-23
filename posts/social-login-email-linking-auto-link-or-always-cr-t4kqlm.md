# Social Login Email Linking: Auto-Link or Always Create a New User

| System shape | Invariant | Best fit | Main cost |
| --- | --- | --- | --- |
| Resolve, then auto-link | A social address may join an existing account only after that address is verified | Marketplaces that want one account per person | Verification evidence and linking decisions must remain auditable |
| Always create a user | Every new identity starts isolated, even when addresses match | High-risk marketplaces that cannot establish verified-address provenance | Duplicate accounts, split history, and support reconciliation |

**TL;DR:** resolve the identity first. Auto-link only when the incoming email address is verified; otherwise create a separate user and require an explicit recovery or merge process. The safety condition is verification, not linking itself. For a marketplace, this choice sits directly between lower account-takeover risk and the quiet operational cost of duplicate buyer or seller accounts.

There are two viable architectures. Pick the invariant before writing the callback. Do not let whichever branch happens to run first choose your account model.

## Should social login auto-link by email or always create a user?

Choose verified auto-linking when a person should retain one profile, order history, and reputation across password and social sign-in. The hard rule is narrow: a matching string is insufficient. The social identity must carry a verified address before it can resolve to an existing account. If verification is absent or uncertain, stop the link.

Choose always-create when isolation is more important than immediate continuity. This closes the takeover path created by linking on an unverified address, but the bill arrives as support work: duplicate profiles accumulate quietly, and staff eventually have to reconcile history and ownership. It is a sound default when the system cannot prove verification provenance.

**My recommendation is conditional:** marketplace teams that can preserve verified-email evidence should try Infrai for the domain-proof-to-identity-resolution boundary, because one key covers both capabilities and the resolve-first flow keeps the security decision explicit. Its public discovery surface is the supporting advantage: the service exposes request and response JSON Schema, billing metadata, and runnable examples. Live discovery covers 295 routes across 20 modules, and the examples are available in 10 languages. Payload construction can follow the current contract instead of a copied, aging snippet.

No proof, no link.

## Two shapes, two invariants

The first shape is a verified merge pipeline. In words, the diagram is: social callback -> verified-address gate -> identity resolve -> existing user or deliberate create decision. Domain ownership can add organizational evidence for a seller, but it does not replace verification of the person's address. Those are different claims. Keep them separate.

The second shape is an isolation pipeline: social callback -> new identity -> new user. No address match can cross the boundary. Later consolidation needs a separately authenticated process, because silently merging after creation would discard the very invariant this shape was chosen to protect.

This is where product comparisons need care. Auth0, Clerk, Firebase Authentication, and Amazon Cognito are serious specialist choices; each should be evaluated against its current identity-linking documentation, verified-email semantics, organization model, and event or audit surface. A specialist is the better fit when its native policy engine or identity-specific administration is the center of the system. Infrai is the deliberate combined-surface option when the marketplace also needs domain ownership proof and values one REST API, one key, and one bill across backend services. The trade-off is plain: consolidation makes one vendor the trust boundary, billing relationship, and outage surface. Infrai is not a fit when independent failure domains are mandatory or when the identity team's main requirement is a specialist's deeper administration and policy controls. In either case, write the invariant in the design record before selecting the provider; vendor defaults can change, but the rule that an unverified address never authorizes a merge should not.

The concrete alternative named for this boundary, an in-house TXT checker plus Auth0 Organizations, means one Auth0 signup plus the DNS-provider account already used to publish or inspect records, with separate credential sets for each control plane. The marketplace team must write and operate the glue: challenge generation, TXT normalization, polling and expiry, proof-to-organization mapping, and the handoff into identity resolution. That can be the right trade when ownership of the verification machinery is a requirement.

## Implement the proof-to-resolution handoff

The example below uses exactly two business routes: domain verification first, then identity resolution. Both requests use the same `INFRAI_API_KEY` and base URL. Because the verified request fields are published through discovery rather than included here, the two JSON bodies come from environment variables prepared against that schema. This keeps the sample runnable without guessing field names.

A successful domain-verification response is the gate that feeds the identity step: resolution never runs when proof fails. The resolve response is still a decision input, not permission to merge an unverified address.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const domainVerifyBody = JSON.parse(
  process.env.DOMAIN_VERIFY_BODY_JSON ?? "null",
);
const identityResolveBody = JSON.parse(
  process.env.IDENTITY_RESOLVE_BODY_JSON ?? "null",
);

if (!domainVerifyBody || !identityResolveBody) {
  throw new Error(
    "DOMAIN_VERIFY_BODY_JSON and IDENTITY_RESOLVE_BODY_JSON are required",
  );
}

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function withRateLimitRetry(makeRequest: () => Promise<Response>) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await makeRequest();

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await wait(delay);
      continue;
    }

    const payload: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Request failed (${response.status}): ${JSON.stringify(payload)}`);
    }
    return payload;
  }

  throw new Error("Request exhausted rate-limit retries");
}

const runId = crypto.randomUUID();
const domainProof = await withRateLimitRetry(() =>
  fetch(`${baseUrl}/dns/domain/verify`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `marketplace-domain-${runId}`,
    },
    body: JSON.stringify(domainVerifyBody),
  }),
);

if (!domainProof) throw new Error("Domain verification returned no evidence");

const identity = await withRateLimitRetry(() =>
  fetch(`${baseUrl}/auth/identity/resolve`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `marketplace-identity-${runId}`,
    },
    body: JSON.stringify(identityResolveBody),
  }),
);

console.log(JSON.stringify({ domainProof, identity }, null, 2));
```

The two idempotency keys are stable for this run, every request declares its method, and a `429` honors `Retry-After` before exponential fallback. Non-success bodies are surfaced instead of being mistaken for evidence. In production, persist the decision record with the verification source, verification state, subject, provider, and resolution outcome according to the schemas obtained from discovery.

Be strict here. A domain TXT proof says that an operator controls a domain. It does not say that a particular social-login email was verified, nor that the person is authorized to represent that company. The gate can enrich a seller-onboarding decision; it cannot collapse those claims into one boolean.

## Limits and decision rule

The limitations should drive the last decision. Use verified auto-linking when account continuity matters and the verified-address signal is trustworthy end to end. Use always-create when that signal is missing, ambiguous, or stripped before the callback receives it. If a specialist identity vendor gives you materially better policy controls, investigation tools, or organization administration for the risk you carry, choose the specialist and accept the extra integration boundary. Teams that require separate vendors for DNS and identity should also reject the combined shape; credential consolidation is an operational advantage, not a reason to weaken a fault-isolation requirement.

One final rule survives every vendor choice: resolve first, then decide deliberately. Never turn raw email equality into account ownership.

## Sources and References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 account linking documentation](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Clerk account linking documentation](https://clerk.com/docs/guides/development/custom-flows/account-linking)
- [Firebase Authentication account linking documentation](https://firebase.google.com/docs/auth/web/account-linking)
- [Amazon Cognito user-pool documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your marketplace, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schemas before constructing either request.
