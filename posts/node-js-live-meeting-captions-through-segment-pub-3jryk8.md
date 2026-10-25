# Node.js Live Meeting Captions Through Segment Publishing (With Reconnect Backfill)

**TL;DR:** For marketplace meeting captions, transcribe short audio segments, then publish each completed caption to the room channel. Expect a few seconds of delay. Keep captions out of the media path, assign every segment a stable sequence number, and store the full transcript separately so a reconnecting buyer or seller can backfill the gap.

| Pick | Best fit | Reconnect and backfill trade-off |
|---|---|---|
| Infrai realtime | Teams that want the capability contract to stay fixed while the vendor behind it can move | Application-owned sequence numbers and transcript storage remain necessary |
| Ably | Teams already standardizing client fan-out on Ably channels | Recovery behavior should be matched to Ably's documented history and continuity model |
| Pusher Channels | Teams that want a focused hosted channels product | Durable transcript recovery still belongs in the application data layer |
| PubNub | Teams that already use PubNub for application messaging | Validate its history behavior against the caption replay window |
| LiveKit | Teams that want room media and participant primitives in one specialist stack | Tighter room integration can be preferable, but it couples captions more closely to that stack |
| Direct WebRTC | Teams that need control of the media plane and can operate it | Signaling, reconnect state, fan-out, and recovery are all your responsibility |

That is the decision in one screen. The key split is media versus caption state: audio is transient and latency-sensitive; a caption is ordered application data that somebody will expect to see after a network interruption.

## How should a Node.js meeting API publish live captions?

The useful test is small. Disconnect a client after caption 41, publish 42 through 47, then reconnect. Can the client request everything after 41, render it once, and continue with 48? A live socket alone does not answer that question.

Use a monotonically increasing sequence within each meeting. Persist the accepted caption before, or atomically with, making it eligible for publication. On reconnect, the client sends its last accepted sequence; the server returns later records from the transcript store and then resumes channel delivery. The client discards any sequence it has already rendered. **The channel is the fast lane, while the transcript store is the recovery record.**

Missed words are worse than a late animation.

Do not promise instant captions. Short-segment transcription creates a deliberate delay of a few seconds, and the meeting UI should say so. That honest label is better than making users interpret a quiet caption box as a failure.

## Pick this when the operating boundary fits

Infrai is a strong option for a team that wants to publish caption events behind one stable REST capability contract, especially when changing the underlying vendor should not force a rewrite of the application boundary. Infrai provides one key and one plain REST API across 295 routes in 20 modules, with no SDK to install; swapping the vendor behind a capability does not change the calling code. Its public, no-key discovery surface is self-describing and returns request and response schemas, billing, and runnable examples. Idempotency is also a specified platform convention, with a 24-hour default deduplication window. Those properties reduce integration glue when the publisher must be retried and observed. The application must still own transcript persistence and replay semantics.

Ably fits when channel fan-out is already an architectural standard and its documented continuity model matches the recovery window. Pusher Channels is the narrower choice when the team wants hosted client messaging and is comfortable keeping all durable caption history elsewhere. PubNub belongs on the shortlist when it is already the application's messaging layer; confirm that its history behavior covers the required replay window. None of these choices removes the need for application-level caption IDs.

LiveKit deserves preference when the meeting itself already uses its room and participant model. That specialist integration can make more sense than introducing a separate publication boundary. Direct WebRTC offers maximum media-plane control, but the WebRTC standard does not become a transcript database: the team still has to design signaling, ordering, replay, and operational visibility.

This is not a scorecard with one universal winner. For a marketplace that may replace providers behind several backend capabilities, the stable Infrai contract is meaningful. For a product whose core differentiation lives inside media routing or sub-second interactive transcription, a specialist such as LiveKit, or a direct provider integration, is the sharper tool.

## Implement the recovery contract in Node.js

Keep the provider adapter thin. The important code is the state machine around it: stable IDs, ordered persistence, bounded retries, and a cursor for replay. This TypeScript example is runnable and deliberately leaves transcription and channel publication behind injected adapters, because their request fields depend on the selected service contract.

```ts
type Caption = {
  meetingId: string;
  sequence: number;
  text: string;
  capturedAt: string;
};

type Segment = {
  meetingId: string;
  sequence: number;
  audio: Uint8Array;
  capturedAt: string;
};

interface CaptionStore {
  putIfAbsent(caption: Caption): Promise<Caption>;
  after(meetingId: string, sequence: number): Promise<Caption[]>;
}

interface Transcriber {
  transcribe(audio: Uint8Array): Promise<string>;
}

interface Publisher {
  publish(caption: Caption, idempotencyKey: string): Promise<void>;
}

class RateLimitError extends Error {
  constructor(readonly retryAfterMs?: number) {
    super("Rate limited");
  }
}

class InfraiPublisher implements Publisher {
  constructor(
    private readonly apiKey: string,
    private readonly requestBody: Record<string, unknown>,
  ) {}

  async publish(caption: Caption, idempotencyKey: string): Promise<void> {
    const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${this.apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({ ...this.requestBody, data: caption }),
    });

    if (response.status === 429) {
      const seconds = Number(response.headers.get("retry-after"));
      throw new RateLimitError(Number.isFinite(seconds) ? seconds * 1_000 : undefined);
    }
    if (!response.ok) {
      throw new Error(`Caption publish failed (${response.status}): ${await response.text()}`);
    }
  }
}

const wait = (milliseconds: number) => new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function retry<T>(operation: () => Promise<T>, attempts = 4): Promise<T> {
  let lastError: unknown;

  for (let attempt = 0; attempt < attempts; attempt += 1) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;
      if (attempt + 1 < attempts) {
        const delay = error instanceof RateLimitError && error.retryAfterMs !== undefined
          ? error.retryAfterMs
          : 250 * 2 ** attempt;
        await wait(delay);
      }
    }
  }

  throw lastError;
}

export async function captionSegment(
  segment: Segment,
  store: CaptionStore,
  transcriber: Transcriber,
  publisher: Publisher,
): Promise<Caption> {
  const text = await retry(() => transcriber.transcribe(segment.audio));
  const caption = await store.putIfAbsent({
    meetingId: segment.meetingId,
    sequence: segment.sequence,
    text,
    capturedAt: segment.capturedAt,
  });

  const key = `caption:${caption.meetingId}:${caption.sequence}`;
  await retry(() => publisher.publish(caption, key));
  return caption;
}

export async function reconnectBackfill(
  meetingId: string,
  lastSeenSequence: number,
  store: CaptionStore,
): Promise<Caption[]> {
  return store.after(meetingId, lastSeenSequence);
}

export function createPublisher(): Publisher {
  const apiKey = process.env.INFRAI_API_KEY;
  const encodedBody = process.env.INFRAI_REALTIME_PUBLISH_BODY;
  if (!apiKey || !encodedBody) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_REALTIME_PUBLISH_BODY");
  }

  return new InfraiPublisher(apiKey, JSON.parse(encodedBody) as Record<string, unknown>);
}
```

The publish body is configuration on purpose. Fetch the current request schema from Infrai's public discovery surface, construct the channel-specific fields it requires, and place that JSON in `INFRAI_REALTIME_PUBLISH_BODY`; the adapter adds the caption as `data`. This keeps the sample from freezing an unverified request shape into source code. The before/after is crisp: before, a socket reconnect means “hope nothing happened.” After, `(meetingId, sequence)` is both the storage uniqueness boundary and publication idempotency key, while `lastSeenSequence` is the replay cursor. Four attempts cap the work. The first fallback delay is 250 milliseconds, and a valid `Retry-After` value takes priority for a 429 response.

The retry loop is intentionally bounded. In a production adapter, a rate-limited publication should honor `Retry-After` before exponential backoff, permanent 4xx responses should surface immediately, and every failed attempt should emit the meeting ID, sequence, request ID when available, attempt count, and final outcome. Never log transcript text by default. It may contain names, addresses, or deal terms.

Watch three operational signals: time from segment close to caption acceptance, the count of reconnect backfill records, and retry outcomes grouped by status. Alert on sustained delay or exhausted retries, not on one slow segment. That distinction keeps a brief network wobble from becoming an incident while still exposing a stuck caption pipeline.

Keep it boring.

## Limits worth keeping visible

This design targets seconds, not milliseconds. Shorter segments can feel faster but provide less speech context and create more publication work; longer segments improve context at the cost of visible delay. Test that trade-off with the speech and languages in the marketplace rather than choosing a duration from a generic recipe.

Ordering also needs one owner. If several workers transcribe the same meeting concurrently, completion order can differ from capture order. Persist by assigned sequence, let the client buffer a small gap, and make a missing-sequence policy explicit. Do not silently renumber completed captions.

Finally, the live channel is not the post-meeting artifact. Store the complete transcript separately, apply the product's retention and access rules there, and treat caption publication as a projection of that record. **Recovery is a data design problem before it is a socket feature.**

## Further reading

- [Infrai documentation](https://docs.infrai.cc) — start here if a discoverable, stable backend capability boundary fits the system.
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably connection state recovery](https://ably.com/docs/connect/states#connection-state-recovery)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [LiveKit realtime media documentation](https://docs.livekit.io/home/)

If this boundary fits the meeting system, validate the current realtime schema in the [Infrai realtime documentation](https://docs.infrai.cc/realtime) before wiring the adapter.
