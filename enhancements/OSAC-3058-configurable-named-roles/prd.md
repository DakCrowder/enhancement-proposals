# Configurable Named Roles with Separation of Duties

| Field       | Value   |
|-------------|---------|
| Author(s)   | Dakota Crowder |
| Jira        | https://redhat.atlassian.net/browse/OSAC-3058 |
| Date        | 2026-10-23 |
| Milestone   | 0.5 |

## Problem Statement

Providers can already grant users a fixed set of built-in roles — tenant-admin
and tenant-idp-manager — and every authenticated user receives a fixed
self-service allowlist, but the roles themselves are built into the service:
no administrator can create, compose, or change a role at runtime, and adding
one requires a platform redeploy. Compliance-driven deployments need to model
documented personas such as ISSO (Information System Security Officer), Auditor, and System Owner as specific sets
of full gRPC methods — none of which the fixed role set can
express.
The existing Role and RoleBinding APIs are descriptive only; they do not
influence authorization decisions. Without configurable roles, providers in
regulated industries cannot satisfy the documented-roles and
separation-of-duties expectations of NIST 800-171/HIPAA assessments.

## In Scope

- A catalog of the OSAC public API's full gRPC methods — complete
  service-qualified references such as `osac.public.v1.Clusters/List` and
  `osac.public.v1.Clusters/Create`, not bare verbs like "get" or "list" —
  where each full method is the grantable unit [Clarify: R1.Q4]
- Named roles composed as sets of catalog methods, authored by Cloud Provider
  Admins (platform-scoped) and by Tenant Admins for their own tenant
  [Clarify: R1.Q1]
- Role management at runtime through the API, the `osac` CLI, and the web
  console [Clarify: R2.Q1]

## Out of Scope

- Audit-trail requirements — role context on user actions and coverage of
  changes to roles and role assignments — out of scope while OSAC-63
  (Activity and Audit Log API) is in flight
- Fine-grained service-account privilege scoping, including deprivileging the
  template-publisher service account — tracked as OSAC-4386, which builds on
  this method-based model
- Per-request role context ("acting as") — a user's held roles apply as-is to
  every action
- Role inheritance or nesting — roles are flat sets of methods

## User Stories

### Cloud Provider Admin

- As a Cloud Provider Admin, I want to define named roles as sets of full
  gRPC methods — for example an "ISSO" role combining the cluster read/write
  methods and the security-policy read/write methods — so that I can model
  my organization's personas to what permissions they need.
- As a Cloud Provider Admin, I want to compose roles that include
  administrator-level methods when my organization needs them so that
  role composition is not limited to a predefined subset of methods.
  [Clarify: R2.Q3]
- As a Cloud Provider Admin, when I make changes to role assignments, I need
  those to propagate and apply automatically so that my users don't need
  to do anything extra. [Clarify: R1.Q5]
- As a Cloud Provider Admin, I want edits to a role to apply to all users
  that has the role so that changes take effect everywhere at once. [Clarify: R3.Q3]
- As a Cloud Provider Admin, I want deletion of a role with active
  assignments to be guarded until those assignments are removed so that
  no user loses access as a side effect of a role cleanup. [Clarify: R3.Q3]
- As a Cloud Provider Admin, I want any role assignment to be bounded by
  the permissions its assigner holds so that access is always delegated
  from real authority and can never be escalated through assignment.
  [User]
- As a Cloud Provider Admin, I want tenant role assignments to stay scoped
  within their own tenant — never crossing tenant or provider-admin method
  boundaries — so that tenant-level administration cannot grant access
  across tenants or into platform administration. [User]

### Tenant Admin

- As a Tenant Admin, I want to author tenant-scoped named roles from the
  methods I hold so that I can tailor access for my organization's users. [Clarify: R1.Q1, R3.Q1]
- As a Tenant Admin, I want to assign one or more roles to users in my tenant
  so that each user's access — the union of their held roles' methods —
  is explicit and reviewable. [Clarify: R1.Q1, R1.Q3]
- As a Tenant Admin, I want to be able to easily select from a list of
  predefined roles that represent capabilities (such as Cluster Creator or VM Viewer)
  so I can easily give my users access to what they need.
- As a Tenant Admin, I want newly onboarded users to receive a configurable
  default role automatically so that they can use self-service resources on
  day one without a manual grant. [Clarify: R3.Q4]

### Tenant User

- As a Tenant User, I want to see which roles I hold and what they permit so
  that I know what I can do without attempting actions to find out.
  [Clarify: R2.Q1]

## Assumptions

- The method catalog's first deliverable covers user-facing public API
  methods; methods governing internal service-to-service calls may be
  represented in the catalog, but are not the focus of this feature.

## Dependencies

- **Existing Role and RoleBinding data model:** the foundation this feature
  builds on; today it is descriptive only and not wired into authorization
  decisions.
- **OSAC-4386 — Service Account Privilege Scoping:** builds on this feature's
  method-based model and is blocked until it lands.
- **OSAC-3018 (HIPAA Compliance Readiness) and OSAC-2889 (EU Sovereignty &
  Compliance Readiness):** compliance outcomes that depend on this feature's
  named roles; their audit-attribution expectations are met through OSAC-63.

---

## Provenance

Authored: revise @ prd 0.11.5 - e2b0d87, workspace main @ 3e4bfdb7b
Phases: draft, revise, respond, respond, revise, manual-edit, revise, revise, manual-edit, revise, manual-edit, revise, manual-edit

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.11.5","ai_workflows":"e2b0d87","source_repo":"3e4bfdb7b","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","respond","respond","revise","manual-edit","revise","revise","manual-edit","revise","manual-edit","revise","manual-edit"],"authoring_modes":["manual","skill"],"context_changed":false,"origin_untracked":false} -->
