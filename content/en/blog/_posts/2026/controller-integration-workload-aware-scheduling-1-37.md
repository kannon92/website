---
layout: blog
title: "Controller integration and workload aware scheduling"
date: 2026-05-13T10:35:00-08:00
slug: controller-integration-and-workload-aware-scheduling
author: >
  Kevin Hannon (Red Hat),
---

AI/ML workloads are getting a major experience update with the workload aware scheduling efforts.
Kubernetes 1.37 will have support for beta for Workload, PodGroup, GangScheduling, Preemption and DRA for pod groups.
The expectation of gang scheduling is that users should not have to worry about specifying it. Most HPC schedulers handle this for you.
The interesting challenge that cloud native workloads take on is supporting elasticity and declarative workload intention.
Preemption can have more granular control ie preempt a subset of the pods.
Topology aware scheduling and DRA both require APIs as it is impossible to surmise user intent.

To bridge the gap between workload aware scheduling and declarative workload APIs, there were two KEPs proposed in 1.37 that are worth highlighting:
[KEP-6089 (Workload Aware Scheduling Controller APIs)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/6089-was-controller-apis)
and [KEP-5547 (Integrate Workload APIs with Job Controller)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/5547-integrate-workload-with-job).
Together they give the ecosystem a shared plan for how controllers should expose WAS to their users, instead of every controller inventing its own scheduling vocabulary.

## Workload Builder Library

Once a controller wants to opt into gang scheduling, topology constraints, or disruption policies, it still has to do the unglamorous work of turning that intent into a `Workload` and `PodGroup`. Every controller author would otherwise write their own version of the same merge-defaults-then-validate logic. KEP-6089 avoids that by introducing a set of reusable building blocks under `scheduling.k8s.io` (things like `WorkloadPodGroupSchedulingPolicy`, `WorkloadPodGroupSchedulingConstraints`, and `WorkloadPodGroupDisruptionMode`) plus a shared Go library that compiles them.

That library, `workloadbuilder`, now lives in-tree under `k8s.io/component-helpers/scheduling/schedulingv1/workloadbuilder`. It was placed in `component-helpers` rather than `kube-scheduler` on purpose: it needs to stay dependency-light so out-of-tree controllers can vendor it without pulling in the scheduler's internals.

The core of the library is a `WorkloadItem`, a node in a controller's logical workload tree. A controller sets a `DefaultConfig` (its own sane defaults), an `Input` (the user's building blocks, mapped from whatever fields the controller exposes), and any `Callbacks` it needs for controller-specific defaulting, such as setting a gang's `MinCount` to a Job's parallelism when the user leaves it unset. Handing the root item to `workloadbuilder.NewBuilder` gives you a `Builder` that can validate the resolved configuration and compile it into a `Workload`, or generate the runtime `PodGroup` for a template.

Validation follows an allow-list rather than a deny-list. A controller passes the specific `SchedulingPolicyOption`s and `DisruptionModeOption`s it supports through `BuildOptions`, and anything outside that set is rejected. That matters because the building blocks are shared: if a new scheduling policy is added to `scheduling.k8s.io` for the benefit of one controller, every other controller that hasn't explicitly opted in keeps rejecting it, instead of silently inheriting a feature its reconciler was never taught to handle.

## Job Scheduling API

`Job` is the first consumer of this library, and the KEP treats it as the reference integration that proves the building blocks work before anyone else adopts them. `JobSpec` gains a new `scheduling` field of type `JobSchedulingConfiguration`, with `schedulingPolicy`, `schedulingConstraints`, `disruptionMode`, and up to four `resourceClaims`, gated behind the `WorkloadWithJob` feature gate on `kube-apiserver` and `kube-controller-manager`. The field stays alpha in v1.37, but it now lets users actually express intent, which is the gap left over from the initial v1.36 integration that only ever produced a hardcoded gang `PodGroup` for fully parallel indexed Jobs.

```yaml
apiVersion: batch/v1
kind: Job
spec:
  parallelism: 4
  completions: 4
  scheduling:
    schedulingPolicy:
      gang: {} # minCount is omitted, so the Job controller defaults it to parallelism (4)
    schedulingConstraints:
      topology:
        - level: "topology.kubernetes.io/zone"
    disruptionMode:
      all: {} # the whole gang is disrupted together, not pod by pod
  template:
    spec:
      containers:
        - name: train-node
          image: training-image:v1
```

If you leave `scheduling` out entirely, the Job controller defaults to `Basic`, meaning ordinary pod-by-pod scheduling. That keeps every existing Job manifest and CI pipeline behaving exactly as it does today. The `scheduling` field itself is immutable once set (you cannot add or remove it after creation), and the only part of it you're allowed to change later is `schedulingPolicy.gang.minCount`. Under the hood, the job controller builds a `WorkloadItem` from the Job's spec and hands it to `workloadbuilder` to compile the `Workload` and `PodGroup`, so it gets the same validation and defaulting behavior as every other adopter of the library.

## JobSet Scheduling API

KEP-6089 leaves the actual `JobSet` integration to the JobSet project, and that work is underway as its own [KEP](https://github.com/kubernetes-sigs/jobset/tree/main/keps/969-WAS-integration), behind a new `WorkloadAwareScheduling` feature gate. It lands as an optional `spec.scheduling` field on `JobSetSpec`, and it's fully opt-in: a JobSet that doesn't set it creates no `Workload` or `PodGroup` objects at all, and behaves exactly as it does today.

Of the two reference shapes KEP-6089 sketches out, JobSet picked the centralized "Targeted Policies" model over template delegation. All scheduling configuration, whether it applies to the whole JobSet or to a single `ReplicatedJob`, lives in one place at the root, and per-replica overrides target a `ReplicatedJob` by name through `replicatedJobPolicies`. That mirrors the `targetReplicatedJobs` pattern JobSet users already know from `FailurePolicy` and `SuccessPolicy`, and it avoids splitting scheduling intent between the root spec and the nested Job template, which also means JobSet doesn't have to wait on `Job` to ship its own `spec.scheduling` field first.

```yaml
apiVersion: jobset.x-k8s.io/v1alpha2
kind: JobSet
metadata:
  name: rdma-training
spec:
  scheduling:
    policy:
      gang: {}
    replicatedJobPolicies:
      - targetReplicatedJob: "driver"
        policy:
          basic: {}
      - targetReplicatedJob: "workers"
        constraints:
          topology:
            - level: "topology.kubernetes.io/rack"
        disruption:
          all: {}
  replicatedJobs:
    - name: driver
      replicas: 1
      # ...
    - name: workers
      replicas: 16
      # ...
```

Here the driver opts back out to `basic` scheduling and schedules independently, while the workers inherit the composite `gang` default, get pinned to the same rack, and are disrupted as a unit. When there's nothing to target and no sequencing involved, the controller compiles a single `Workload` for the whole JobSet and defaults its `gang.minCount` to the sum of `parallelism × replicas` across every `ReplicatedJob`. As soon as a `dependsOn` or `InOrder` `startupPolicy` is in play, though, one top-level `PodGroup` would deadlock, since not every pod exists yet when the first one needs to be admitted. The controller detects that and automatically falls back to a `PodGroup` per `ReplicatedJob` instead, each sized to its own `parallelism × replicas`.

Under the hood, JobSet maps each compiled group to a `workloadbuilder.WorkloadItem` and calls `BuildWorkload`, the same as `Job` does. The library doesn't yet support a real composite/leaf parent-child tree, so in the per-`ReplicatedJob` case JobSet builds one `Workload` per item and merges their `PodGroupTemplates` together before creating the object. Once the `Workload` exists, the controller stamps each child `Job` with the `scheduling.k8s.io/group-template-name` and `scheduling.k8s.io/parent-composite-podgroup` annotations from KEP-6089's downward mapping convention, and sets `pod.spec.schedulingGroup.podGroupName` so pods land in the right `PodGroup`. It's centralized management for this alpha: JobSet owns the `Workload` and every `PodGroup` it produces, and the child `Job` controllers see the `JobSet` owner reference and skip creating their own.

The scheduling objects also track the JobSet's lifecycle rather than sitting there statically. Suspending a JobSet deletes its `Workload` and `PodGroup`s so the scheduler stops holding a reservation for pods that aren't running, and resuming recreates them. Because both objects are immutable once created, scaling an `ElasticJobSet`'s parallelism means the controller has to delete and recreate them with the new `minCount` rather than patching in place.
