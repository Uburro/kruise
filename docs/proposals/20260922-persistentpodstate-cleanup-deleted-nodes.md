---
title: Clean Up PersistentPodState After Node Deletion
creation-date: 2026-09-22
last-updated: 2026-09-22
status: provisional
---

# Clean Up PersistentPodState After Node Deletion

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
- [Goals and Non-Goals](#goals-and-non-goals)
- [Proposal](#proposal)
- [Implementation Details](#implementation-details)
- [Risks and Limitations](#risks-and-limitations)
- [Alternatives](#alternatives)
- [Upgrades](#upgrades)
- [Test Plan](#test-plan)
- [Open Questions](#open-questions)
- [References](#references)

## Summary

Add an Alpha feature gate named `PersistentPodStateAutoCleanup`. It is off by default.
When it is on, PersistentPodState removes saved PodState entries for Nodes that no longer exist.
This rule applies to workloads that use required persistent topology.
New Pods can then use another Node without the old topology rules.

While the saved Node exists, PersistentPodState keeps its required topology rules.
The feature uses the existing API and annotations. It does not need new CRD fields.
This document describes a proposed feature. It does not include code changes.

## Motivation

GPU workloads may need to return to the same Node after a Pod restart.
For example, the Node may already have downloaded models in its local cache.
With `kruise.io/required-persistent-topology: kubernetes.io/hostname`, PersistentPodState saves the Node's hostname.
It adds that value to the new Pod's `nodeSelector`.

A Node replacement process can delete the Node.
Today, PersistentPodState can remove saved state when a workload scales down or is deleted.
It does not remove saved state because a Node was deleted.
As a result, a new Pod may require a hostname that no longer exists in the cluster.

We want to keep the required topology rules while the Node exists.
When the Node no longer exists in the Kubernetes API, PersistentPodState should remove its saved state.

### Why Preferred Topology Is Not Enough

`preferred-persistent-topology` asks the scheduler to prefer the saved Node.
It does not require the scheduler to choose it.
Even with `weight: 100`, other scheduling scores can affect the final choice.
See the [Kubernetes node affinity documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity-weight).

Preferred topology is enough when using another Node at any time is acceptable.
It also lets a new Pod use another suitable Node after the saved Node is deleted.
However, this use case needs a stricter rule: keep the saved Node while it exists.
For example, an operator may choose to wait for that Node to have a free GPU again.
This can avoid downloading large models again on another Node.
The operator accepts a longer wait to keep that placement.

For required hostname topology, the proposed behavior is:

| Saved Node | Preferred topology | Required topology with this feature |
| --- | --- | --- |
| Exists and can run the Pod | The scheduler may choose another Node. | Keep the saved hostname rule. |
| Exists but cannot run the Pod now | The scheduler may use another suitable Node. | Keep the rule and wait. |
| No longer exists | A new Pod may use another suitable Node. | Remove saved state and skip its rules for new Pods. |

The reason to release the required rule is confirmed Node deletion.
A busy GPU, cordon, or a temporary Node problem does not release it.
This does not fix existing Pending Pods, as explained in [Risks and Limitations](#risks-and-limitations).

### Example

This example uses a Pod named `gpu-worker-0` and a Node named `compute-xxx`.
These are example names, not data from a real cluster.
The workload has these annotations:

```yaml
metadata:
  annotations:
    kruise.io/auto-generate-persistent-pod-state: "true"
    kruise.io/required-persistent-topology: kubernetes.io/hostname
```

After `gpu-worker-0` becomes Ready on `compute-xxx`, the controller saves its state in PersistentPodState.
The YAML below is part of the **PersistentPodState status**, not the Pod status.
Each key under `podStates` is a Pod name. Here, that key is `gpu-worker-0`.

```yaml
status:
  podStates:
    gpu-worker-0:
      nodeName: compute-xxx
      nodeTopologyLabels:
        kubernetes.io/hostname: compute-xxx
```

With the proposed gate on, cleanup removes `status.podStates["gpu-worker-0"]` when `compute-xxx` is deleted.
A new `gpu-worker-0` created after Node deletion does not get the old hostname rule.
This also applies when the controller has not finished cleanup yet.
The scheduler can choose another GPU Node that meets the Pod's other rules.
When the Pod becomes Ready, PersistentPodState saves its new state as usual.

## Goals and Non-Goals

Goals:

- Remove old state after the API confirms that the Node no longer exists.
- Keep required topology while the saved Node exists, even when it is not ready.
- Stop the webhook from adding old topology rules while cleanup is still pending.
- Keep current behavior when the feature gate is off.
- Handle Nodes deleted while the controller was stopped.

This first proposal does not cover:

- Cleanup because a Node is NotReady, cordoned, tainted, or has no free GPUs.
- Deleting, evicting, or changing existing Pods to fix their scheduling rules.
- Creating new GPU capacity or moving data from a Node's local disk.
- Changing user-defined scheduling rules, PVC topology, or workload retention policies.
- Detecting a new Node with the same name by its UID. PodState saves only the Node name.

## Proposal

### Feature Gate

Add `PersistentPodStateAutoCleanup` to `pkg/features/kruise_features.go`.
Its stage is Alpha, and its default value is `false`.
To turn it on, operators would use this kruise-manager flag:

```text
--feature-gates=PersistentPodStateAutoCleanup=true
```

The feature applies to PersistentPodState objects with at least one key in `requiredPersistentTopology`.
It works for PersistentPodState objects created by users and those created from workload annotations.
PersistentPodState objects that use only preferred topology keep their current behavior.
The gate applies to all matching PersistentPodState objects managed by kruise-manager.
This proposal does not add a separate switch for each workload.

### Cleanup Rules

Use `status.podStates[podName].nodeName` to find the Node.
Do not use the hostname label as the Node name. The two values can be different.

| Condition | Proposed behavior |
| --- | --- |
| Gate is off | Keep current behavior. Do not add Node checks. |
| No required topology | Keep current behavior. |
| Saved Node exists | Keep the PodState and use its topology as usual. |
| Node is NotReady, cordoned, tainted, or marked for deletion but still exists | Keep the PodState. |
| A direct API read returns NotFound for the saved Node | Remove that PodState. Do not add its topology to new Pods. |
| Node check returns a timeout, access error, or another error | Keep the state. The controller retries later. The webhook returns an error. |
| Saved Node name is empty | Keep the entry because the Node cannot be checked. |

The proposed cleanup removes the whole PodState entry, including saved topology labels and annotations.
It keeps other PodState entries and the PersistentPodState object itself.
Cleanup works with both `WhenScaled` and `WhenDeleted`.
These policies still control when state is removed after workload changes.
The new gate adds Node deletion as another reason to remove state.

## Implementation Details

### Controller

Run cleanup in the PersistentPodState reconcile loop. Write changes through the status subresource.
If several entries use the same Node, check that Node only once per loop.
A Node missing from the cache may still exist in the API.
Before removing state, use a direct API read to confirm NotFound.
Old Node data in the cache may delay cleanup until the cache is updated.

Check matching PersistentPodState objects with saved Node names once per minute.
This also handles workloads with no new events, saved state after scale-down, and Nodes deleted during controller downtime.
Stop these regular checks when no saved Node names remain.
API errors or a busy work queue may delay a check.

If a Node check or status update returns an error, use the normal controller retry process.
If a status update has a conflict, read the latest PersistentPodState and check its entries again.
Do not replace new saved state with an old copy.
Running cleanup more than once must give the same result.
A Pod left on a deleted Node must not restore old state from the Node cache.

After a successful cleanup, write a structured log with the PersistentPodState, Pod name, and deleted Node name.
Keep Node checks and cleanup in the reconcile loop, outside event handlers.

### Pod Webhook

The webhook runs when a new Pod is created.
Before adding saved topology for a matching PersistentPodState entry, it checks the Node through a direct API read.
On NotFound, it skips the saved entry. Rules from the Pod template and other webhooks still apply.
On other errors, it returns an error so the workload controller can retry Pod creation.
It must not remove required topology rules just because a Node check returned an error.

The webhook must not change PersistentPodState status or delete Pods. This also applies to dry-run requests.
The controller is responsible for removing saved state.
The webhook adds a Node check only when a new Pod has a matching saved entry.
This check protects Pod creation when the Node is deleted before the controller finishes cleanup.

## Risks and Limitations

### Pods Created Before Node Deletion

A new Pod may be created while the old Node still exists, for example during a drain.
The Pod gets the saved hostname rule. Then the Node may disappear before the Pod is scheduled.
Cleaning PersistentPodState status does not remove this rule from an existing Pod.
An operator or another recovery tool must recreate that Pod after Node deletion.

A Node can also be deleted between the webhook's Node check and the API saving the new Pod.
The feature therefore cannot recover Pods in every drain or Node replacement case.
Automatic recovery of these Pending Pods needs a separate design.
It must define when Pod deletion is allowed and how to respect disruption protection.
It must also track which scheduling rules were added by PersistentPodState.

### Removed State

A deleted Node's zone may still contain other Nodes.
However, this proposal removes all saved labels and annotations in that PodState entry.
Some of these values may still be useful.
Turning on the gate can therefore allow a different zone and remove saved annotations.
Upstream reviewers should discuss whether to remove the whole entry or keep some parts.

The feature cannot recover data stored only on the deleted Node.
Existing volume rules and user-defined scheduling rules still apply.

### Node Identity and API Load

A new Node may use the same name as a deleted Node.
If it exists when the check runs, PersistentPodState treats it as the saved Node.
Detecting this replacement would need more saved data, such as a Node UID.
This proposal does not add that data.

Regular checks add cache reads for each different Node name in a matching PersistentPodState.
When a Node is missing from the cache, the controller also makes a direct API read.
Each new Pod with a matching saved entry needs one direct Node read in the webhook.
Test this extra load with large workloads before moving the feature beyond Alpha.

## Alternatives

- **Preferred persistent topology:** suitable when moving to another Node at any time is acceptable.
  It does not provide the rule described in [Why Preferred Topology Is Not Enough](#why-preferred-topology-is-not-enough).
- **Manual cleanup:** an operator removes saved state for each deleted Node.
  A new Pod can still get old topology rules before cleanup finishes.
- **Watch Node deletion events:** can start cleanup sooner.
  It needs a fast way to find PersistentPodState objects for each Node and a way to handle missed events.
  Start with regular checks, then consider a watch if large cluster tests show a need.
- **Remove only Node topology:** keeps saved annotations.
  We would need to define how PersistentPodState uses and updates the remaining state.
- **Add a policy to each PersistentPodState:** lets users choose which workloads use cleanup.
  This needs API changes, version conversion, validation, and updated CRDs.

## Upgrades

Upgrades keep current behavior when the gate is off.
Turning it on starts cleanup for all matching PersistentPodState objects, including existing ones.
This includes state for Nodes deleted before the upgrade.
All kruise-manager replicas should use the same gate setting.
This keeps the controller and webhooks working with the same rules during an upgrade.

Turning the gate off stops new cleanup and webhook Node checks.
It does not restore removed entries. PersistentPodState saves new state when a Pod becomes Ready.
There are no proposed changes to API versions or stored data formats.

## Test Plan

Add controller and webhook tests with the gate both on and off:

- Keep current behavior when the gate is off or PersistentPodState uses only preferred topology.
- Keep state for existing Nodes, including NotReady, cordoned, and terminating Nodes.
- Remove only entries with confirmed NotFound results. Test both retention policies.
- Keep entries with empty Node names. Use Node names, not hostname labels, for checks.
- Keep state when a Node is missing from the cache but exists in the API.
- Keep state when the API returns an error other than NotFound.
- Find deleted Nodes without new Pod events and after a controller restart.
- Do not restore removed state from a remaining Ready Pod and old Node cache data.
- Get the same result when cleanup runs again. Keep new saved state when a status conflict occurs.
- Skip old topology in the webhook before cleanup finishes. Keep Pod template rules.
- Return a webhook error when the Node check returns an error other than NotFound.
- Do not change stored objects during a dry-run request.
- Save new state when the replacement Pod becomes Ready on another Node.
- Check that existing Pending Pods are not recreated automatically.

An end-to-end test should delete the saved Node and create a new Pod before PersistentPodState cleanup finishes.
Check that the Pod can use another suitable Node and that PersistentPodState saves its new state.
A second test should scale the workload to zero, delete the saved Node, and scale the workload up again.

## Open Questions

1. Should cleanup work with all required topology keys, or only with `kubernetes.io/hostname`?
   The second option would keep zone-only rules unchanged.
2. Should cleanup remove the whole PodState entry or keep annotations and shared topology values?
3. Is one global gate enough, or should users also turn cleanup on for each PersistentPodState?
4. Are regular checks enough for large clusters, or should the first version watch Node deletion events?
5. Should we design recovery for existing Pending Pods later, or is it needed for the first version?

## References

- [Original PersistentPodState proposal](20220421-persistent-pod-state.md)
- [PersistentPodState custom workload support](20220824-persistentpodstate-custom-workload-support.md)
- [PersistentPodState user documentation](https://openkruise.io/docs/user-manuals/persistentpodstate)
- [Kubernetes Pod placement constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [PersistentPodState controller](../../pkg/controller/persistentpodstate/persistent_pod_state_controller.go)
- [PersistentPodState Pod admission handler](../../pkg/webhook/pod/mutating/persistent_pod_state.go)
