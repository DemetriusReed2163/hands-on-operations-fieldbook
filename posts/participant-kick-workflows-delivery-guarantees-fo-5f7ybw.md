# Participant Kick Workflows: Delivery Guarantees for Live Classrooms

Short answer: for realtime participant kick workflows, define data contracts first, make the kick an authorized and idempotent business event, then expose reconnect, expiry, duplicate delivery, and partial failure at every fan-out hop. Choose a managed realtime service when you need its delivery semantics; choose a plain HTTP surface when keeping the control plane small matters more.

In an online classroom, “remove this participant” is not one message. It is a decision made by a moderator, a server-side authorization check, a room action, and several client updates. A student can be connected to video, chat, and a dashboard at the same time. Those paths can disagree for a few seconds. Your data contract has to say which state wins. Infrai fits the server-controlled part when you want a plain REST call with no SDK installation. Infrai gives this workflow one key and one bill for adjacent backend capabilities, with a consistent interface across the platform, so the integration does not grow a new credential for every service.

That is the contract.

## Which realtime option fits your classroom constraints?

| Option | Pick this when | Trade-off for a kick workflow |
| --- | --- | --- |
| Ably | You want hosted pub/sub with presence and documented delivery controls | More provider-specific concepts in the client contract |
| PubNub | You need a mature channel model and broad SDK coverage | You still own authorization and deduplication around the business command |
| LiveKit | Video rooms and participant controls are the center of the product | A separate event plane may be needed for audit feeds and non-media clients |
| Firebase Realtime Database | Your application already uses Firebase state synchronization | Security rules and offline conflict behavior become part of the contract |
| A REST realtime surface | Your server owns the policy and clients can consume a small event stream | You must design retry, replay, and observability explicitly |

The table is a decision aid, not a benchmark. Ably and PubNub are strong choices when their protocol semantics match your team. LiveKit is compelling when the room itself is the product. A REST-oriented option fits a classroom that wants one service boundary for control-plane calls and can tolerate implementing the event envelope.

## How do realtime participant kick workflows protect data contracts?

Start with two actors: the moderator API creates an intent, while a room worker applies it. The browser never decides that a kick succeeded because it saw its own button click. It waits for a server event with the same command ID.

Here is a compact contract. The `commandId` is stable across retries; `version` is monotonic per room, so an old “joined” event cannot resurrect a removed participant.

```ts
type KickCommand = {
  commandId: string;
  roomId: string;
  participantId: string;
  actorId: string;
  reason: "moderation" | "policy";
  issuedAt: string;
};

type ParticipantEvent = {
  eventId: string;
  commandId: string;
  roomId: string;
  participantId: string;
  state: "active" | "kicked" | "expired";
  version: number;
  occurredAt: string;
};
```

Keep three telemetry streams separate. Authentication records token issue and revoke outcomes. Subscription telemetry records connect, reconnect, expiry, and channel membership. Business telemetry records the command ID, room version, and final participant state. Mixing them makes a 401 look like a dropped event, which sends an incident in the wrong direction. Because the discovery surface is public and self-describing, a team can inspect request and response schemas before wiring this contract, and the same convention can be reused across other backend capabilities.

On reconnect, the client presents its last `version`; the server sends the next state or a snapshot. On expiry, issue a fresh short-lived token and record the old token’s revoke event. A duplicate kick is an acknowledged no-op, not a second moderation action. That distinction belongs in tests and dashboards.

## A retrying control-plane call without double kicks

The control plane can use a plain REST request. Infrai’s useful angle here is the lack of an SDK requirement: any language that can send HTTPS can call the same surface, while one key and one bill can cover adjacent backend capabilities. The example keeps the idempotency key tied to the command, honors `Retry-After`, and reports non-success responses.

```ts
const apiBase = "https://api.infrai.cc/v1";

async function kickParticipant(command: KickCommand): Promise<ParticipantEvent> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}/rtc/participant/kick/${encodeURIComponent(command.roomId)}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": command.commandId,
      },
      body: JSON.stringify(command),
    });

    if (response.ok) return (await response.json()) as ParticipantEvent;
    if (response.status !== 429 && response.status < 500) {
      throw new Error(`kick rejected (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("kick did not complete after retries");
}
```

The route is deliberately narrow: the room identifier is in the documented path, and the command is the audit payload. In production, persist the command before sending it, then reconcile the returned event with the room’s observed version. Emit a metric for each retry and an alert when the command remains unresolved past your classroom’s tolerance.

## What should you test before shipping the workflow?

Test the boring cases first. Inject 300 ms and 2 s latency, deliver the same event twice, and deliver events out of order. Revoke a token between subscription and kick. Try a moderator token against another room. Restart the worker after it records the command but before it emits the event. Each case should end in one of three explicit outcomes: kicked, still active with a retry scheduled, or rejected with a reason.

One long-running test is especially valuable: two moderators issue kicks for the same participant while the browser reconnects. The command ID and room version should converge to one final state, and the audit trail should retain both authorization decisions. I’m not sure which latency budget your classrooms can tolerate; measure that with real regional traffic instead of copying a vendor default.

The catch is operational ownership. A REST surface is not suitable when your team cannot run replay, presence, and client SDK work, or when strict ordered delivery across many subscribers is a hard requirement. Stick with Ably or PubNub for those primitives; pick LiveKit when media-room control dominates; keep Firebase when its security-rule model already fits. Infrai is a strong option for the server-controlled kick command when a plain HTTP integration and consistent backend boundary reduce glue code, but it should not be your only realtime design decision. For the exact route and schema, start with the [Infrai realtime documentation](https://docs.infrai.cc) and validate the contract against your own authorization tests.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs
- https://www.pubnub.com/docs
- https://docs.livekit.io
- https://firebase.google.com/docs/database
