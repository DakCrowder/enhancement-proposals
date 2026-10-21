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

Tenant users keep the existing cluster list columns and **Status** label. Only stalled provisioning changes that label to a warning with a popover. The detail header keeps its existing state-only label; a separate read-only progress card follows the [BMaaS detail-page pattern](../OSAC-3459-bmaas-provisioning-progress/ui-design.md). Health appears on the detail page, alongside node sets, conditions, and endpoint URLs. Details remain accessible during deletion.

Cloud Provider Admins cannot see cluster navigation or open cluster pages, including direct URLs. [User decision]

## Display rules

All values come from the public `Cluster` returned by `Clusters.List/Get`. The UI does not read HyperShift resources or decide that a stage is stalled from elapsed time.

- **List Status:** Keep the current column, labels, and other columns, including **Version** and **Lifecycle**. Only when `status.state=PROGRESSING` **and** the `PROGRESSING` condition has `reason=Stalled`, show **Provisioning** in a warning-color label with a warning icon. A keyboard-accessible popover explains that progress has stalled and shows the condition's `message` (which names the stuck stage), or a generic fallback if absent. This is not a failure. All other rows keep their current label and have no stage text or popover; `StageUnknown`, missing reasons, and `DELETING` do not trigger this list warning.
- **Detail progress:** Keep the header badge state-only. For `PROGRESSING` and `FAILED`, place a full-width **Provisioning progress** card between the header and detail content. Match the [merged BMaaS implementation](https://github.com/osac-project/osac/pull/1686)'s layout: a centered, non-interactive PatternFly `ProgressStepper` with accessible step labels/descriptions and an in-progress icon. Show **Control plane setup**, **Workers joining**, and **Finalizing** in order. `CONTROL_PLANE_AVAILABLE=True` completes the first step and `WORKERS_READY=True` completes the second; `status.state=READY` ends provisioning and hides the card, including when worker health later degrades. While progressing, the first unmet step is current, earlier steps are complete, and later steps are not started. Show the API's `PROGRESSING.reason` and `message` on the current step; only `Stalled` and `StageUnknown` are machine signals, and other sub-stage reasons never select a step. `Stalled` warns without marking failure. Missing or inconsistent signals leave progress unavailable. While the card is visible, `FAILED` state or a true `FAILED` condition shows failure without guessing an unreported step. Icons and text accompany each step state so color is not the only cue.
- **Deletion progress:** For `DELETING` and `DELETE_FAILED`, use a separate **Deletion progress** card in the same placement. During `DELETING`, show the current teardown reason/message, with a warning for `Stalled`, and keep the list Status label **Deleting**. Keep the card visible on `DELETE_FAILED` and show its `FAILED` condition reason/message. Do not infer ordered teardown steps from opaque reasons.
- **Health:** On details, outside the header badge, use `CONTROL_PLANE_AVAILABLE` and `WORKERS_READY` for separate Control plane and Workers indicators. Their true/false labels are **Available/Unavailable** and **Ready/Not ready**. Missing or unknown conditions show **Unknown**. Keep true `DEGRADED` or `FAILED` conditions visible in the existing Conditions table; never infer health from lifecycle or replica counts. Do not add a Health list column.
- **Node sets:** In the existing table, keep Name and Host type; replace Size with **Nodes** as current/requested (for example, **1/3 nodes present**) and show a **State** label on every node-set row. Map `PROVISIONING`, `READY`, `SCALING`, and `DEGRADED` to **Provisioning**, **Ready**, **Scaling**, and **Degraded**; missing or `UNSPECIFIED` state shows **Unknown**. Join `spec.node_sets` and `status.node_sets` by key: current is `current_replicas`, requested is `spec.size`. A missing current count shows **Unknown**; a reported zero shows **0**. Do not label the ratio “ready”: the backend reports whole-set readiness, not individual nodes. The public `Cluster` has no individual node inventory.

## UI changes

- `ClustersTable`: keep its columns and state labels; add only the stalled-provisioning warning treatment and popover. The previously agreed deleting-cluster detail link remains available while actions stay disabled.
- `ClusterDetailsPageContent`: leave `ClusterStatusLabel` in the header state-only. Place the applicable read-only progress card below it and above detail content, with a separate Health summary in the detail content. Keep the Conditions table's existing Reason and Message columns, with readable condition types; add no conditions-table popover or column.
- Node sets tab: replace Size with Nodes and show each node set's own State in its existing table.
- `ClusterDetailsPage`: show the existing not-found view when `Clusters.Get` returns `NotFound` after archival; other errors remain retryable.
- Share the `PROGRESSING` condition lookup, but keep list warning and detail-card presentation separate. Reuse the existing `useClusters`/`useCluster` hooks and 10-second refresh; add no RPC, timer, or per-row request. Preserve the admin route gate. Use generated enum names and regenerate `@osac/types` after the backend API changes.

## Backend dependencies

- **Ready now:** The public `PROGRESSING` reason/message support the list's `Stalled` warning and the detail card's current-stage text. A backend fix is still needed to replace a stale reason with `StageUnknown` when the HostedCluster is unavailable. Deleting-cluster detail access and the admin route gate can be verified now.
- **Wait for health and progress milestones:** The backend must publish and populate the control-plane and worker conditions and preserve `READY` during degradation and scaling. Until then, the card must not claim those steps are complete.
- **Wait for node sets:** The backend must publish and populate `current_replicas` and per-set `state`, including `SCALING`.
- **Wait for teardown:** The backend must publish deletion stage reasons and `DELETE_FAILED` signals.

## Verification

UI unit tests cover the list's exact `PROGRESSING` + `Stalled` predicate, warning icon/color and accessible popover with message/fallback; unchanged columns and labels otherwise; the state-only detail badge; progress-card placement, accessible steps, visibility by State, condition-derived steps, unknown/stalled/failure behavior, and current reason/message; separate Health; node-set State mapping and missing/zero counts; deletion and `NotFound`; and admin direct URLs with no cluster RPCs. Run `pnpm run typecheck`, `pnpm lint`, and `pnpm test` for implementation. Browser E2E checks require the corresponding backend phase and an owned UI E2E harness.

---

## Provenance

Authored: draft @ design 0.11.5 - 1c6e8de, workspace main @ f51d7c726
Final: revise @ design 0.11.5 - d1824b1, workspace main @ 340adb5d7

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.5","ai_workflows":"d1824b1","source_repo":"340adb5d7","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":359,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","respond","respond","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
