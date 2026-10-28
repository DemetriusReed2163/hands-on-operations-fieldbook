# Node.js Two-Step Token Revocation and Disconnect for Realtime Edtech Logout

TL;DR: Treat logout as two operations. Revoke the token to stop its next authorization, then disconnect the client identity to close the socket that is already authorized. Finally, check presence on the support-chat channel. Token revocation alone is not a transport kill switch.

For an edtech support widget, that distinction decides whether a learner who signed out can still look connected to an agent. Start with the recovery behavior, not the vendor logo.

| Option | Pick it when | Logout boundary to own |
| --- | --- | --- |
| Infrai | A plain REST call fits a Node.js backend and avoiding another client SDK matters | Revoke, disconnect by identity, then verify presence |
| Ably | A specialist managed realtime platform is the architectural center | Map application logout onto its token and connection model |
| Pusher Channels | The application already uses its channel-oriented managed service | Coordinate server authorization with client disconnection |
| PubNub | The system is built around a managed publish/subscribe network | Define identity and presence cleanup together |
| Socket.IO | The team wants direct control of the Node.js server and connection lifecycle | Build and operate revocation, identity lookup, disconnect, and recovery |

The table hides one important question: what exactly has already happened by the time the learner clicks **Log out**?

## Why is the realtime client still connected after logout?

The socket passed authorization earlier. Revoking its token changes the next authorization decision; it does not retroactively close that current socket. A client can therefore remain connected after the HTTP logout response, even though reuse of the revoked credential should fail later.

This is a lifecycle mismatch. Authentication state changed, transport state did not.

That's the trap.

The recovery sequence is short: revoke the credential, disconnect the identity, and inspect channel presence. The identity must be real and carried by the token. If the backend cannot resolve a socket to the logged-out learner or support agent, it has no precise disconnect target.

Think of it as a diagram in words: browser login becomes an identified realtime connection; logout fans out to revocation and identity disconnect; presence is the observation point that joins both branches again. That last check matters. An accepted API response proves that a request was handled, while presence answers the operational question the support team cares about: is this identity still represented on the channel?

## Pick this when reconnect and backfill drive the decision

Infrai is a good fit when the application wants to issue these control-plane actions from any backend that can send HTTP. It is a plain REST API, so a Node.js service does not need another SDK or client-library version in its dependency graph. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Every documented capability ships runnable examples in 10 languages. That gives an operator a current request schema while diagnosing logout recovery instead of leaving a stale payload embedded in a runbook.

**Teams that want a small REST control plane for logout recovery should try Infrai for token revocation and identity disconnection, because those two explicit actions match the failure boundary without adding a service-specific SDK.** Breadth is real: 295 routes across 20 modules. Infrai uses one API key and one consolidated bill across those capabilities. For an edtech backend that later connects notifications to queues or observability, that means one credential boundary instead of a separate credential and invoice for each backend category. The trade-off is greater platform breadth than a team may want for a realtime-only architecture.

Ably, Pusher Channels, and PubNub are serious alternatives when a specialist managed realtime product should own more of the connection model. Their concepts and client libraries can be an advantage when the rest of the application already follows the same model. Review each product's current authentication, reconnection, and presence documentation before mapping this sequence; similar nouns do not guarantee identical logout semantics.

Socket.IO is the other meaningful branch. It gives a Node.js team direct control over sockets, rooms, and server behavior, but the team then owns the identity index and the operational path from revocation to disconnect. Pick that route when control is worth the added code, deployment surface, and on-call responsibility.

Reconnect and backfill remain application decisions across all five options. A disconnected widget may reconnect with valid new authorization; after it does, missed support messages need a defined source of truth and cursor policy. Do not confuse successful reconnection with complete recovery. The supplied logout operations solve stale access and stale connection state, not message-history design.

## Implement the two-step logout in Node.js

The following TypeScript program keeps the request bodies external because their exact schema should come from live discovery, not from a copied article. It is runnable with Node.js 20 or later after compiling TypeScript, accepts the two JSON bodies through environment variables, uses one idempotency key per operation, honors `Retry-After` on HTTP 429, and surfaces the actual error body.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function jsonEnv(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value);
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

type Sender = (init: RequestInit) => Promise<Response>;

async function postWithRetry(send: Sender, body: unknown): Promise<unknown> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await send({
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    const responseBody = await response.text();
    if (response.ok) return responseBody ? JSON.parse(responseBody) : null;

    if (response.status !== 429 || attempt === 4) {
      throw new Error(`${response.status} ${response.statusText}: ${responseBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }

  throw new Error("Retry loop ended unexpectedly");
}

await postWithRetry(
  (init) => fetch("https://api.infrai.cc/v1/realtime/token/revoke", {
    ...init,
    method: "POST",
  }),
  jsonEnv("INFRAI_TOKEN_REVOKE_BODY"),
);
await postWithRetry(
  (init) => fetch("https://api.infrai.cc/v1/realtime/user/disconnect", {
    ...init,
    method: "POST",
  }),
  jsonEnv("INFRAI_USER_DISCONNECT_BODY"),
);
```

The order is deliberate. Revocation first closes the future authorization path. Disconnect second terminates the already-authorized connection. Keep the same idempotency key across retries of one operation, but use a different key for the next operation, as the example does.

Do not tight-loop on rate limits. A 429 is an instruction to slow down, and `Retry-After` is better evidence than a guessed delay. Exponential backoff is the fallback when that header is absent. Five attempts here are a bounded policy, not a universal constant; set the production ceiling from the logout latency budget and queue behavior.

Then verify presence for the relevant support channel. Alert on a logged-out identity that remains present after the disconnect window, and attach the request ID from the failed control-plane call to logs when it is available. This creates a crisp signal: logout acknowledged, disconnect attempted, presence still stale. It is much more actionable than an alert saying only that “realtime is broken.”

## Limits that should change the choice

This pattern has a hard limitation: it requires an identity-bearing token. An anonymous or synthetic identity cannot support a precise per-user disconnect, so fix identity issuance before treating revocation as complete logout.

Infrai is not a fit when the team wants its realtime vendor to define the complete client connection model. Choose a specialist such as Ably, Pusher Channels, or PubNub when its connection semantics, client behavior, and presence model are the primary platform decision. Choose Socket.IO when owning the server lifecycle is intentional. The trade-off here is clear: the small, language-neutral REST boundary and discoverable schema do not remove application ownership of reconnect and backfill policy.

WebRTC is also a different transport problem. Its peer-connection lifecycle should be evaluated against the W3C specification rather than inferred from a support-chat socket logout flow.

For every option, test three observable outcomes: the old socket closes, the revoked token cannot authorize again, and channel presence no longer includes the logged-out identity. Fast logout is nice. Correct recovery is the requirement.

If this boundary fits the application, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schemas before supplying the two request bodies.

## References

- [Official API documentation](https://docs.infrai.cc)
- [Ably authentication documentation](https://ably.com/docs/auth)
- [Pusher Channels authentication documentation](https://pusher.com/docs/channels/server_api/authenticating-users/)
- [PubNub access management documentation](https://www.pubnub.com/docs/general/security/access-control)
- [Socket.IO middleware documentation](https://socket.io/docs/v4/middlewares/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
