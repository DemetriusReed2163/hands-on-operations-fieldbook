# Batched Metric Updates for Support Dashboards — Channel Reconnect Without Missing Fan-Out

Short answer: batch metric samples on the server, assign every batch a monotonically increasing sequence, and retain a small replay window per dashboard channel. On reconnect, the browser presents its last applied sequence; the server replays newer batches before resuming live delivery. This gives a customer-support dashboard an explicit at-least-once contract, while the reducer makes duplicate delivery harmless.

A video room and its dashboard have different jobs. WebRTC carries the interactive media session between participants. The dashboard channel carries operational state: waiting callers, active rooms, token issuance outcomes, and media-health summaries. Keep those paths separate. A delayed chart must never delay a call, and a fresh dashboard connection must not imply that a video room needs a new scoped token.

## What guarantee does the dashboard actually need?

Start with the delivery contract, because "reconnect handling" is otherwise too vague to test. For operational metrics, exactly-once transport is an expensive promise and usually the wrong target. Use **at-least-once delivery within a bounded replay window**, then make application deterministic with sequence-based deduplication.

Here is the before/after mental model. Before: each metric update is pushed immediately, a socket disappears, and the client reconnects with no way to distinguish a gap from a quiet interval. After: the server closes a short batch, labels it `417`, stores it, and fans it out. A returning client says `lastSequence=414`; the server sends `415`, `416`, and `417` in order, then attaches the client to the live stream.

That contract exposes the real trade-off. A larger replay window tolerates longer disconnections but consumes more memory. A shorter batch interval reduces chart latency but increases fan-out work. Pick both from the operation's target, then alert when reality exceeds either bound. Do not hide the boundary.

For a support room, scope every stream and token to a tenant and room. The metric channel can use a coarser tenant-dashboard scope, but authorization still belongs on the server. Browser input must never choose an unrestricted channel name.

## How should a dashboard channel push batched metric updates?

The example below uses an in-memory channel so the mechanism stays visible. It flushes every 250 milliseconds, keeps 120 batches, emits newline-delimited JSON through an Express response, and accepts a last applied sequence on reconnect. Those numbers are example policy values, not universal defaults. In a multi-process deployment, replace the in-memory log and subscriber set with shared infrastructure that preserves the same interface and ordering rule.

```ts
import express, { Request, Response } from "express";

type Metric = {
  roomId: string;
  name: "participants" | "token_issued" | "packet_loss_ratio";
  value: number;
  observedAt: string;
};

type Batch = { sequence: number; metrics: Metric[] };
type Subscriber = { response: Response; nextSequence: number };

class DashboardChannel {
  private pending: Metric[] = [];
  private history: Batch[] = [];
  private subscribers = new Set<Subscriber>();
  private sequence = 0;

  constructor(
    private readonly maxHistory: number,
    flushEveryMs: number,
  ) {
    const timer = setInterval(() => this.flush(), flushEveryMs);
    timer.unref();
  }

  record(metric: Metric): void {
    this.pending.push(metric);
  }

  subscribe(response: Response, lastSequence: number): () => void {
    const oldest = this.history.at(0)?.sequence ?? this.sequence + 1;
    if (lastSequence + 1 < oldest) {
      response.write(JSON.stringify({ type: "reset_required", latest: this.sequence }) + "\n");
    } else {
      for (const batch of this.history) {
        if (batch.sequence > lastSequence) this.write(response, batch);
      }
    }

    const subscriber = { response, nextSequence: this.sequence + 1 };
    this.subscribers.add(subscriber);
    return () => this.subscribers.delete(subscriber);
  }

  private flush(): void {
    if (this.pending.length === 0) return;

    const batch: Batch = {
      sequence: ++this.sequence,
      metrics: this.pending.splice(0),
    };
    this.history.push(batch);
    if (this.history.length > this.maxHistory) this.history.shift();

    for (const subscriber of this.subscribers) {
      if (batch.sequence < subscriber.nextSequence) continue;
      this.write(subscriber.response, batch);
      subscriber.nextSequence = batch.sequence + 1;
    }
  }

  private write(response: Response, batch: Batch): void {
    response.write(JSON.stringify({ type: "metrics", ...batch }) + "\n");
  }
}

const app = express();
const channel = new DashboardChannel(120, 250);
app.use(express.json());

app.post("/internal/metrics", (request: Request, response: Response) => {
  channel.record(request.body as Metric);
  response.sendStatus(202);
});

app.get("/dashboard/stream", (request: Request, response: Response) => {
  // Production code derives tenant scope from authenticated server-side claims.
  const parsed = Number.parseInt(String(request.query.lastSequence ?? "0"), 10);
  const lastSequence = Number.isSafeInteger(parsed) && parsed >= 0 ? parsed : 0;

  response.status(200);
  response.setHeader("Content-Type", "application/x-ndjson");
  response.setHeader("Cache-Control", "no-store");
  response.flushHeaders();

  const unsubscribe = channel.subscribe(response, lastSequence);
  request.on("close", unsubscribe);
});

app.listen(3000);
```

The browser reducer should persist `lastSequence` only after applying the whole batch. If it receives a sequence it has already applied, it drops the duplicate. If the server sends `reset_required`, the client fetches a fresh snapshot, atomically replaces its displayed aggregates, stores the snapshot sequence, and reconnects from there. That reset is part of the protocol.

One subtle race deserves attention: replay and live subscription must share one serialization point. In the example, synchronous execution completes `subscribe` without an `await`, so a timer callback cannot interleave halfway through it. Once history moves to an external broker or database, preserve that property explicitly. Otherwise, a batch can land between the replay query and live attachment. Imagine the stored history ending at `41`: replay returns through `41`, batch `42` is appended before the subscriber attaches, and the live stream begins with `43`. Every individual operation succeeded, yet the dashboard skipped `42`. Treat replay plus attachment as one ordered transition, or take a snapshot at a known sequence and stream strictly after it. The chart can look calm while silently missing a point.

Test that gap.

## Observe the recovery path, not just the happy path

A live chart is a weak health signal. Instrument the sequence protocol itself. Record batch size and flush duration; count reconnects, duplicates discarded, replayed batches, and resets caused by an expired window. Track each subscriber's sequence lag as `latestSequence - lastAppliedSequence`. Alert on sustained lag or reset rate, because both reveal a dashboard that is technically connected but operationally stale. The trade-off is explicit: tighter alert thresholds detect gaps sooner but also page on harmless short reconnects, so set them from the declared replay objective rather than an arbitrary round number.

Keep labels bounded. Tenant and room identifiers are useful in logs, where an operator can search a specific support case, but they can create uncontrolled metric cardinality. Aggregate telemetry by deployment, channel class, or outcome, and put request-level identifiers in structured logs.

Test the ugly orderings:

- Connect at sequence `10`, interrupt delivery during batch `11`, publish through `14`, then reconnect from `10`. The reducer must finish at `14` with no double counting.
- Reconnect from a sequence older than retained history and verify a snapshot reset.
- Run two browser sessions with different cursors and confirm that each advances independently.
- Terminate the server between accepting a metric and closing its batch. This forces a decision about durable storage.

Crash semantics matter.

The sample's `202` response means accepted by this process, not durably committed. If losing that metric during a process crash is unacceptable, acknowledge only after a durable append. Then build dashboard batches from that log. The extra write adds latency and operational machinery, but the guarantee is honest.

## But what about multiple server instances?

Process-local state stops being authoritative as soon as requests can reach different instances. Use a shared ordered log partitioned by tenant or dashboard channel, and give each batch a sequence from that partition. Every web node may fan out records, but a client must observe one ordered stream for its scope.

Avoid generating independent counters on separate nodes. Two batches labeled `52` cannot be deduplicated correctly, and arrival time cannot repair the ambiguity. The diagram in words is short: producers append metrics; one batcher closes windows; an ordered log retains batches; stateless edge nodes stream them; clients reduce by sequence.

Capacity planning follows from the contract. Estimate retained bytes from batches per second, average encoded batch size, and replay duration. Measure it under a reconnect wave, not just a steady connection count. Backpressure also needs a policy: disconnect a subscriber whose buffered writes keep growing, then let it recover by replay or snapshot. Allowing one slow browser to accumulate unbounded memory is not delivery reliability.

## Do media events belong in the same stream?

Use WebRTC for the room's media session and its standardized peer-connection behavior. Feed selected operational observations into the dashboard pipeline, but do not treat dashboard delivery as control-plane authority for the call. A dashboard reconnect should restore the operator's view. It should not mint a broader token, renegotiate media, or replay a privileged room command.

Scoped video-room tokens should be short-lived capabilities issued by a trusted server after authorization. Keep their issuance result in telemetry as a bounded status such as success or denied; never publish the credential itself. Redact session descriptions, connectivity credentials, and user-provided support text from general metric payloads. Operational visibility does not require copying sensitive session material into every subscriber.

The final decision is crisp: choose a batch interval from acceptable display delay, choose replay retention from the reconnect objective, and state what happens beyond it. **Sequences plus an explicit snapshot reset make the boundary testable.** Durability before acknowledgement is a separate choice; pay for it when losing an accepted observation would violate the support operation's contract.

## References

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
