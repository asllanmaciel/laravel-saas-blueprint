# Observability for multi-tenant SaaS

Observability should make it possible to answer three questions quickly: what failed, which tenant was affected, and whether the failure is isolated or systemic.

## Structured context

Attach consistent fields to logs, metrics and traces whenever available:

- environment;
- service or application name;
- tenant id;
- request id or correlation id;
- authenticated actor id when appropriate;
- route or operation name;
- job id for asynchronous work;
- external provider name for integrations.

Prefer identifiers over names or email addresses to reduce unnecessary personal data in telemetry.

## Logs

Use structured logs for events that help reconstruct a flow. Avoid turning every request into a wall of text.

Useful categories include:

- authentication and authorization failures;
- tenant resolution failures;
- billing state changes;
- webhook acceptance and processing result;
- queued job lifecycle for important workflows;
- integration failures;
- deployment and configuration events.

Never log secrets, authorization headers, full payment payloads or passwords.

## Metrics

Start with a small operational set:

- request count and latency;
- error rate;
- queue depth and oldest-job age;
- failed jobs;
- webhook processing latency and failures;
- database latency and connection pressure;
- cache hit ratio when cache is operationally important;
- external API latency and error rate.

Product-specific limits may need tenant-level metrics, but avoid labels with unbounded cardinality in systems where the metrics backend cannot support them.

## Traces

Distributed tracing is most valuable around boundaries:

```text
HTTP request
  -> database
  -> cache
  -> queue dispatch
      -> worker
          -> provider API
```

Propagate correlation context into queued jobs and outgoing requests so a single business operation can be reconstructed.

## Tenant-aware incident analysis

A multi-tenant system should distinguish:

- one tenant misconfigured;
- one tenant generating abnormal load;
- one integration provider failing;
- a region or database partition failing;
- the application failing globally.

Dashboards and alerts should help make that distinction without requiring direct database exploration during an incident.

## Alerting

Alert on symptoms that require action, not every unusual number.

Examples:

- sustained elevated error rate;
- billing webhook backlog growing beyond a safe age;
- queue oldest-job age above the user-facing SLO;
- repeated authentication or tenant-isolation failures;
- database saturation;
- critical provider outage affecting active workflows.

Use lower-severity dashboards or daily reports for trends that do not require immediate intervention.

## Service objectives

Define a few service level indicators before scale makes them hard to reconstruct. Common candidates:

- successful request rate;
- p95 or p99 latency for key interactions;
- time to process billing webhooks;
- time to deliver user-facing queued work;
- successful scheduled-job completion rate.

The goal is not formal SRE ceremony for an MVP. It is to know what 'healthy enough' means and detect when the product leaves that range.

## Retention and privacy

Telemetry has a lifecycle too. Define retention periods and remove data that no longer serves operational or compliance needs. Treat logs and traces as production data with access controls, not as harmless debug output.
