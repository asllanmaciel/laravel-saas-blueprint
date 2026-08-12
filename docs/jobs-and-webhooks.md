# Jobs, webhooks and idempotency

Background work becomes part of the consistency model of a SaaS. A queue is not only a performance tool: retries, duplicate delivery and tenant context must be designed explicitly.

## Carry tenant context explicitly

A queued job should contain enough immutable identifiers to restore its execution context safely.

Avoid relying on ambient session state, the current host or a globally mutable tenant singleton that may survive between worker jobs.

A useful envelope is conceptually:

```text
job type
job id
 tenant id
actor id (when relevant)
resource id
correlation id
attempt metadata
```

Resolve the tenant again when the worker starts and fail closed if the referenced tenant or resource no longer belongs to that context.

## Design jobs for retry

Assume any job may execute more than once.

Prefer operations that are naturally idempotent. When that is not possible, persist an idempotency key or a processed-event record before performing an irreversible side effect.

Common examples:

- payment provider event id;
- import batch id plus row id;
- notification purpose plus recipient plus business event id;
- export request id;
- external API command id.

## Keep database state and side effects separate

A robust pattern is:

1. validate tenant and resource ownership;
2. perform the local state transition;
3. commit the transaction;
4. dispatch or execute external side effects;
5. record outcome and correlation identifiers.

For workflows where losing the post-commit side effect would be unacceptable, use an outbox-style approach so the intent is stored atomically with the state transition and delivered asynchronously.

## Webhook inbox

Incoming webhooks benefit from the mirror image of an outbox: an inbox.

Store a normalized record containing:

- provider;
- external event id;
- received timestamp;
- signature validation result;
- event type;
- tenant mapping when known;
- processing status;
- attempt count;
- last error summary.

Use a unique constraint on provider plus external event id to make duplicate delivery cheap to detect.

## Queue separation

Do not put every workload in one undifferentiated queue. Separate classes of work when they have different latency and failure characteristics, for example:

- user-facing short jobs;
- webhook processing;
- email and notifications;
- imports and exports;
- slow third-party synchronization;
- scheduled maintenance.

This prevents a large import from delaying password emails or billing webhooks.

## Retry policy

Retry only errors that may succeed later. Permanent validation failures should fail quickly and visibly.

Use bounded retries with backoff and jitter for remote dependencies. Record enough context to investigate the final failure without logging secrets or complete sensitive payloads.

## Dead-letter handling

A failed-job table is not an operational strategy by itself. Define:

- who reviews failures;
- which failures page someone immediately;
- which can wait for a daily queue review;
- how a job is safely replayed;
- how replay avoids repeating side effects.

## Observability

Every important async flow should be traceable from the originating request or webhook to the job and any external call. Carry a correlation id and include tenant id, job id and resource identifiers as structured log fields.

Do not put access tokens, full payment payloads, passwords or sensitive personal data in job names or logs.
