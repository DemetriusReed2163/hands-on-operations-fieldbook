# Tracing Feature Flags at the Edge: AbortController Signals in Fetch Polling

Short answer: treat a remote feature-flag refresh as a bounded state machine, not as a boolean fetch: give each attempt its own deadline, preserve the last valid snapshot, prevent overlapping polls, and record enough evidence to distinguish local cancellation from an HTTP response or malformed data.

Start with the decision table. It keeps a slow control-plane request from becoming a vague `flag_fetch_failed` event and, more importantly, keeps that request from consuming the edge function's whole execution budget.

| Evidence at the boundary | Pick this action | Emit this outcome | Trade-off |
|---|---|---|---|
| Local deadline expires first | Abort this attempt and evaluate the cached snapshot | `deadline` | The answer may be stale |
| The server returns an HTTP status such as 429 | Honor the response policy; don't relabel it as a timeout | `response` plus status | Recovery may wait for a later refresh |
| The body isn't valid JSON or has the wrong shape | Reject the candidate snapshot | `invalid_payload` | The previous snapshot remains active |
| A refresh is already running | Skip or join it instead of starting another | `coalesced` | Poll timing becomes completion-relative |
| No valid snapshot exists | Apply a reviewed default | `default` | The default becomes part of product behavior |

## How should Node.js edge functions troubleshoot feature flag fetch timeouts?

First, draw the path in words: request enters; local snapshot answers; refresh fetch starts on a separate branch; a local deadline closes that branch; validation decides whether a new snapshot may replace the old one. This drawing exposes two clocks. The invocation budget protects user work, while the refresh deadline protects only the remote attempt. They aren't interchangeable.

The cleanest option for most flag reads is **evaluate locally and refresh out of band**. A last-known-good snapshot makes control-plane latency independent from the request path. Attach a fetch timestamp and a configuration version to that snapshot, because “cached” without age or identity isn't useful during an incident. The catch is freshness: a flag change won't be visible until a refresh succeeds. If a flag enforces an authorization or safety boundary, a stale value may be unacceptable; use a reviewed fail-closed default or move that policy to a system with the required consistency guarantee.

Direct, request-time fetching is the smaller design when traffic is low, the remote service has a firm latency objective, and no meaningful fallback exists. It is also easier to reason about at first. Its limit is coupling. Every remote delay now spends the same budget needed by the edge handler, so the deadline must leave time to evaluate fallback behavior and produce the response.

Shared snapshot storage is the serious third option when many short-lived instances need a more consistent view than warm-process memory can provide. It improves convergence across instances but introduces another network boundary, another availability decision, and another freshness policy. Don't add it merely because “edge” sounds distributed. Add it when the allowed divergence between instances is explicit and smaller than process-local refresh can provide.

The diagnostic sequence follows the boundaries. Check whether the local signal aborted. If it didn't, check whether `fetch` returned a response. Then validate decoding and schema. Only after classification should retry logic run. A 429 response, an invalid body, and a locally expired timer may all leave the same old snapshot in use, but they demand different operational action.

Small labels win.

## Pick a polling model before tuning the interval

Fixed-rate polling starts work every N milliseconds regardless of whether the previous attempt finished. That can turn one slow request into overlapping attempts and a noisy burst of cancellations. Completion-relative polling starts the next timer only after the current refresh settles. The effective cadence becomes `refresh duration + interval`, but concurrency stays bounded. For feature flags, that trade is usually easier to operate than a nominally precise cadence that multiplies in-flight work.

A single-flight guard completes the model. If an invocation notices that a refresh is already in progress, it can reuse that promise or skip the duplicate. Either choice is observable. Record `coalesced` rather than pretending no refresh was requested, and avoid unbounded flag-key labels in metrics. A bounded outcome vocabulary makes alerts readable: `fresh`, `deadline`, `response`, `invalid_payload`, `transport`, `coalesced`, and `default` are enough to locate the failing boundary without putting credentials or response bodies into logs.

Edge lifecycle rules complicate background timers. Some runtimes provide an explicit mechanism for extending work beyond the response; others may freeze execution. I'm not sure which guarantee applies to an unnamed deployment target, so verify its lifecycle contract before depending on an in-memory poller. Where post-response work isn't guaranteed, trigger refresh during invocations or from an external scheduler, and keep the same deadline and single-flight rules.

Alerts should describe user-visible risk, not a single slow sample. A rising `deadline` rate is diagnostic. A snapshot age beyond the team's declared tolerance is actionable. Sustained `default` evaluation is more urgent when no previous snapshot exists. This separation is the before/after that matters: before, every problem increments one failure counter; after, operators can tell whether to inspect latency, response policy, payload compatibility, scheduling, or initial configuration.

No overlap. Ever.

## Implement the deadline and the evidence together

The following TypeScript uses a generic `/flags` contract: a successful response contains a JSON object whose `flags` member maps names to booleans. Replace that contract with the documented endpoint and schema of the system you operate. The important part is the local behavior around it, not the path name.

```ts
type FlagSnapshot = Readonly<{
  flags: Readonly<Record<string, boolean>>;
  fetchedAtMs: number;
}>;

type RefreshOutcome =
  | { kind: "fresh"; snapshot: FlagSnapshot; elapsedMs: number }
  | { kind: "deadline"; elapsedMs: number }
  | { kind: "response"; status: number; elapsedMs: number }
  | { kind: "invalid_payload"; elapsedMs: number }
  | { kind: "transport"; errorName: string; elapsedMs: number };

function isFlagMap(value: unknown): value is Record<string, boolean> {
  if (typeof value !== "object" || value === null || Array.isArray(value)) return false;
  return Object.values(value).every((flag) => typeof flag === "boolean");
}

async function refreshFlags(
  endpoint: URL,
  deadlineMs: number,
  now: () => number = Date.now,
): Promise<RefreshOutcome> {
  const startedAtMs = now();
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort("feature-flag deadline"), deadlineMs);

  try {
    const response = await fetch(endpoint, { signal: controller.signal });
    if (!response.ok) {
      return {
        kind: "response",
        status: response.status,
        elapsedMs: now() - startedAtMs,
      };
    }

    const body: unknown = await response.json();
    const flags =
      typeof body === "object" && body !== null && "flags" in body
        ? (body as { flags: unknown }).flags
        : undefined;

    if (!isFlagMap(flags)) {
      return { kind: "invalid_payload", elapsedMs: now() - startedAtMs };
    }

    return {
      kind: "fresh",
      snapshot: { flags, fetchedAtMs: now() },
      elapsedMs: now() - startedAtMs,
    };
  } catch (error) {
    const elapsedMs = now() - startedAtMs;
    if (controller.signal.aborted) return { kind: "deadline", elapsedMs };
    return {
      kind: "transport",
      errorName: error instanceof Error ? error.name : "UnknownError",
      elapsedMs,
    };
  } finally {
    clearTimeout(timer);
  }
}

function createSingleFlightPoller(
  refresh: () => Promise<RefreshOutcome>,
  intervalMs: number,
  observe: (outcome: RefreshOutcome) => void,
): { start: () => void; stop: () => void } {
  let stopped = true;
  let inFlight: Promise<void> | undefined;
  let timer: ReturnType<typeof setTimeout> | undefined;

  const run = (): Promise<void> => {
    if (inFlight) return inFlight;
    inFlight = refresh()
      .then(observe)
      .finally(() => {
        inFlight = undefined;
        if (!stopped) timer = setTimeout(() => void run(), intervalMs);
      });
    return inFlight;
  };

  return {
    start: () => {
      if (!stopped) return;
      stopped = false;
      void run();
    },
    stop: () => {
      stopped = true;
      if (timer) clearTimeout(timer);
    },
  };
}
```

Notice the order. `response.ok` is checked before decoding, the candidate payload is validated before it can replace a snapshot, and `controller.signal.aborted` determines whether this code caused the cancellation. The deadline timer is always cleared. The poller schedules its successor in `finally`, after the current promise settles, so a slow attempt changes cadence rather than concurrency.

Tests should exercise each transition with a fake clock and a controlled fetch implementation: a response that waits for the abort signal, a 429 response, malformed JSON, a valid body with a non-boolean flag value, and a valid snapshot. Also assert that repeated `start` calls don't create multiple requests. This is more useful than sleeping for the real interval, and it gives the telemetry contract the same scrutiny as the data contract.

For structured logs, emit one event when an attempt settles. Include outcome, elapsed milliseconds, deadline milliseconds, snapshot age, runtime class, and a non-secret configuration version. Metrics can count outcomes and observe both refresh duration and snapshot age. Tracing is useful when refresh happens inside a request: put the local deadline on the refresh span and add a separate fallback-evaluation event. The custom-appender model in Logback is a useful cross-ecosystem reminder that observability output has its own lifecycle and delivery semantics; logging must not become an unbounded second failure path.

## Know where this field guide stops

Polling is not suitable when every evaluation must observe an immediately current value. Process-local snapshots are also the wrong choice when instances require tight convergence, and fail-open defaults are wrong for policy decisions whose stale state would violate a security boundary. In those cases, choose an architecture with the required consistency and availability contract, then keep the same outcome classification at its network edges.

There is no universal timeout number. Measure the remote latency distribution, reserve enough of the invocation budget for fallback and response work, and set a freshness tolerance from product risk. Your mileage may vary across runtimes and traffic shapes. The durable rule is shorter: bound attempts, preserve valid state, prevent overlap, and report the boundary that actually failed.

## References

- https://logback.qos.ch/manual/appenders.html

## Further reading

- Logback appenders and custom output handling: https://logback.qos.ch/manual/appenders.html
