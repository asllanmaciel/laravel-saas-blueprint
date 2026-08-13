# Backup and tenant restore drills

Backup strategy is useful only when restoration is defined, tested and measured. In a multi-tenant SaaS, the harder question is often not "can we restore the database?" but "can we recover one tenant without corrupting or rolling back everyone else?"

This playbook is vendor-independent and focuses on operational decisions that should exist before an incident.

## 1. Define recovery objectives

Record explicit targets for each data class:

| Data class | Example | RPO | RTO | Restore scope |
|---|---|---:|---:|---|
| Primary relational data | accounts, subscriptions, domain records | product-specific | product-specific | tenant or full system |
| Object/file storage | uploads, exports, documents | product-specific | product-specific | tenant or object set |
| Audit/security records | access logs, security events | product-specific | product-specific | usually append-only |
| Cache/search projections | Redis, search indexes | rebuildable | rebuild time | rebuild rather than restore |
| Secrets and keys | integration credentials, encryption keys | near-zero loss | incident-specific | controlled recovery |

RPO and RTO must be promises the current backup architecture can actually meet. A daily backup cannot support a one-hour RPO.

## 2. Inventory what a complete restore means

A database snapshot alone may not restore a usable tenant. Map dependencies explicitly:

- relational records and tenant ownership keys;
- object/file storage paths;
- externally stored documents or media;
- billing provider identifiers and entitlement state;
- queued jobs and webhook deduplication state;
- audit records required for investigation;
- encryption keys and application secrets;
- derived indexes, caches and materialized projections.

Classify every dependency as **restore**, **rebuild**, **reconcile**, or **do not restore**.

## 3. Shared database: tenant-level restore

A shared-schema database makes full-system restore straightforward but tenant-only recovery harder.

A safe tenant restore should:

1. restore the backup into an isolated temporary database;
2. select records only through authoritative tenant ownership relationships;
3. validate referential closure before copying anything back;
4. put the affected tenant into a maintenance/read-only state when required;
5. export the tenant dataset from the temporary restore;
6. reconcile IDs and globally unique records explicitly;
7. import inside controlled transactions where feasible;
8. rebuild derived caches/search projections;
9. verify counts, checksums and critical domain invariants;
10. reopen the tenant only after validation.

Do not restore tenant data with ad-hoc `WHERE tenant_id = ?` scripts unless every related table and ownership path has been audited. Some records may belong through joins rather than a direct tenant column.

## 4. Database-per-tenant restore

Database-per-tenant architecture makes tenant recovery more isolated, but control-plane data still matters.

A restore drill should verify:

- the tenant database can be restored independently;
- control-plane metadata still points to the restored database;
- schema version matches the application version;
- encryption/key material is available;
- storage objects correspond to the restored point in time;
- background workers will not replay obsolete work after recovery.

If a tenant database is restored to an earlier point than shared billing/control-plane state, define which system is authoritative and how reconciliation happens.

## 5. File and object storage

Tenant recovery frequently fails because database rows are restored but files are not.

Design storage so tenant scope can be identified safely, for example with tenant-qualified prefixes or metadata. A drill should prove that you can:

- enumerate one tenant's objects;
- recover deleted/overwritten objects from versioning or backup;
- verify object integrity;
- avoid overwriting unrelated tenants;
- reconcile database references with restored objects.

Signed URLs and CDN caches are derived delivery mechanisms; regenerate or invalidate them instead of treating them as backup data.

## 6. Queues, webhooks and idempotency after restore

Restoring persistent data can make old asynchronous work dangerous.

Before workers resume, decide what to do with:

- jobs created after the restore point;
- webhook events already acknowledged externally;
- idempotency/deduplication keys;
- scheduled billing actions;
- outbound notifications and exports.

A restore must not accidentally resend invoices, duplicate charges, recreate already completed exports or replay destructive jobs.

Prefer reconciliation against authoritative external systems rather than blindly replaying historical queues.

## 7. Secrets and encryption keys

Backups that cannot be decrypted are not recoverable.

Maintain a controlled recovery procedure for:

- application encryption keys;
- tenant-specific keys when used;
- backup encryption keys;
- credentials needed to reach backup/storage systems.

Do not store backup-decryption keys only inside the same environment or account whose loss the backup is meant to protect.

Key recovery should be tested with the same access controls and audit requirements expected during a real incident.

## 8. Restore drill checklist

Run a restore exercise periodically and preserve evidence.

### Preparation

- [ ] choose a disposable environment isolated from production;
- [ ] record the selected backup timestamp;
- [ ] record expected RPO/RTO targets;
- [ ] choose one representative tenant and expected record counts;
- [ ] identify required database, storage and key material.

### Restore

- [ ] restore database data into the isolated environment;
- [ ] restore or reconcile tenant files/objects;
- [ ] load the correct application/schema version;
- [ ] recover required keys/secrets through the documented process;
- [ ] keep outbound email, payments and external side effects disabled.

### Validation

- [ ] tenant can authenticate/access expected resources;
- [ ] record counts match expected ranges;
- [ ] critical foreign-key/domain relationships are intact;
- [ ] another tenant's records are absent from the tenant-level restore;
- [ ] files referenced by restored records are available;
- [ ] billing identifiers/entitlements reconcile with the external provider;
- [ ] caches/search indexes rebuild successfully;
- [ ] no stale queue/job is replayed unexpectedly.

### Evidence

Record at minimum:

```text
backup timestamp
restore start/end time
measured RPO
measured RTO
scope restored
validation queries/checksums
failed checks
manual interventions
follow-up actions
owner and date
```

A restore drill that only says "backup restored successfully" is not enough evidence for a production recovery capability.

## 9. Partial-restore risks

Common failure modes include:

- restoring tenant rows without associated storage objects;
- restoring database state while leaving newer queue jobs active;
- restoring subscription state that disagrees with the billing provider;
- reusing IDs that now belong to newer records;
- restoring encrypted data without the matching key version;
- recovering one table but not dependent join/pivot tables;
- restoring a tenant while users continue writing to the live dataset;
- treating caches/search indexes as authoritative data.

Document which operations require a tenant maintenance window and which can be reconciled online.

## 10. Incident communication

Recovery is also an operational and communication process.

Define in advance:

- who can authorize a restore;
- who owns database/storage execution;
- who validates tenant isolation and business invariants;
- when security/legal/privacy stakeholders must be involved;
- what evidence is preserved;
- what and when affected customers are told.

Avoid estimating data loss from backup timestamps alone. Confirm the actual recovered state before communicating an RPO outcome.

## 11. Minimum production baseline

Before calling backups "production ready," be able to answer yes to these questions:

- Can we restore outside the primary production environment?
- Have we restored using the real encrypted backups, not a synthetic copy?
- Can we recover one tenant without rolling back unrelated tenants?
- Can we restore tenant files as well as database rows?
- Can we recover required encryption keys under incident conditions?
- Can we reconcile external billing state after a restore?
- Can we prevent stale jobs/webhooks from replaying destructive side effects?
- Do we have measured evidence for the RPO and RTO we claim?

If any answer is unknown, the restore capability is not yet proven.
