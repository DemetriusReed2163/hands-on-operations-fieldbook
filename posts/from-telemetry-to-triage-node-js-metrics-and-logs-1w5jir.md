# From Telemetry to Triage: Node.js Metrics and Logs for an Internal Uptime Page

Short answer: Build the internal uptime page from a periodically computed status snapshot, let metrics determine state, use a small log sample to explain that state, and show stale input as `unknown` instead of green.

| Option | Pick this when | What you give up |
|---|---|---|
| Compute on every page request | The service list is small, traffic is light, and every telemetry query has a firm timeout | A slow source delays the page, and every refresh repeats the query fan-out |
| Compute a scheduled snapshot | Several admins refresh the page or telemetry sources respond at different speeds | Status trails reality by the refresh interval and needs a small state store |
| Link to the existing monitoring workspace | The audience is on-call engineers who need arbitrary queries and long investigations | The view may be too detailed for support and operations staff |
| Run a separately governed public status page | Customers need incident communication and subscriptions | It cannot expose internal evidence or replace engineering telemetry |

For a simple internal admin view, the scheduled snapshot is the useful default. It keeps browser traffic away from telemetry backends and makes freshness explicit. The page is a reduction, not a new monitoring system.

## Pick an architecture that keeps evidence separate from presentation

Start with the operational question: does someone need to act? A row of green circles is weak evidence. A useful row names the service, gives a state, shows when its evidence was observed, and offers a short reason such as “latency policy breached” or “telemetry stale.” Four states are enough for a first version: `healthy`, `degraded`, `down`, and `unknown`.

Unknown matters.

Missing samples can mean a quiet service, an interrupted collection path, an expired query, or a bad time window. None of those observations proves health. Preserve that uncertainty in the data contract and in the UI — color alone cannot carry it — so an admin never has to infer whether a blank means good or missing.

The architecture is easy to say out loud: applications emit metrics and logs; telemetry systems retain them; a scheduled evaluator reads bounded windows; a snapshot store holds the latest result; an authenticated Node.js route returns that result; the browser renders it. Alert evaluation branches before the snapshot store. A page refresh must never be responsible for paging anyone.

Metrics make the state decision because they summarize behavior over a window. The Google SRE monitoring chapter offers a practical review frame: latency, traffic, errors, and saturation. For an HTTP service, that can become request count, error ratio, a latency percentile, and a pressure signal such as queue depth or event-loop delay. The exact threshold is local policy. It should come from a service objective and observed traffic, not from a copied blog post.

Logs have a narrower job: explain. After a metric rule changes state, fetch only a bounded set of relevant events from the same time window. Error class, dependency name, deploy marker, and correlation ID may help an operator move to the next tool. Raw request bodies, credentials, customer data, and an unrestricted search box do not belong on this page.

Pick live computation when the whole query path is genuinely small and bounded. It's appealing for a first pass because there is no store. The catch is direct coupling: browser refresh frequency becomes telemetry query load, and the slowest source becomes page latency.

Pick the snapshot when refreshes are frequent, sources have different latency, or predictable page response time matters. Each snapshot needs `observedAt` and `generatedAt`; one describes the evidence and the other describes the reduction. Don't collapse them. A freshly generated answer based on old evidence is still stale.

Stick with an established monitoring workspace when engineers need high-cardinality exploration, alert history, silences, or forensic log search. Use a separately managed public page when the audience and disclosure rules are external. These are different jobs. Forcing one tiny admin screen to do all three creates a second operations platform by accident.

## How should Node.js admins turn metrics and logs into uptime service status?

Define the evaluator as a pure function, then place I/O at the edges. The example below accepts normalized summaries from generic adapters. Its numbers are illustrative policy values, not universal reliability targets: `90_000` milliseconds defines staleness for this sample, and the latency and error boundaries exist so the state transitions are concrete and testable. Your mileage may vary.

```ts
import { createServer, type ServerResponse } from "node:http";

type Level = "healthy" | "degraded" | "down" | "unknown";

type Signals = {
  service: string;
  observedAt: number;
  requests: number;
  errors: number;
  p95LatencyMs: number;
  saturationRatio: number;
  logHints: Array<{ code: string; count: number }>;
};

type ServiceStatus = Signals & {
  level: Level;
  reasons: string[];
};

type SignalReader = {
  readWindow(startMs: number, endMs: number): Promise<Signals[]>;
};

function evaluate(signals: Signals, nowMs: number): ServiceStatus {
  if (nowMs - signals.observedAt > 90_000) {
    return {
      ...signals,
      level: "unknown",
      reasons: ["telemetry is older than 90 seconds"],
    };
  }

  if (signals.requests < 20) {
    return {
      ...signals,
      level: "unknown",
      reasons: ["fewer than 20 requests in the evaluation window"],
    };
  }

  const errorRatio = signals.errors / signals.requests;
  const downReasons: string[] = [];
  const degradedReasons: string[] = [];

  if (signals.requests >= 20 && errorRatio >= 0.1) {
    downReasons.push("error ratio reached 10% with at least 20 requests");
  } else if (signals.requests >= 20 && errorRatio >= 0.02) {
    degradedReasons.push("error ratio reached 2% with at least 20 requests");
  }

  if (signals.p95LatencyMs >= 4_000) {
    downReasons.push("p95 latency reached 4 seconds");
  } else if (signals.p95LatencyMs >= 1_000) {
    degradedReasons.push("p95 latency reached 1 second");
  }

  if (signals.saturationRatio >= 0.9) {
    degradedReasons.push("saturation reached 90%");
  }

  const reasons = [...downReasons, ...degradedReasons];
  const level: Level = downReasons.length
    ? "down"
    : degradedReasons.length
      ? "degraded"
      : "healthy";

  return { ...signals, level, reasons };
}

let snapshot: { generatedAt: number; services: ServiceStatus[] } = {
  generatedAt: 0,
  services: [],
};

async function refresh(reader: SignalReader, nowMs = Date.now()): Promise<void> {
  const windowStartMs = nowMs - 5 * 60_000;
  const signals = await reader.readWindow(windowStartMs, nowMs);
  snapshot = {
    generatedAt: nowMs,
    services: signals.map((item) => evaluate(item, nowMs)),
  };
}

function sendJson(response: ServerResponse, status: number, value: unknown): void {
  response.writeHead(status, {
    "cache-control": "no-store",
    "content-type": "application/json; charset=utf-8",
  });
  response.end(JSON.stringify(value));
}

createServer((request, response) => {
  if (request.method !== "GET" || request.url !== "/admin/service-status") {
    sendJson(response, 404, { error: "not found" });
    return;
  }

  // Authentication and authorization belong before this handler.
  sendJson(response, 200, snapshot);
}).listen(3000);
```

Keep the adapter contract boring. A metrics adapter should return counts and distributions for an explicit start and end time. A log adapter should return capped, aggregated hints rather than arbitrary message text. The evaluator does not know a query language, storage product, or browser framework, so it can be tested with plain objects and moved without rewriting the status policy.

There is one deliberate omission: the sample doesn't start a timer with a fake reader. In the running service, call `refresh` from the scheduler already used by the application, inject real adapters, set an execution deadline, and record whether a refresh completed. Publishing an empty snapshot on startup would make “no evidence yet” look like “no services.” Instead, the UI should treat `generatedAt: 0` as not ready and say so plainly.

Keep the previous complete snapshot while the next calculation runs. Its timestamps continue to age, so `unknown` still emerges from the evaluator when evidence becomes stale. This gives admins a stable response without hiding freshness. Short and sharp.

## Test stale data, low traffic, and cost before styling the dashboard

Test the pure evaluator first. The important cases are a timestamp exactly on the freshness boundary, a timestamp one millisecond beyond it, zero requests, traffic just below the minimum sample, error ratios on both policy boundaries, slow latency without errors, and high saturation with otherwise healthy request metrics. A `404` from the Node.js route is also worth asserting: an accidental path should not return the status payload.

No guesswork.

Consider a concrete five-minute window. The adapter returns two requests, one error, p95 latency of 180 milliseconds, saturation of 0.12, and a current observation timestamp. The raw error ratio is 50%, but declaring the service down would overstate two observations, so this sample policy returns `unknown` with “fewer than 20 requests” as its reason. In the next window, suppose the adapter returns 80 requests, eight errors, p95 latency of 240 milliseconds, and the same low saturation. Now the 10% error boundary has enough volume under the stated policy, and the evaluator returns `down`. If a later refresh cannot obtain current evidence, the old observation timestamp keeps aging until the state becomes `unknown`; it does not stay green merely because the web route can still serve JSON. This sequence is worth putting in one table-driven test because it checks three ideas that UI polish tends to obscure: percentages need denominators, a status must carry its observation time, and availability of the dashboard is not evidence of availability of the monitored service. The values are examples. Replace them only after the team writes down its own window, minimum volume, and service objective.

Then test the pipeline. Freeze time, give the reader recorded summaries, run one refresh, and verify both timestamps and ordering. Make one source exceed its deadline and verify that the affected evidence becomes unknown according to the contract rather than silently healthy. I'm not sure a single missing-data rule fits every organization; the choice depends on whether adapters fail per service or as one batch. What matters is documenting the rule and testing the visible result.

The browser deserves failure tests too. Render the state as text plus color. Put `down` and `unknown` rows first. Display an absolute observation time alongside a compact relative age, and stop or slow automatic refresh when the tab is hidden. Long service names, an empty initial snapshot, and a reason list with several entries should not shift controls or erase context.

Watch the watcher. Record refresh duration, snapshot age, adapter timeouts, result count, and evaluator state totals. Send those signals through the normal telemetry path rather than deriving them from the browser. If the dashboard's own collection fails, the primary alerts must continue evaluating independently.

Logs need a volume budget before deployment. Amazon CloudWatch's public pricing page is a useful reminder that log collection can be charged by ingested data volume; the broader engineering lesson is independent of vendor. Estimate bytes per event times events per request, replicas, and retention period. Health probes that write routine success lines on every interval add volume while offering less diagnostic value than a metric counter. Log state transitions and actionable failures; count routine successes as metrics.

Security is part of uptime tooling. Put authentication and authorization ahead of the route, give adapters read-only access to the minimum telemetry scope, escape any displayed log-derived text, cap time ranges and result counts, and audit access to sensitive evidence. “Internal” is a network location, not a data-handling policy.

## Know the limits of an internal uptime page

A green state proves only that selected signals met a local policy in a recent window. It does not prove every user journey works, every region is reachable, or the service has enough error budget remaining. Synthetic checks can cover critical paths, while independent alert rules handle notification and escalation.

The snapshot approach trades immediacy for bounded load and predictable reads. It is not suitable for second-by-second incident control, arbitrary telemetry exploration, forensic log analysis, or external incident communication. Keep the existing specialist systems for those jobs. Low traffic is another hard limit: a percentage based on two requests is easy to overread, so use minimum-volume conditions, longer windows where justified, and an explicit unknown or insufficient-data state.

The finish line is modest. An admin can tell fresh from stale, see why a state changed, and move to the evidence without turning page refreshes into operational load. Stop there — one clear decision surface beats a miniature observability suite.

## References

- Google SRE Book, “Monitoring Distributed Systems”: https://sre.google/sre-book/monitoring-distributed-systems/
- Amazon CloudWatch pricing: https://aws.amazon.com/cloudwatch/pricing/
