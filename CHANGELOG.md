# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project follows semantic versioning for published releases.

## [Unreleased]

### Added

- Security policy for responsible vulnerability reporting.
- Changelog to track meaningful documentation and architecture updates.
- Billing adapter and entitlement architecture guidance.
- Jobs, webhooks and idempotency reliability guidance.
- Tenant-aware observability guidance.
- Vendor-independent backup and restore playbook with restore-drill guidance for multi-tenant systems.
- Practical README usage path connecting isolation, security, billing, jobs, observability, restore and MVP decisions.
- Practical tenant-isolation testing playbook covering cross-tenant CRUD, route binding, fail-closed context, jobs, cache, storage and privileged impersonation paths.

### Changed

- README now makes explicit that the repository is an architectural decision blueprint rather than an executable Laravel starter kit.
- README usage path now links isolation design to an explicit testing gate before downstream SaaS architecture decisions.

## [0.1.0] - 2026-08-07

### Added

- Initial public Laravel SaaS architecture blueprint.
- Guidance covering tenancy, security, billing and operations.
- Contributor guidance, code of conduct and repository documentation structure.
