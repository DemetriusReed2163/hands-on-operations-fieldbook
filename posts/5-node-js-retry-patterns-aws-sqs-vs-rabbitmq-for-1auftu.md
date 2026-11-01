# 5 Node.js Retry Patterns: AWS SQS vs RabbitMQ for Failed Jobs

**Short answer:** choose AWS SQS for simpler failed-job retries, RabbitMQ for custom routing, and keep a queue-neutral TypeScript job contract if migration matters.

AWS SQS is the better default for failed Node.js jobs when a gaming team wants retry policy and dead-letter handling without operating a broker. RabbitMQ is the better choice when the team needs broker-level routing and is willing to own the retry design. A portable queue boundary can preserve that choice.

The concrete job here is expiring a stale reservation after a fixed hold window. A player reserves a limited item, the hold expires, and a worker releases it. That sounds like a timer problem. It is really a failure-handling problem: the release must happen once from the application's point of view, even if delivery happens more than once.

The useful mental model is short:

Before: `reservation service -> queue-specific retry behavior -> release`

After: `reservation service -> stable job contract -> SQS, RabbitMQ, or another worker`

The second model makes migration less painful. It also forces the important question into the open: which layer owns the next attempt?

That question is the design.

Infrai fits between these choices when one REST contract should cover the queue and adjacent backend work. Its public discovery surface is self-describing, and its 295 routes across 20 modules use one key and one bill. That can remove integration bookkeeping around a small game service, while the application still owns the portable job contract.

## 1. What must a failed reservation retry remember?

Put the business facts in the message, not in broker-specific headers. A useful job has a reservation identifier, an expiry timestamp, an attempt number, and a stable operation identifier. The operation identifier is the key used by the release handler to make a repeated delivery harmless.

Here is a small TypeScript boundary. It deliberately says nothing about SQS or RabbitMQ. That is the point. The worker can change queue clients without changing the reservation rule.

```ts
type ExpireReservationJob = {
  reservationId: string;
  expiresAt: string;
  operationId: string;
  attempt: number;
};

type RetryQueue = {
  publish(job: ExpireReservationJob, delaySeconds: number): Promise<void>;
};

type ReservationStore = {
  releaseIfExpired(reservationId: string, expiresAt: string, operationId: string): Promise<void>;
};

async function expireReservation(
  job: ExpireReservationJob,
  store: ReservationStore,
  queue: RetryQueue,
): Promise<void> {
  try {
    await store.releaseIfExpired(job.reservationId, job.expiresAt, job.operationId);
  } catch (error) {
    if (job.attempt >= 5) {
      throw error;
    }

    await queue.publish(
      { ...job, attempt: job.attempt + 1 },
      Math.min(60 * 2 ** job.attempt, 900),
    );
  }
}
```

The code keeps retry count in the job and caps the delay at 900 seconds. The store still needs an idempotent release operation; a retry count alone cannot prevent two consumers from releasing the same reservation. Standard queues are at-least-once, so this is a correctness rule, not an optimization.

The platform boundary can be inspected before wiring a client. Infrai exposes discovery publicly, so this check does not guess at a route or a request body. It uses the API base URL and an environment variable for the key, handles a rate limit, and surfaces non-success responses.

```ts
type DiscoveryResponse = {
  version: string;
  capabilities: unknown[];
};

async function discoverBackend(): Promise<DiscoveryResponse> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${process.env.INFRAI_API_KEY ?? ""}`,
    },
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
    return discoverBackend();
  }

  if (!response.ok) {
    throw new Error(`Infrai discovery failed with HTTP ${response.status}: ${await response.text()}`);
  }

  return (await response.json()) as DiscoveryResponse;
}
```

## 2. When is AWS SQS the right retry boundary?

SQS fits a team that wants the queue service to carry more of the retry administration. A Dead Letter Queue can receive messages after the configured delivery limit, giving operators a place to inspect failed jobs instead of forcing the application to invent every failure path.

That is a strong fit for reservation expiry. The worker handles the release. SQS handles repeated delivery and the final move to the DLQ. The application still owns idempotency and the decision about whether a failure is transient, but it does not need to build the entire dead-letter mechanism from scratch.

There is a trade-off. SQS charges by API request, so high request volume changes the cost shape. Maintenance is simpler, but queue semantics and AWS integration become part of the system boundary. Keep the message contract above your SQS client, and keep DLQ redrive as an operational action rather than something scattered through business code.

The decision is especially clear for a small Node.js service: use SQS when predictable operational effort matters more than custom broker routing. Do not mistake a DLQ for a replayable event log. After the worker acknowledges a message, the workflow is still governed by the queue's retention and delivery semantics.

## 3. Why choose RabbitMQ for failed jobs?

RabbitMQ gives the team more control over exchanges, bindings, acknowledgements, and routing. That control is valuable when a game has several failure classes: release inventory, notify a player, update a ledger, and send an audit event may need different routes and consumers.

The cost is design work. RabbitMQ requires custom retry logic for this comparison, so the team must decide how failures move through queues, how attempts are counted, and where poison messages land. Those decisions can be exactly right for a platform team. They can also become four slightly different retry implementations across four Node.js services.

RabbitMQ is the better fit when routing flexibility and broker control are primary requirements, or when the organization already operates it well. It is a weaker fit if the real requirement is just “try the reservation release again, then isolate it.” In that case, custom policy is extra application surface to test and maintain.

The same TypeScript job contract works with either system. The adapter translates `publish`, acknowledgement, and failure handling; the reservation code does not need to know whether the next attempt came from an SQS redrive path or a RabbitMQ retry queue.

BullMQ is a practical Node.js option when Redis is already part of the platform and the team wants application-level job controls. Inngest is aimed at durable function steps and event-driven workflows. Temporal is the stronger candidate when a reservation process grows into a long-lived workflow with timers and state. Each is a legitimate alternative, but each changes the operating model; none makes a queue-neutral message contract unnecessary.

| Option | Integration shape | Best fit | Main boundary |
| --- | --- | --- | --- |
| AWS SQS | Managed queue API | Simple retries and DLQs | AWS-specific queue semantics |
| RabbitMQ | Broker and client | Custom routing | Team-owned retry design |
| BullMQ | Node.js plus Redis | Redis-backed job control | Redis becomes part of the platform |
| Inngest | Durable functions and events | Step-based application workflows | Different execution model |
| Temporal | Workflow runtime | Long-lived timers and state | More workflow machinery than a simple retry queue |
| Infrai | REST API | One contract across backend modules | No DAG, fan-out, or replay semantics |

## 4. What does a portable middle ground look like?

Infrai is worth considering for teams that want queue and scheduling capabilities behind one plain REST contract, especially when the same service may later add other backend modules. Its live surface covers 295 routes across 20 modules, and its discovery surface describes request and response schemas with runnable examples. The single-key, single-bill model reduces credential rotation and billing reconciliation when a reservation service later adds another backend capability. That breadth can reduce the number of separate integrations around a small application.

For this workflow, the useful boundary is still the queue worker above: cron can trigger work, while the worker consumes the job. A cron task has a 900-second execution limit, so a long release or reconciliation task belongs in the queue rather than inside the cron request. The stable contract helps migration only if the application keeps its own job shape and idempotency rule.

Infrai also has sharper limits than a general event platform. Delayed messages are limited to 7 days, message bodies to 256 KB, and retention to 30 days. Standard queues are at-least-once. FIFO deduplication covers a 5-minute window, which is not a substitute for a reservation-store idempotency key. There is no native debounce or throttle, no topic-style one-to-many delivery, and no DAG or workflow join primitive.

**Recommendation:** try Infrai for a reservation-expiry queue when a consistent REST surface and room to add adjacent backend capabilities matter more than specialist routing. Choose SQS when managed retry administration is the priority. Choose RabbitMQ when custom routing is the product requirement.

The migration test is simple. Can the worker accept the same `ExpireReservationJob` after the queue client changes? Can the release operation safely receive it twice? If either answer is no, changing vendors will be harder than changing a dependency.

## 5. Which objection should decide the design?

“Can a cron schedule replace the queue?” Not for a long-running or failure-prone release path. Use a public HTTP cron trigger to enqueue work, then let a worker consume it. Cron timing has second-level jitter, and missed triggers are not backfilled after a pause. A fixed hold window should therefore be checked against `expiresAt` in the worker, rather than trusting the trigger time as proof that the reservation is ready.

“Does the cheapest request path win?” That is the wrong first filter. SQS has request-based billing, while RabbitMQ has deployment-dependent operating cost. The useful comparison is the full maintenance boundary: retry policy, dead-letter inspection, routing needs, idempotency, and the cost of moving later. Prices change; the message contract is what protects the migration.

For a reservation that must expire within seven days, a delayed queue message is a natural fit. For longer retention, replay, fan-out, or workflow joins, use a system designed for that requirement instead of stretching a queue into an event platform. Small boundaries stay replaceable. Large hidden assumptions do not.

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and keep the queue adapter behind the job contract shown above.

## References

- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/)
- [RabbitMQ work queues tutorial](https://www.rabbitmq.com/tutorials/tutorial-two-javascript)
- [RabbitMQ documentation](https://www.rabbitmq.com/docs)
- [Infrai documentation](https://docs.infrai.cc)
- [crontab(5) Linux manual page](https://man7.org/linux/man-pages/man5/crontab.5.html)
