---
title: cluster-upgrade-caas-ui
authors:
  - elayaha@gmail.com
creation-date: 2026-10-13
last-updated: 2026-10-25
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

Tenant Users and Tenant Admins choose one OpenShift target version from cluster details. One `Clusters.Update` changes `spec.version`. OSAC upgrades the control plane first, then automatically moves every node set to the same version. The UI offers no separate node-set upgrade action; add-on operator versions remain unchanged.

## Entry and versions

- The details **Cluster version** rail shows **Current** from `status.observed_cp_version` (`—` when absent), **Upgrade available** when eligible, and an **Upgrade cluster** link. After acceptance, **Desired** resolves `spec.version` to its semver. The UX mock's pre-request suggested Desired value needs a selection rule; it is not the stored desired version.
- **Upgrade history** opens a modal from the rail; omit the link when history is empty. Keep existing header actions. Remove the planned upgrade action from the header and cluster-row menu; add no Upgrades tab or Overview card.
- The Clusters table and configuration card show observed **Current version**, with lifecycle badges matched to that version. The Node sets table adds one observed **Current version** column (`—` when absent). During an upgrade, the rail and affected rows label their observed→target transition; add no desired-version column.

## Upgrade dialog

**Upgrade cluster** opens a compact modal with read-only Current version and one required **Target version** selector. Offer enabled ACTIVE/DEPRECATED catalog versions newer than the observed control-plane version, subject to the control-plane-first admission and skew rules; the API makes the final decision. These choices do not guarantee a directly reachable path in the OpenShift upgrade graph. Explain blocked actions or confirmation when observations, admission, or targets are unavailable, or cluster state is `FAILED`, `DELETING`, or `DELETE_FAILED`. `PROGRESSING` alone does not block an admitted upgrade.

The modal says the control plane upgrades first and node sets automatically follow to the same version. It shows approved typical-duration and “included in this version” copy when supplied, and warns that an accepted upgrade cannot be cancelled or rolled back. It has no component picker, node-set version list, or multi-hop builder. Link the external [OpenShift Update Graph](https://access.redhat.com/labs/ocpupgradegraph/update_path/) for the tenant's path/risk check; OSAC does not verify it. **Cancel** dismisses before submission. **Upgrade cluster** changes and masks only `spec.version`; it sends no node-set, operator, or status update. Keep the selection and show API errors inline without automatic retry.

## Progress and history

After acceptance, show an **Upgrade progress** card above the detail content. Its PatternFly `ProgressStepper` shows control-plane, automatic node-set, then finalizing status for one cluster attempt. Node sets begin only after control-plane success; they may upgrade in parallel with one another. Report success only after every node set reaches the target. The rail says **Upgrading** and affected node sets say **Updating** when their stage is active. Keep this separate from provisioning status. Show remaining time only from an approved source.

The **Upgrade history** modal shows one row per cluster request: Version (X→Y), State, Started, Completed, and View release notes, with failure/timeout messages available. The progress card and history modal require the same server-defined cluster attempt. Include the current attempt; steady clusters show no progress card. On `TimedOut`, replace the active stepper with a warning and retain the row. Observed versions keep refreshing; do not infer success from late convergence. Admission controls the next upgrade. Do not group component entries into cluster attempts in the browser.

## UI implementation

On the existing `/clusters/:clusterId` route, `ClusterDetailsPageContent` composes new `ClusterVersionRail({ cluster, onUpgrade, onHistory })`, `UpgradeClusterDialog({ cluster, onClose })`, `ClusterUpgradeProgress({ cluster })`, and `ClusterUpgradeHistoryModal({ cluster, onClose })` under `components/Cluster/Details/`. It owns modal visibility; the dialog owns target selection. Keep existing conditions/networking content. Update `ClusterConfigurationCard`, `ClustersTable`, and `ClusterNodeSetsTab` for observed versions. No new route or global state is needed.

Use generic `useGetResource(Clusters, { id })` and `useListResource(Clusters, { filter })` for affected details and list reads, preserving `useProjectFilterQuery` scoping. Use `useListResource(ClusterVersions, { filter: CLUSTER_VERSION_ACTIVE_LIST_FILTER })` for target choices. Migrate affected legacy readers so `useUpdateResource(Clusters)` invalidates their matching generic query keys without a second cache path. Existing 10-second polling refreshes progress. Use PatternFly modal focus handling, a labeled selector, accessible disabled reasons, and announced request errors.

## Contract work before implementation

- The merged [PRD](prd.md) and [backend design](design.md) still describe independent component upgrades. Open [PR #412](https://github.com/osac-project/enhancement-proposals/pull/412) proposes one `spec.version` request that upgrades the control plane first, then node sets automatically. Current Fulfillment handles one component version change per edit; this orchestration must be implemented before the full UI flow ships. Finalize the HCP version ceiling and recovery when a node set starts at N-3 skew; do not invent a UI recovery sequence. [UX PR #413](https://github.com/osac-project/enhancement-proposals/pull/413) still says simultaneous and needs its copy and progress stages aligned.
- PR #412's public `status.upgrade_history` has one entry per component and keeps attempt IDs private. Before implementing progress and history, the backend must expose one cluster-attempt record per request with a stable public ID, source/target versions, aggregate state/times, and control-plane/node-set stages. The latest record is current/latest, with no separate upgrade status; both views consume it. Define `TimedOut`/partial/late-convergence outcomes, finalizing/ETA, and `CanUpgrade`, then update the UI generated types.
- The merged PRD calls for directly reachable targets, while PR #412 excludes graph discovery and assigns the path/risk check to the tenant. This UI has no graph eligible-target set; the dialog links the external graph without claiming to verify a path. Define the pre-request Desired suggestion and sources for duration, version-offer copy, and release-notes URLs. `ClusterVersion` exposes none of the last three; show them only from approved data.

## Verification

Vitest/Testing Library with mocked Connect transport covers rail eligibility, catalog/admission choices and the external graph warning, dialog/request/error behavior, generic query invalidation, observed versions, control-plane-first stages, one-attempt progress/history, timeout followed by late convergence, and keyboard/focus behavior. Backend contract tests must prove one `spec.version` request upgrades the control plane before node sets and reports aggregate outcomes, with success only after all components reach the target.

---

## Provenance

Authored: respond @ ui-design 0.1.0 - e2b0d87, workspace main @ d81c18823
Phases: plan, review-api, revise, revise, respond

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"ui-design","workflow_version":"0.1.0","ai_workflows":"e2b0d87","source_repo":"d81c18823","source_repo_branch":"main","commits_behind_main":0,"commits_ahead_main":446,"main_ref":"main","phases":["plan","review-api","revise","revise","respond"],"authoring_modes":["skill"],"context_changed":false,"origin_untracked":false} -->
