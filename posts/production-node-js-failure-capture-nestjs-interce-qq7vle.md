# Production Node.js Failure Capture: NestJS Interceptors, HTTP, Schedulers, Consumers

The operational constraint changes the design: NestJS HTTP requests, scheduled work, and queue consumers don't share one execution boundary. **Short answer:** put normalization and capture behind one small port, then call it from a global HTTP exception filter and from explicit `try/catch` boundaries around every cron job and queue worker; use an interceptor only to attach request context and timing.

That split is the important part. A filter can observe an exception leaving the HTTP path. It can't see work that began in a scheduler or consumer. Pretending otherwise creates a quiet blind spot in production.

## What should a NestJS error tracking example capture from HTTP exceptions, cron jobs, and queue workers?

Capture the same event shape at each boundary: exception name, safe message, stack, operation, execution kind, timestamp, and a correlation key. Keep transport details behind an interface so application code doesn't know where events go.

The before/after model is crisp. Before: each runtime path logs a different object, some failures are swallowed, and an HTTP-oriented filter is treated as universal. After: each boundary converts `unknown` into the same envelope, submits it once, and preserves the runtime's normal failure semantics.

In words, the flow is: **entry point -> context -> business operation -> thrown value -> normalizer -> capture port -> original failure behavior**. For HTTP, original behavior means returning the framework's response. For background work, it means rethrowing so the scheduler or queue runtime can apply its configured policy. Capture is evidence. It is not control flow.

Don't report every expected client response as an incident. The exact policy belongs to the application, but it should be explicit and tested: validation outcomes can remain ordinary responses, while unexpected exceptions enter the tracking stream. Scrub secrets and personal data before the event crosses the capture port. Also decide what happens when reporting itself cannot complete. The application must have a bounded fallback, such as a structured local log; otherwise telemetry can delay the work it is meant to describe.

## Build one capture port, then keep the adapters thin

Start with the smallest useful contract. This example intentionally avoids a vendor SDK and leaves delivery to an injected adapter.

```ts
export type ExecutionKind = "http" | "cron" | "queue";

export interface FailureEvent {
  kind: ExecutionKind;
  operation: string;
  correlationId: string;
  errorName: string;
  message: string;
  stack?: string;
  occurredAt: string;
}

export interface FailureCapture {
  capture(event: FailureEvent): Promise<void>;
}

export function toFailureEvent(
  thrown: unknown,
  context: Pick<FailureEvent, "kind" | "operation" | "correlationId">,
): FailureEvent {
  const error = thrown instanceof Error ? thrown : new Error(String(thrown));

  return {
    ...context,
    errorName: error.name,
    message: error.message,
    stack: error.stack,
    occurredAt: new Date().toISOString(),
  };
}
```

Now wire that contract to the three execution boundaries. The interceptor sets a request correlation ID before controller work begins. The exception filter owns final HTTP capture and response delegation. The cron and queue wrappers capture, then rethrow.

```ts
import {
  ArgumentsHost,
  CallHandler,
  Catch,
  ExceptionFilter,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Cron } from "@nestjs/schedule";
import { randomUUID } from "node:crypto";
import { Observable } from "rxjs";

type RequestWithCorrelation = {
  method: string;
  url: string;
  correlationId?: string;
};

@Injectable()
export class CorrelationInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<RequestWithCorrelation>();
    request.correlationId ??= randomUUID();
    return next.handle();
  }
}

@Catch()
export class HttpFailureFilter implements ExceptionFilter {
  constructor(private readonly failures: FailureCapture) {}

  async catch(thrown: unknown, host: ArgumentsHost): Promise<void> {
    const request = host.switchToHttp().getRequest<RequestWithCorrelation>();

    await this.failures.capture(
      toFailureEvent(thrown, {
        kind: "http",
        operation: `${request.method} ${request.url}`,
        correlationId: request.correlationId ?? randomUUID(),
      }),
    );

    throw thrown;
  }
}

@Injectable()
export class ReconciliationJob {
  constructor(private readonly failures: FailureCapture) {}

  @Cron("0 * * * *")
  async run(): Promise<void> {
    const correlationId = randomUUID();
    try {
      await this.reconcile();
    } catch (thrown) {
      await this.failures.capture(
        toFailureEvent(thrown, {
          kind: "cron",
          operation: "reconciliation.run",
          correlationId,
        }),
      );
      throw thrown;
    }
  }

  private async reconcile(): Promise<void> {
    // Application work belongs here.
  }
}

@Injectable()
export class InvoiceQueueWorker {
  constructor(private readonly failures: FailureCapture) {}

  async process(job: { id: string; data: unknown }): Promise<void> {
    try {
      await this.generateInvoice(job.data);
    } catch (thrown) {
      await this.failures.capture(
        toFailureEvent(thrown, {
          kind: "queue",
          operation: "invoice.generate",
          correlationId: job.id,
        }),
      );
      throw thrown;
    }
  }

  private async generateInvoice(_data: unknown): Promise<void> {
    // Application work belongs here.
  }
}
```

One detail deserves scrutiny: the filter above demonstrates capture and rethrow, not a complete HTTP response strategy. In a real application, compose capture with the existing framework or application exception handler so status codes and response bodies keep their established behavior. Don't casually replace that policy in an observability patch.

The delivery adapter also needs a deadline. I'm not sure one timeout fits every service; latency budgets and shutdown behavior vary. What matters is that the limit is explicit, capture failure is handled without recursion, and tests prove the business exception still reaches its owning runtime.

## Test the blind spots before deployment

The most useful test matrix has rows for HTTP, cron, and queue execution, then columns for a normal `Error`, a non-`Error` thrown value, capture-adapter failure, and correlation propagation. Assert both sides of each boundary: the normalized event and what the runtime sees afterward.

| Boundary | Correlation source | Required post-capture behavior |
| --- | --- | --- |
| HTTP | Request ID | Existing exception policy handles the response |
| Cron | Run ID | The scheduled method rejects |
| Queue | Job ID | The consumer rejects for its runtime to handle |

Context wins.

For an HTTP test, trigger a controller exception and verify that the event says `http`, includes the request correlation ID, and leaves response handling to the configured exception policy. For a cron test, invoke `run()` directly with a business dependency that rejects; verify one capture and the same rejection afterward. For a queue test, use a stable job ID, reject the handler, and verify that ID reaches the event before the error is rethrown. Then exercise the uncomfortable branch: make the capture adapter reject while the business operation is already failing. The assertion should show which failure owns control flow, confirm that reporting cannot recurse into itself, and prove that the fallback contains enough fields to connect the local record to the original request, run, or job. That single case exposes designs that look tidy in a happy-path unit test but erase the first exception when telemetry has trouble.

Big payoff.

Avoid snapshotting full stack traces because paths and line numbers churn. Match the error name, message, kind, operation, and correlation ID. Then add one integration test with the real delivery adapter pointed at a local fake that records requests. This catches serialization and timeout mistakes without binding the whole suite to an external service.

Deployment needs a rollback lever because capture sits on failure paths, where accidental latency is especially expensive. A feature toggle can select the new adapter while leaving the boundary code in place. Martin Fowler's feature-toggle guidance distinguishes release, experiment, ops, and permission toggles; this is an ops control, so give it an owner and a removal date rather than letting permanent branching spread through the handlers. GrowthBook is one example of an open-source flag and experimentation platform, but the implementation is less important than keeping the decision at the composition root.

```ts
export function buildFailureCapture(config: {
  trackingEnabled: boolean;
  primary: FailureCapture;
  fallback: FailureCapture;
}): FailureCapture {
  return config.trackingEnabled ? config.primary : config.fallback;
}
```

Roll out by execution kind. Enable HTTP first, then cron, then queue consumers; watch event volume, adapter latency, and duplicate correlation IDs at each step. Your mileage may vary, especially when background work has a much higher failure rate than request traffic. A staged switch makes that difference visible before it becomes noise.

## Can a global interceptor replace all those exception boundaries?

No. An HTTP interceptor is valuable for request-scoped context and timing, but it isn't a universal wrapper around scheduler callbacks or queue consumers. Keep cross-cutting request concerns in the interceptor and failure capture at the boundary that actually owns the execution.

There is a tempting alternative: process-level handlers for uncaught exceptions and unhandled promise rejections. Treat those as a last-resort signal, not the primary integration. At that point, operation names and job context may already be gone, and continuing the process may be unsafe depending on the failure. The local boundary has better context and a clear owner.

The catch is repetition. Every background entry point needs a wrapper, and a large codebase can miss one. A small higher-order helper or base consumer can reduce that risk, but don't hide retry semantics behind a clever abstraction. Stick with explicit `try/catch` blocks when different jobs require different acknowledgement, retry, or cleanup behavior. This design is also not suitable when synchronous capture would violate a tight latency budget; choose a buffered adapter and define its flush behavior instead.

The finish line is operational, not visual: one event schema, three deliberate boundaries, preserved runtime semantics, a bounded delivery path, and tests that make missing coverage obvious.

## Sources

- Martin Fowler, "Feature Toggles": https://martinfowler.com/articles/feature-toggles.html
- GrowthBook, open-source feature flag and A/B experimentation platform: https://www.growthbook.io/
