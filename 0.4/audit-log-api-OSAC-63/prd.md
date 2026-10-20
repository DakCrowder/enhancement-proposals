# Activity and Audit Log API

| Field       | Value   |
|-------------|---------|
| Author(s)   | Elkana Hendler |
| Jira        | [OSAC-63](https://redhat.atlassian.net/browse/OSAC-63) |
| Date        | 2026-10-20 |

## Problem Statement

OSAC does not record or expose an audit trail of platform activity. No
persona can see who performed an action, when it occurred, or what changed
on any resource. Platform operators cannot produce compliance-ready audit
reports, and tenants have no self-service access to their organization's
activity history.

Sovereign cloud deployments are subject to regulatory frameworks (GDPR,
DPDPA, NIS2, local data protection laws) that mandate auditable records of
data access and administrative actions, with guarantees about where audit
data is stored and who can access it. Without an audit log, OSAC cannot
satisfy these requirements — blocking compliance readiness efforts across
multiple regulatory domains (HIPAA, NIST SP 800-53, EU sovereignty) that
depend on this capability.

## In Scope

- Audit event capture for control-plane actions across all OSAC services
  (BMaaS, CaaS, VMaaS, MaaS, Enclave): resource mutations, API access to
  sensitive resources, failed authorization attempts, and administrative
  actions (organization lifecycle, role changes, IdP configuration, project
  permissions). [Clarify: D8]
- Keycloak authentication event capture: login success/failure, token
  operations, session events — ingested per tenant organization alongside
  fulfillment events. [Clarify: D3]
- Before/after state capture for all resource mutations. [Clarify: D4]
- Tenant-scoped and cross-tenant query via gRPC and HTTP+JSON API, with
  filtering by time range, actor, action type, resource type, resource ID,
  and outcome. [Clarify: D7, D9]
- Bulk export of audit records in JSON and CSV formats for compliance
  reporting workflows. [Clarify: R2.Q3]
- Append-only, immutable storage with cryptographic integrity verification
  — no API exposes update or delete of audit records, and readers can
  independently verify that records have not been tampered with at the
  storage layer. [Clarify: D5]
- Configurable retention policy: installation default via Helm values,
  runtime-adjustable by the Cloud Provider Admin via API, with a 30-day
  minimum floor. [Clarify: D10]
- Audit-of-audit: Cloud Provider Admin access to tenant audit data is
  itself recorded. [Clarify: D9]
- Data residency: audit data is co-located with the platform deployment,
  inheriting the deployment's sovereign jurisdiction. [Clarify: D11]
- Target milestone: 0.4 (Dev Preview) — functional but not
  production-hardened. [Clarify: D1]

## Out of Scope

- UI and CLI for audit log browsing — API-only for this milestone.
  [Clarify: D7]
- Real-time alerting on audit events (event-driven alerting and webhook
  integration are a follow-up Feature).
- Audit log forwarding to external SIEM systems.
- Data-plane audit logging (individual object reads/writes in storage,
  network packet logging) — control-plane actions only.
- Audit log analytics and dashboards.
- Specific regulatory certification (GDPR, DPDPA, NIS2) — this feature
  provides the technical controls; certification is an organizational
  process.
- Performance SLOs (query latency, ingestion lag, throughput) — deferred
  to a post-Dev-Preview milestone. Functional correctness is the bar for
  0.4. [Clarify: R3.Q1]

## User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want to query audit events across all
  tenants — filtering by time range, actor, action type, resource type,
  resource ID, and outcome — so that I can investigate security incidents
  and produce platform-wide compliance reports. [Clarify: D9]

- As a Cloud Provider Admin, I want to query audit events scoped to a
  specific tenant so that I can produce tenant-scoped audit extracts for
  regulatory authorities upon lawful request without exposing other
  tenants' data. [Clarify: D9]

- As a Cloud Provider Admin, I want to set the audit log retention period
  at installation and adjust it at runtime (subject to a 30-day minimum)
  so that I can balance storage costs against compliance requirements.
  [Clarify: D10]

### Tenant Admin

- As a Tenant Admin, I want to query a chronological audit trail of all
  actions performed within my organization — resource creation,
  modification, and deletion; API calls; authentication events; role and
  permission changes — so that I can investigate incidents and track
  changes without requesting platform operator assistance.

- As a Tenant Admin, I want each audit record to include the actor,
  timestamp, action, target resource, outcome, and before/after state so
  that I can see exactly who did what and what changed. [Clarify: D4]

- As a Tenant Admin, I want to filter audit events by time range, actor,
  action type, resource type, resource ID, and outcome (success/failure)
  so that I can find specific events efficiently.

- As a Tenant Admin, I want to export audit records in JSON and CSV
  formats so that I can submit them to regulatory authorities or feed
  them into internal compliance workflows. [Clarify: R2.Q3]

- As a Tenant Admin, I want audit records to be append-only and
  cryptographically tamper-evident so that I can trust the integrity of
  my organization's audit trail for regulatory submissions. [Clarify: D5]

- As a Tenant Admin, I want authentication events (login success/failure,
  token issuance, session events) from my organization's users to appear
  in the audit trail so that I have complete visibility into identity
  activity. [Clarify: D3]

## Assumptions

- Keycloak is deployed as part of the OSAC platform and can be configured
  to emit authentication events to the audit log's ingestion pipeline.
- All OSAC services that manage tenant resources emit events to the
  existing Kafka-based event distribution infrastructure (OSAC-983). If a
  service does not yet emit events for all its resource types, adding
  event emission is part of this feature's scope.

## Dependencies

- **OSAC-6154 — Actor attribution in Fulfillment Events (blocking):**
  Each audit record must identify the user who initiated the action.
  OSAC-6154 adds actor identity to the event contract. The audit log
  cannot ship without it. [Clarify: D2]

- **OSAC-983 — Reliable Event Distribution via Message Broker
  (satisfied):** The audit log consumes durable, tenant-isolated events
  from Kafka. OSAC-983 is Closed — the message broker infrastructure is
  in place.

- **Keycloak event integration:** Authentication events (login, token,
  session) originate from Keycloak, which is a separate event source from
  the Fulfillment Events pipeline. Integration requires a Keycloak event
  listener or SPI configuration. [Clarify: D3]

---

## Provenance

Authored: draft @ prd 0.11.5 - 4164796, workspace main @ f4607b916

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.5","ai_workflows":"4164796","source_repo":"f4607b916","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
