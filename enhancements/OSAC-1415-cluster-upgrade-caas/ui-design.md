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

A Tenant User or Tenant Admin chooses one target OpenShift version for a cluster. The UI sends one `Clusters.Update` request for `spec.version`. OSAC upgrades the control plane first, then automatically upgrades every node set to that version.

## Where the user starts

The cluster details page has a **Cluster version** rail. It shows **Current** from `status.observed_cp_version` (or `—` if unknown), **Upgrade available** when eligible, and the **Upgrade cluster** action.

During an active attempt, the rail says **Upgrading from X to Y** using the attempt's source and target. Once the attempt finishes, it returns to the current-version display. When history exists, **Upgrade history** opens a modal from the rail.

The Clusters table and configuration card show the observed control-plane **Current version**. The Node sets table shows each set's observed **Current version** from `status.node_sets[name].observed_version`. Use `—` for missing observations, and match lifecycle badges to the observed control-plane version. During an upgrade, affected rows show their observed-to-target transition.

## Choosing a target

**Upgrade cluster** opens a compact dialog with a read-only **Current version** and one required **Target version** selector. The selector displays `ClusterVersion.spec.version`, marks deprecated entries, and keeps the selected catalog entry's `id` as its value.

Use `semver` (`parse`, `gt`, `gte`, `rcompare`) in a shared helper, with `semver` and `@types/semver` added to `@osac/ui-components`. Fetch enabled ACTIVE/DEPRECATED versions with `CLUSTER_VERSION_ACTIVE_LIST_FILTER`. Skip invalid catalog versions. Offer only versions newer than the observed control plane that do not downgrade a node set or exceed the [same-major, three-minor skew limit](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/hosted_control_planes/hcp-updating/). Compare version strings, not catalog names, and sort newest first. Disable upgrading if an observed version is missing or invalid.

Enable the action when `CanUpgrade=True`, observed versions and targets are available, and the cluster state permits upgrades. `PROGRESSING` is permitted; `FAILED`, `DELETING`, and `DELETE_FAILED` are blocked. Show a specific disabled reason. The API rechecks admission at submission.

The dialog links **Check upgrade path and risks** to the external [OpenShift Update Graph](https://access.redhat.com/labs/ocpupgradegraph/update_path/), where tenants check target reachability and risks.

The dialog explains that the control plane upgrades first and node sets follow automatically. It warns that an accepted request cannot be cancelled or rolled back. **Cancel** dismisses the dialog before submission.

## Sending the request

Use `useUpdateResource(Clusters)` to call `Clusters.Update` with the selected catalog entry's ID as a `ClusterVersionReference`:

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

`object.id` identifies the cluster; `spec.version` carries the chosen version. The backend resolves that reference and performs the control-plane-then-node-set sequence.

Update the `useUpdateResource` request builder to preserve an explicit non-empty `updateMask`; keep automatic mask generation for callers that omit it. This request then masks exactly `spec.version`.

On API rejection, keep the selected target and show the error inline so the tenant can retry. On acceptance, the hook invalidates Cluster queries; the details page then shows progress.

## Following progress

After acceptance, an **Upgrade progress** card appears above the detail content, separate from provisioning status. A PatternFly `ProgressStepper` shows control plane, node sets, then finalizing. Node sets start after control-plane success and may run in parallel with each other. The cluster succeeds when every node set reaches the target.

The rail shows **Upgrading from X to Y** while the attempt is active; affected node sets say **Updating** while their stage is active. The **Upgrade history** modal includes the current attempt and one row per cluster request with X→Y, state, start and completion times, and failure or timeout messages.

Both views use a server-defined cluster-attempt record with a stable ID, source and target versions, aggregate state and times, and control-plane and node-set stages. The latest record represents the current or last attempt. On `TimedOut`, replace the active stepper with a warning naming the target and retain `TimedOut` in history even if observed versions later converge. `CanUpgrade` determines when another request is available.

## UI implementation

On `/clusters/:clusterId`, `ClusterDetailsPageContent` composes `ClusterVersionRail`, `UpgradeClusterDialog`, `ClusterUpgradeProgress`, and `ClusterUpgradeHistoryModal` under `components/Cluster/Details/`. The page owns modal visibility; the dialog owns target selection. Update `ClusterConfigurationCard`, `ClustersTable`, and `ClusterNodeSetsTab` for observed versions.

Use `useGetResource(Clusters, { id })` and project-scoped `useListResource(Clusters, { filter })` for the affected reads, preserving `useProjectFilterQuery`. Use `useListResource(ClusterVersions, { filter: CLUSTER_VERSION_ACTIVE_LIST_FILTER })` for choices. Migrate affected legacy Cluster readers to the generic query keys so `useUpdateResource(Clusters)` invalidates them together. Existing 10-second polling refreshes progress. Use PatternFly modal focus handling, a labeled selector, accessible disabled reasons, and announced request errors.

## Verification

Vitest/Testing Library with mocked Connect transport covers rail eligibility and observed-version labels; catalog and SemVer filtering, including missing data, prereleases, and skew; the exact `spec.version` request and mask, errors, and query invalidation; control-plane-first progress, one-attempt history, `TimedOut` with later observed convergence, and keyboard/focus behavior.

---

## Provenance

Authored: plan @ ui-design 0.1.0 - e2b0d87, workspace main @ d81c18823
Final: revise @ ui-design 0.1.0 - 7b2b8de, workspace main @ df9051d31

> Context changed between plan and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"ui-design","workflow_version":"0.1.0","ai_workflows":"7b2b8de","source_repo":"df9051d31","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":530,"main_ref":"main","phases":["plan","review-api","revise","revise","respond","respond","respond","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
