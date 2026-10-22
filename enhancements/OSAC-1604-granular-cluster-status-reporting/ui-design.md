---
title: granular-cluster-status-reporting-ui
authors:
  - Elay Aharoni
creation-date: 2026-10-13
last-updated: 2026-10-21
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-2597
prd:
  - prd.md
see-also:
  - design.md
replaces:
  - N/A
superseded-by:
  - N/A
---

# Granular Cluster Status Reporting — UI

## Experience

Tenant users keep the table/card content from [the UI alignment PR](https://github.com/osac-project/osac/pull/1876) and the same **Status** label in the list and detail header. Stalled provisioning adds a warning and popover to that label without changing its state text. A read-only detail progress card follows the [BMaaS pattern](../OSAC-3459-bmaas-provisioning-progress/ui-design.md). Details remain accessible during deletion.

Cloud Provider Admins cannot see cluster navigation or open cluster pages, including direct URLs. [User decision]

## Display rules

All values come from the public `Cluster` returned by `Clusters.List/Get`. The UI does not read HyperShift resources or decide that a stage is stalled from elapsed time.

- **Shared Status:** Keep the state text used by the aligned table, card, and detail header (**Progressing** for `PROGRESSING`). Only `status.state=PROGRESSING` with `PROGRESSING.reason=Stalled` adds a warning icon/color and keyboard-accessible popover on all three surfaces. The popover explains the stall and shows `PROGRESSING.message` or a generic fallback. This is not failure; `StageUnknown`, missing reasons, and `DELETING` do not trigger this provisioning warning. Never put stage text in the label or add list columns.
- **Provisioning progress:** For `PROGRESSING` or `FAILED`, place a full-width, read-only **Provisioning progress** card between the detail header and summary. Match the [BMaaS implementation](https://github.com/osac-project/osac/pull/1686): a centered PatternFly `ProgressStepper` with accessible text/icons. Its ordered steps are **Control plane setup**, **Workers joining**, and **Finalizing**. While `PROGRESSING`, select the current step from public conditions only:

  | Control plane | Workers | Current step |
  |---|---|---|
  | False | False, missing, or Unknown | Control plane setup |
  | True | False | Workers joining |
  | True | True | Finalizing |
  | Other combinations | Any | Current step unavailable |

  Only a confirmed, non-contradictory `True` milestone completes a step. `StageUnknown` overrides the table: select no current step. Missing/Unknown conditions needed for a decision, or Workers=True before Control plane=True, also select no current step; never reuse a stale step or complete a contradictory later step. An absent `PROGRESSING.reason`/`message` alone keeps the condition-derived step and uses generic detail text. Other reason strings are display text, not step selectors. `Stalled` warns without marking failure. For `FAILED` state or a true `FAILED` condition, use that condition's reason/message (or a fallback) for a failure summary without guessing a failed step. `READY` always hides the card, including after worker degradation.
- **Deletion detail:** Do not show a deletion progress card. On the detail **Deleting** status, an accessible popover shows the current `PROGRESSING.reason`/`message` or a fallback; `Stalled` adds a warning icon/color there without implying failure. On **Delete failed**, use the true `FAILED` condition reason/message or a fallback. The table/card keep their normal deletion labels, with no teardown stage text. Do not infer ordered teardown steps.
- **Node sets:** In the existing table, keep Name and Host type; replace Size with **Nodes** as current/requested (for example, **1/3 nodes present**) and show a **State** label on every node-set row. Map `PROVISIONING`, `READY`, `SCALING`, and `DEGRADED` to **Provisioning**, **Ready**, **Scaling**, and **Degraded**; missing or `UNSPECIFIED` state shows **Unknown**. Join `spec.node_sets` and `status.node_sets` by key: current is `current_replicas`, requested is `spec.size`. A missing current count shows **Unknown**; a reported zero shows **0**. Do not label the ratio “ready”: the backend reports whole-set readiness, not individual nodes. The public `Cluster` has no individual node inventory.

## UI changes

- The aligned `ListPage/ClusterStatusLabel` is shared by `ClusterTableRow`, `ClusterCard`, and `ClusterDetailsPageContent`; pass the public condition data needed for the stalled warning. Keep the table/card fields and actions from [PR #1876](https://github.com/osac-project/osac/pull/1876). That PR currently disables deleting-cluster detail links; restore the link in **both** views while leaving unavailable actions disabled.
- `ClusterDetailsPageContent`: keep the same state text in its header label; add the provisioning card below the header and the detail-only deletion popover. The existing Conditions table shows reported `CONTROL_PLANE_AVAILABLE`, `WORKERS_READY`, `DEGRADED`, and `FAILED` with Status, Reason, and Message, even when State stays **Ready**; add no duplicate condition summary or label.
- Node sets tab: replace Size with Nodes and show each node set's own State in its existing table.
- `ClusterDetailsPage`: show the existing not-found view when `Clusters.Get` returns `NotFound` after archival; other errors remain retryable.
- Share the `PROGRESSING` condition lookup; keep label/popover and progress-card presentation separate. Reuse `useClusters`/`useCluster` and the 10-second refresh; add no RPC, timer, or per-row request. Preserve the admin route gate. Regenerate `@osac/types` after backend API changes.

## Backend dependencies

- **Ready now:** Public `PROGRESSING.reason/message` support the shared stalled warning and detail text. Deleting-cluster detail access and the admin route gate can be verified now. A backend fix is still needed to replace a stale reason with `StageUnknown` when the HostedCluster is unavailable.
- **Wait for progress milestones:** The backend must publish and populate the control-plane and worker conditions and preserve `READY` during degradation and scaling. Until then, the card must not claim those steps are complete; the existing Conditions table shows only conditions the API reports.
- **Wait for node sets:** The backend must publish and populate `current_replicas` and per-set `state`, including `SCALING`.
- **Wait for teardown:** The backend must publish deletion stage reasons and `DELETE_FAILED` signals.

## Verification

UI unit tests cover the exact stalled predicate and accessible warning in table, card, and detail; unchanged aligned list fields/state text; condition-derived steps and explicit unavailable/failure behavior; the existing Conditions table with new condition types; node-set State/counts; deletion popover, links in both views, and archived `NotFound`; and admin direct URLs with no cluster RPCs. Run `pnpm run typecheck`, `pnpm lint`, and `pnpm test` for implementation. Browser checks require the corresponding backend phase.

---

## Provenance

Authored: draft @ design 0.11.5 - 1c6e8de, workspace main @ f51d7c726
Final: respond @ design 0.11.5 - d1824b1, workspace main @ 340adb5d7

> Context changed between draft and respond.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"d1824b1","source_repo":"340adb5d7","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":359,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","respond","respond","revise","revise","revise","revise","respond","respond"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
