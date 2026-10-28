---
title: cluster-upgrade-caas-ui
authors:
  - elayaha@gmail.com
creation-date: 2026-10-13
last-updated: 2026-10-28
tracking-link:
  - https://redhat.atlassian.net/browse/OSAC-1415
prd:
  - "prd.md"
see-also:
  - "design.md"
  - "https://github.com/osac-project/enhancement-proposals/pull/413"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Cluster Upgrade — CaaS UI

## Summary

A Tenant User or Tenant Admin chooses one target OpenShift version for a cluster. One `Clusters.Update` request changes `spec.version`. OSAC upgrades the control plane first, then brings every node set to the same version automatically. The UI has no separate node-set upgrade action. Add-on operator versions do not change.

## Where the user starts

The cluster details page has a **Cluster version** rail. It shows **Current** from `status.observed_cp_version` (or `—` if unknown), **Upgrade available** when eligible, and **Upgrade cluster**. The upgrade action appears only here; existing header actions stay, and there is no cluster-row action, Upgrades tab, or Overview card. The rail does not show a standing **Desired** version.

During an active attempt, the rail says **Upgrading from X to Y** using the attempt's source and target. Remove that label after success, failure, or timeout. Show **Upgrade history** in the rail only when history exists; it opens a modal.

The Clusters table and configuration card show the observed control-plane **Current version**. The Node sets table shows each set's observed **Current version** from `status.node_sets[name].observed_version`. Use `—` for missing observations, and match lifecycle badges to the observed control-plane version. During an upgrade, affected rows show their observed-to-target transition. None of these views adds a Desired-version column.

## Choosing a target

**Upgrade cluster** opens a compact dialog with a read-only **Current version** and one required **Target version** selector. The selector displays `ClusterVersion.spec.version`, marks deprecated entries, and keeps the selected catalog entry's `id` as its value. It has no component picker, node-set version list, or multi-hop builder.

Use the `semver` v7 package as a direct dependency of `@osac/ui-components`, with `@types/semver` for TypeScript. A pure helper shared by the rail and dialog applies these basic checks:

1. Request `ClusterVersions.List` with `CLUSTER_VERSION_ACTIVE_LIST_FILTER`, which includes only enabled ACTIVE or DEPRECATED entries.
2. Parse each entry's `spec.version`, the observed control-plane version, and every observed node-set version with `semver.parse(value, { loose: false })`. Omit an invalid catalog version. If any observed version is missing or cannot be parsed, disable the upgrade with a reason.
3. Keep a target only if `semver.gt(target, observedControlPlane)`. Compare version numbers, never catalog names. Direct comparison also handles catalog prereleases by SemVer precedence; do not use range matching to silently omit them.
4. For every node set, require `semver.gte(target, observedNodeSet)`, the same major version, and `target.minor - observedNodeSet.minor <= 3`. This checks the temporary skew after the control plane moves but before node sets catch up. For example, a 4.20 control plane with a 4.17 node set cannot offer 4.21: that node set would briefly be four minors behind. See the [HCP skew policy](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/hosted_control_planes/hcp-updating).

Sort the remaining choices newest first with `semver.rcompare`; this changes their order, not their eligibility.

The rail action also needs `CanUpgrade=True` and a cluster state other than `FAILED`, `DELETING`, or `DELETE_FAILED`. `PROGRESSING` alone does not block it. Show a specific disabled reason when admission, observations, or eligible targets are unavailable. The API rechecks admission at submission and remains authoritative.

These checks do not establish a directly reachable OpenShift upgrade path. The dialog links **Check upgrade path and risks** to the external [OpenShift Update Graph](https://access.redhat.com/labs/ocpupgradegraph/update_path/); tenants check whether the selected target is directly reachable and review risks there. OSAC does not query the graph.

The dialog explains that the control plane upgrades first and node sets follow automatically. It warns that an accepted request cannot be cancelled or rolled back; **Cancel** only closes the dialog before submission. Show typical duration and “included in this version” copy only when an approved source supplies them.

## Sending the request

There is no `useResourceUpgrade` hook in the current UI. Use `useUpdateResource(Clusters)`, which calls `Clusters.Update`. Submit the selected catalog entry's ID as a `ClusterVersionReference`:

```ts
const updateCluster = useUpdateResource(Clusters);

updateCluster.mutate({
  object: {
    id: cluster.id,
    spec: { version: { id: selectedVersion.id } },
  },
  updateMask: { paths: ['spec.version'] },
});
```

`object.id` identifies the cluster. The only field to change is `spec.version`; the backend resolves that reference and performs the control-plane-then-node-set sequence. Do not copy the whole Cluster or send `spec.node_sets`, status, add-on operators, or a release image.

Today `useUpdateResource` derives a mask from every field in `object` and overwrites a supplied mask. Extend its request builder to preserve an explicit non-empty `updateMask` for this call, while retaining automatic masks for existing callers. That is necessary to send exactly `spec.version`, rather than nested reference fields or `id`, in the mask.

On API rejection, keep the selected target and show the error inline without automatic retry. On acceptance, the hook invalidates Cluster queries; the details page then shows progress.

## Following progress

After acceptance, an **Upgrade progress** card appears above the detail content. A PatternFly `ProgressStepper` shows control plane, node sets, then finalizing. Node sets start only after control-plane success and may run in parallel with each other. The cluster succeeds only when every node set reaches the target. A steady cluster has no progress card. Keep upgrade progress separate from provisioning status, and show remaining time only from an approved source.

The rail shows **Upgrading from X to Y** while the attempt is active; affected node sets say **Updating** while their stage is active. The **Upgrade history** modal includes the current attempt and one row per cluster request with X→Y, state, start and completion times, release notes when available, and failure or timeout messages.

Both views use the same server-defined cluster-attempt record. Do not group component history entries in the browser. On `TimedOut`, replace the active stepper with a warning naming the target and retain the recorded outcome in history. Observed versions continue to refresh if HyperShift later converges; that does not rewrite the recorded outcome. `CanUpgrade` determines whether another request is available.

## UI implementation

On `/clusters/:clusterId`, `ClusterDetailsPageContent` composes `ClusterVersionRail`, `UpgradeClusterDialog`, `ClusterUpgradeProgress`, and `ClusterUpgradeHistoryModal` under `components/Cluster/Details/`. The page owns modal visibility; the dialog owns target selection. Keep the existing conditions and networking content. Update `ClusterConfigurationCard`, `ClustersTable`, and `ClusterNodeSetsTab` for observed versions. No new route or global state is needed.

Use `useGetResource(Clusters, { id })` and project-scoped `useListResource(Clusters, { filter })` for the affected reads, preserving `useProjectFilterQuery`. Use `useListResource(ClusterVersions, { filter: CLUSTER_VERSION_ACTIVE_LIST_FILTER })` for choices. Migrate affected legacy Cluster readers to the generic query keys so `useUpdateResource(Clusters)` invalidates them together. Existing 10-second polling refreshes progress; add no separate polling loop. Use PatternFly modal focus handling, a labeled selector, accessible disabled reasons, and announced request errors.

## Backend dependencies

**Upgrade sequencing.** The merged [PRD](prd.md) and [backend design](design.md) still describe independent component requests. Open [PR #412](https://github.com/osac-project/enhancement-proposals/pull/412) proposes one `spec.version` request for the sequence. Fulfillment currently handles one component change per edit, so this orchestration must change before the UI ships. The backend must settle the HCP version ceiling and recovery for a node set already at N-3 skew; the UI does not invent a recovery sequence. [UX PR #413](https://github.com/osac-project/enhancement-proposals/pull/413) still needs its simultaneous-upgrade and pre-request Desired copy aligned.

**One attempt record.** PR #412 currently exposes per-component `status.upgrade_history` and keeps attempt IDs private. Progress and history need one public cluster-attempt record per request, with a stable ID, source/target, aggregate state and times, and control-plane/node-set stages. The latest record is current/latest; there is no separate upgrade status. Define `TimedOut`, partial and late-convergence behavior, finalizing/ETA, and `CanUpgrade`, then regenerate the UI types.

**Version information.** PR #412 excludes OSAC upgrade-path discovery and risk review for this phase. Catalog filtering is the UI's basic preview; the tenant uses the external graph link. Define approved sources for duration, version-offer copy, and release-notes URLs; `ClusterVersion` currently exposes none of these.

## Verification

Vitest/Testing Library with mocked Connect transport covers rail eligibility and observed-version labels; catalog and SemVer filtering, including missing data, prereleases, skew, and the external graph warning; the exact `spec.version` request and mask, errors, and query invalidation; control-plane-first progress, one-attempt history, `TimedOut` with later observed convergence, and keyboard/focus behavior. Backend contract tests must prove that one request upgrades the control plane before node sets and reports success only after all reach the target.

---

## Provenance

Authored: plan @ ui-design 0.1.0 - e2b0d87, workspace main @ d81c18823
Final: revise @ ui-design 0.1.0 - 7b2b8de, workspace main @ df9051d31

> Context changed between plan and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"ui-design","workflow_version":"0.1.0","ai_workflows":"7b2b8de","source_repo":"df9051d31","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":530,"main_ref":"main","phases":["plan","review-api","revise","revise","respond","respond","respond","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
