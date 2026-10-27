# Add actor attribution to Fulfillment Events

| Field     | Value |
|-----------|-------|
| Author(s) | Juan Hernandez |
| Jira      | [OSAC-6154 — Feature](https://redhat.atlassian.net/browse/OSAC-6154); tracked by [OSAC-6855 — PRD task](https://redhat.atlassian.net/browse/OSAC-6855) |
| Date      | 2026-10-26 |

## Problem Statement

Fulfillment object-change events are available to the Cloud Provider Admin,
but they do not identify who initiated a change. The Admin cannot reliably
attribute object activity to a person or to the system, and internal billing
and metering services cannot use these events to attribute activity. This
Feature adds actor identity to object-change events.

## In Scope

- Actor attribution covers all object changes and is available through both
  the private API and Kafka. [Clarify: R1.Q2, R2.Q2]
- Events for user-initiated object changes identify the authenticated user's
  name; events for system-initiated changes identify the system. [Jira: OSAC-6154]
  [Clarify: R1.Q3, R2.Q3]
- If an actor's identity is genuinely unavailable, the event remains valid
  without an actor identity. [Jira: OSAC-6154]
- The Cloud Provider Admin can use attributed events, and internal billing and
  metering services can consume them. [Clarify: R1.Q1, R2.Q1]

## Out of Scope

- Attribution for events that do not describe object changes, such as reads,
  authentication, or administrative actions. [Clarify: R1.Q2]
- Additional actor metadata such as source IP address, user agent, and region.
  [Clarify: R1.Q4]

## User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want object-change events to identify the
  user or system that initiated each change, so that I can attribute object
  activity when consuming or reviewing events. [Clarify: R1.Q1, R2.Q1, R2.Q3]

## Dependencies

- **Internal billing and metering services:** These services consume
  actor-attributed object events; they are downstream consumers, not OSAC
  personas. [Clarify: R1.Q1, R2.Q1]
- **Activity and Audit Log API (OSAC-63):** A related consumer of actor
  attribution; audit logging is not the sole scope of this Feature. [Jira:
  OSAC-6154]
- **Metering event contract (OSAC-6056):** A related downstream consumer of
  Fulfillment event data. [Jira: OSAC-6154]
- **Design (OSAC-6856):** Define the exact event representation for the system
  actor before implementation. [Clarify: R2.Q3]

---

## Provenance

Authored: revise @ prd 0.11.5 - 7b2b8de, workspace feat/osac-6637-reader-role-access-docs @ 700534eab
Phases: draft, revise

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.5","ai_workflows":"7b2b8de","source_repo":"700534eab","source_repo_branch":"feat/osac-6637-reader-role-access-docs","commits_behind_main":0,"commits_ahead_main":1,"main_ref":"main","phases":["draft","revise"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
