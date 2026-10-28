# Clarification Log — OSAC-3058

## Status

- Rounds completed: 3
- Open gaps: 0
- Exit criteria met: Yes

## Round 1 — Personas, SoD semantics, and capability model

### R1.Q1: Who can author and manage custom named roles

The ticket says Cloud Provider Admins define named roles ("provider-defined
compositions"), but OSAC already has tenant-scoped seeded roles
(tenant-admin, tenant-user), raising the question of tenant-level
customization. For this feature's first deliverable, who should be able to
create and manage custom named roles?

#### Answer

Both: Cloud Provider Admins author platform-scoped named roles AND Tenant
Admins can author tenant-scoped custom roles usable only within their own
tenant.

#### Impact

The PRD includes two authoring surfaces: platform-level role management
(Cloud Provider Admin) and tenant-scoped role management (Tenant Admin).
Tenant-authored roles are usable and visible only within the authoring
tenant. This expands scope beyond the ticket's literal text and needs
explicit user stories for both personas.

#### Decision (D1)

Tenant Admins can author tenant-scoped custom named roles in addition to
Cloud Provider Admin platform-scoped named roles. Tenant-scoped roles cannot
leak outside their tenant.

---

### R1.Q2: Treatment of the "Compliance Admin" actor

One user story says "As a Compliance Admin, I need the audit trail to show
the named role that authorized each action". OSAC's canonical personas are
Cloud Provider Admin, Cloud Infrastructure Admin, Tenant Admin, and Tenant
User — there is no Compliance Admin persona. How should the PRD treat this
actor?

#### Answer

Compliance-viewing is a Cloud Provider Admin responsibility (a "hat"), not a
new persona. The PRD uses canonical persona names.

#### Impact

The Compliance Admin user story is rewritten for the Cloud Provider Admin
persona. No new persona definition is introduced.

#### Decision (D2)

"Compliance Admin" is a Cloud Provider Admin responsibility; the PRD uses
canonical persona names only.

---

### R1.Q3: Separation-of-duties semantics

The ticket says certain roles "are mutually exclusive on the same user" and
also that "each user has one compliance role, which is their identity for
audit purposes". These can be read as (a) users may hold multiple roles
blocked only by configured mutual exclusions, or (b) each user holds exactly
one named role at a time.

#### Answer

Users may hold multiple named roles simultaneously; only configured
mutually-exclusive pairs are blocked at assignment time. "One compliance
role" applies only to compliance-sensitive roles.

#### Impact

The PRD describes assignment-time validation for mutual-exclusion
constraints rather than single-role cardinality. Audit role context needs a
rule for which role is recorded when a user holds several (likely: the role
whose capability authorized the action).

#### Decision (D3)

Users can hold multiple named roles; separation of duties is enforced via
configurable mutual-exclusion constraints checked at assignment time.

*Superseded in part during revision (2026-10-27): users may still hold
multiple named roles, but mutual-exclusion constraints and assignment-time
denial were removed from the PRD at the author's instruction (see D12).*

---

### R1.Q4: Capability atom granularity

The Definition of Done says "granular capabilities defined for all OSAC
resource types (read, write per resource)", but the design-note examples
imply finer distinctions (audit-log read-only, secrets read-only as separate
capabilities). What should the capability atoms be?

#### Answer

Capabilities are verb × resource pairs (get/list/create/update/delete/watch
per resource type), matching the API's method structure.

#### Impact

The PRD's capability catalog is organized per resource type with standard
verbs, letting providers compose precise roles (e.g., "list and get clusters,
never create"). The catalog must enumerate resource types and verbs as a
user-visible listing requirement.

#### Decision (D4)

Capability atoms are verb × resource-type pairs covering the standard method
set per resource type.

*Superseded during revision (2026-10-27): per reviewer feedback, role atoms
are the public API's full gRPC methods — complete service-qualified
references such as `osac.public.v1.Clusters/List` — not verb × resource-type
pairs or bare verbs. A role is a set of full gRPC methods.*

---

### R1.Q5: Assignment effect latency

When an admin grants or revokes a named role, how quickly must it affect the
user's actual access? Today, assignments synced to Keycloak only take effect
when the user's token refreshes.

#### Answer

Grants and revocations must propagate without requiring re-login, token
refresh, or redeployment; no next-request guarantee is made, leaving room
for eventual consistency.

#### Impact

The PRD states propagation without re-login or redeployment as a
user-observable requirement. This rules out token-claim-only enforcement
as sufficient and is a non-functional requirement on the authorization
path.

#### Decision (D5)

Role grant/revocation takes effect on the user's next request (no token
refresh or re-login required).

*Revised during review response (2026-10-28): the next-request guarantee
was removed per reviewer feedback to leave room for eventual consistency.
Grants and revocations must still propagate without re-login or
redeployment.*

---

## Round 2 — Scope boundaries, dependencies, and interfaces

### R2.Q1: Which interfaces ship with the first deliverable

The ticket's user stories mention managing roles "via API" but don't say
which client surfaces ship first. OSAC convention allows API/CLI-only
increments with UI deferred.

#### Answer

API, CLI, and the osac-ui console all ship in the same deliverable.

#### Impact

The PRD includes user stories for role/assignment management and
role-visibility workflows across all three surfaces, per affected persona.
UI work follows the Jira component conventions for affected personas.

#### Decision (D6)

First deliverable includes API, `osac` CLI, and osac-ui console support.

---

### R2.Q2: Existing access behavior at upgrade

Today's permissions come from hardcoded realm roles (tenant-admin,
tenant-idp-manager, admins group, admin service accounts) plus the fixed
"client" allowlist for ordinary users. What must happen to existing access
when this feature lands?

#### Answer

Hard cutover at upgrade: a migration switches every user to explicit
named-role assignments at upgrade time, and realm-role grants stop being
honored.

#### Impact

The PRD requires a migration that computes equivalent named-role
assignments from existing grants before enforcement switches over, and
states that legacy realm-role permissions no longer authorize anything
after cutover. Note: OSAC does not currently support in-place upgrades, so
this is stated as data-migration/backward-compatibility behavior rather
than a live upgrade procedure.

#### Decision (D7)

Hard cutover: upgrade migrates all users to explicit named-role
assignments; legacy realm-role grants stop being honored.

*Superseded during draft revision (2026-10-23): there are no production
deployments, so no migration of existing grants is required. Migration
requirements were removed from the PRD at the author's instruction. D14's
default-role auto-assignment at onboarding remains; deny-all remains the
fallback state for any user without an assignment.*

---

### R2.Q3: Can custom roles include administrator-level capabilities

This is the privilege-escalation boundary for role composition. Options were
catalog-bounded composition (admin capabilities reserved to built-in
protected roles) versus fully composable admin capabilities.

#### Answer

Cloud Provider Admins may compose roles that include administrator-level
capabilities and grant them freely (full flexibility, audited).

#### Impact

The PRD does not restrict provider-composed roles to a sub-admin catalog;
admin capabilities are composable. All such grants are audited. The ceiling
for tenant-authored roles (D1) remains to be resolved — see Round 3.

#### Decision (D8)

Platform-admin capabilities are composable into custom roles by Cloud
Provider Admins; grants are audited.

*Superseded in part during revision (2026-10-27): composability is
unchanged, but the audited-grants requirement is deferred with the other
audit-trail requirements while OSAC-63 is in flight (see D9).*

---

### R2.Q4: Sequencing against the audit log dependency (OSAC-63)

The ticket depends on OSAC-63 (Activity and Audit Log API, In Progress) for
"audit log records the named role for each action". What is the deliverable
if OSAC-63 is not ready?

#### Answer

This feature delivers role context on the existing events/activity surface
and defines the integration contract; full audit-log integration lands when
OSAC-63 ships. OSAC-63 is listed as a dependency with a defined integration
point.

#### Impact

The audit-trail user story is split: named-role attribution is delivered on
the available activity surface now, with the OSAC-63 integration as a
named dependency rather than a blocker.

#### Decision (D9)

Deliver the role-context contract on existing surfaces; full audit-log
integration completes when OSAC-63 lands.

*Superseded during revision (2026-10-27): audit-trail requirements — role
context on user actions and audit coverage of role/assignment changes — are
out of scope for this feature while OSAC-63 is in flight; they land with
OSAC-63.*

---

### R2.Q5: Is the policy-engine replacement itself in scope

The reviewer comment recommends a complete Go rewrite of the Rego
authorization; the DoD says "Rego authorization evaluates role-based
capabilities instead of hardcoded role checks". Should the PRD scope the
engine replacement, or only the outcome?

#### Answer

The PRD requires the outcome — authorization decisions derive from the
database-backed role model — but engine mechanics (Go vs Rego) stay in the
design doc. The reviewer's rewrite becomes a design recommendation, not a
PRD requirement.

#### Impact

The PRD expresses the requirement as data-driven authorization at runtime
(no redeploy to change roles) without naming the policy engine. Design
decides the mechanism.

#### Decision (D10)

PRD scopes the outcome (runtime-configurable, database-backed
authorization), not the engine; mechanism is a design-doc concern.

---

## Round 3 — Tenant ceilings, lifecycle, onboarding, and audit semantics

### R3.Q1: Ceiling on tenant-authored roles

D1 lets Tenant Admins author tenant-scoped roles, and D8 lets
provider-composed roles include admin capabilities. What can a TENANT-authored
role contain? (Tenant isolation is a hard OSAC invariant.)

#### Answer

Tenant-authored roles draw only from capabilities the Tenant Admin themselves
holds, and only over resources within their own tenant. They cannot compose
platform-admin or cross-tenant capabilities.

#### Impact

The PRD states tenant-role composition is bounded by the authoring Tenant
Admin's own capabilities and tenant scope. This preserves tenant isolation
while D8 keeps provider-side composition unrestricted.

#### Decision (D11)

Tenant-authored roles are bounded by the authoring Tenant Admin's own
capabilities and their tenant's scope; no platform-admin or cross-tenant
capabilities.

*Extended during review response (2026-10-26): the ceiling also governs
assignment — Tenant Admins may assign provider-defined roles only when
those roles contain no platform-admin or cross-tenant capabilities.*

---

### R3.Q2: Who configures mutual-exclusion constraints

The ticket says exclusions are "configurable" but not by whom. Exclusions
involving platform roles look provider-defined; tenant-scoped roles might
warrant tenant-level rules.

#### Answer

Both levels: Cloud Provider Admins manage platform-wide exclusions AND
Tenant Admins can declare exclusions among their tenant-scoped roles.

#### Impact

The PRD includes two configuration surfaces for separation-of-duties
constraints with user stories for each persona.

#### Decision (D12)

Mutual-exclusion constraints are configurable at platform level (CPA) and
tenant level (Tenant Admin, tenant-scoped roles only).

*Superseded during revision (2026-10-27): mutual-exclusion constraints were
removed from the PRD at the author's instruction; roles may be freely
combined with no conflict rules declared or enforced.*

---

### R3.Q3: Role lifecycle — edit in use, delete with assignments

What happens when an admin edits a named role that users currently hold, or
deletes a role with active assignments?

#### Answer

Edits to a role's capability set apply to all holders without re-login or
redeployment (per D5). Deletion of a role with active assignments is
blocked until assignments are removed (or auto-revoked with confirmation).

#### Impact

The PRD's role lifecycle includes live edits that propagate to all holders
and guarded deletion. Combined with D5, the lifecycle never requires
re-login or redeployment.

#### Decision (D13)

Live edit with immediate effect on holders; deletion is guarded until
assignments are removed or explicitly revoked.

*Revised during review response (2026-10-28): "immediate effect" is
restated as propagation to all holders without re-login or redeployment,
matching the revised D5; no next-request guarantee is made.*

---

### R3.Q4: First role for newly onboarded users (deny-all interaction)

Context gathered during clarification: today's onboarding is frictionless by
design — users auto-join the tenant's Keycloak organization at IdP login,
and the JIT provisioning interceptor (which runs after authn and authz)
creates their OSAC User row on their first authorized API call. A brand-new
user immediately receives today's client allowlist (self-service CRUD within
their tenant). Granting roles requires the provisioned User row's UUID —
creating a chicken-and-egg problem under literal deny-all (an unassigned user
cannot make an authorized call, so is never provisioned, so can never be
granted their first role).

#### Answer

The tenant onboarding/user-creation flow assigns a seeded default role
(equivalent to today's client allowlist) automatically.

#### Impact

The PRD preserves frictionless onboarding (new users are productive
immediately with baseline self-service) and avoids the JIT deadlock.

#### Decision (D14)

Newly onboarded users automatically receive a seeded default role equivalent
to today's client allowlist.

*Extended during review response (2026-10-26): auto-assignment at
onboarding and the seeded platform default are unchanged, but Tenant
Admins may now replace the default role for their tenant with a
tenant-scoped role (bounded by D11); a tenant without an override — or
one that removes it — falls back to the platform default, never
deny-all.*

---

### R3.Q5: Audit role-context form when a user holds multiple roles

When a user holding multiple named roles performs an action, what does the
audit trail record? (User asked whether a union of the allowlist could be
shown; compliance nuance: auditors check for named, documented roles, and a
raw capability union maps to no documented role.)

#### Answer

Each audit event records the named roles the user held at action time (the
union of roles) AND the specific role whose capability authorized the action.

#### Impact

The audit contract includes both the held-roles list and the authorizing-role
attribution, satisfying the NIST named-role requirement while preserving
multi-role context.

#### Decision (D15)

Audit events record the union of named roles held plus the specific
authorizing role for the action.

*Extended during review response (2026-10-26): when several held roles
grant the capability that authorized an action, attribution must be
deterministic — a single defined authorizing role or all matching roles
recorded — so audit attribution is stable; the selection mechanism is a
design concern.*

*Superseded during revision (2026-10-27): audit role-context requirements
are out of scope for this feature while OSAC-63 is in flight (see D9).*

---

## Remaining Gaps

None — all identified gaps are resolved. Items to carry into the PRD as
assumptions or risks:

- D8 (admin capabilities composable) heightens the importance of audited
  grants and the D11 tenant ceiling; the PRD should call out both.
- The JIT provisioning flow depends on some method remaining callable by
  newly onboarded users (D14 default role satisfies this).

## Locked Decisions Summary

- **D1** — Tenant Admins can author tenant-scoped custom named roles
- **D2** — "Compliance Admin" is a Cloud Provider Admin responsibility; canonical personas only
- **D3** — Users can hold multiple roles; ~~SoD via configurable mutual
  exclusions~~ mutual-exclusion enforcement removed during revision
  (2026-10-27)
- **D4** — ~~Capability atoms are verb × resource-type pairs~~ Superseded
  during revision (2026-10-27): role atoms are the public API's full gRPC
  methods
- **D5** — ~~Grant/revocation takes effect on the next request~~ Revised
  during review response (2026-10-28): grant/revocation propagates without
  re-login or redeployment; eventual consistency acceptable
- **D6** — First deliverable includes API, CLI, and UI
- **D7** — ~~Hard cutover at upgrade~~ Superseded during draft revision: no
  production deployments exist, so no grant migration is required
- **D8** — Platform-admin capabilities composable by CPA; ~~grants audited~~
  audit of grants deferred to OSAC-63 during revision (2026-10-27)
- **D9** — ~~Deliver role-context contract now; full OSAC-63 audit integration
  later~~ Superseded during revision (2026-10-27): audit-trail requirements
  out of scope while OSAC-63 is in flight
- **D10** — PRD scopes the authorization outcome, not the policy engine
- **D11** — Tenant-authored roles bounded by authoring Tenant Admin's capabilities and tenant scope
- **D12** — ~~Mutual exclusions configurable at platform and tenant level~~
  Superseded during revision (2026-10-27): removed from the PRD at the
  author's instruction
- **D13** — Live role edits; guarded deletion. Revised during review
  response (2026-10-28): edits propagate to all holders without re-login
  or redeployment (no next-request guarantee)
- **D14** — Auto-assign seeded default role at onboarding; deny-all remains
  the fallback for a user with no assignment. Extended during review
  response: Tenant Admins may replace the default role per tenant with a
  tenant-scoped role; a tenant without an override receives the platform
  default
- **D15** — ~~Audit records roles held plus the authorizing role~~ Superseded
  during revision (2026-10-27): out of scope while OSAC-63 is in flight
