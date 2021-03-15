# KEP-2411: CRI Container Log Rotation

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

nodepools serve one purpose which is to tell the provider which sorts of nodes the cluster is seeking to provision. This strikes me as exactly backwards: The controller (eg: deployment, daemonset, statefulset) should declare a set of labels of nodes it is looking to run on, and along with that should be a hint to the cloud provider what nodes to provision. This should follow the pattern of the PersistentVolumeClaim where we should have something new like a NodeClaim. The autoscaler would function as normal to provision the nodes the deployment is looking for, and could be integrated.

## Motivation

Nodepool should just be deprecated. We should "shift left" the requests for nodes into something like the PersistentVolumeClaim's storageClass where deployments can ask for a "Default" or they can get down into the weeds and spell out specifically what they're looking for in a node.

### Goals

- deployments dictate what they want to run on, facilitated through labels or a similar mechanism to storage.

### Non-Goals

- less things to manage: Stop thinking about nodepools and let the cloud worry about provisioning stuff
- "shift left" the cluster design by letting developers pick the correct sort of nodes to run on (or accept the defaults)

## Proposal

Dispose of nodepools, and redo them as labels or similar. Cluster autoscalers should provision the nodes as requested by the application.

Our "nodelabels" need to consist of:
1. a region (eg: "us-central1")
1. desired amount of nodes (eg: "3")
1. labels of nodes (inherited from the deployment?) (eg: "app: myapp", "gpu: yes")
1. node type (eg: "m1-medium")

This also follows the cloud providers march towards less and less interaction with the nodes themselves. Examples are similar to google's shielded nodes idea where the node are simply something that runs but you can't do anything with except Kubernetes.

Similar to a namespace security policy, a network security policy, there should also be a node security policy which defines constraints related to provisioning nodes. EG: Cannot provision more than 6 nodes, cannot provision nodes of a certain type, cannot declare that nodes cannot be shared, etc.

### User Stories (Optional)

#### Story 1

As a kubernetes user, I want to provision my application without having to think at all about the nodes. I am happy with the defaults. When I provision my application via a Deployment, I did not include any labels, so I get a cluster default of 3 nodes of m1-medium sizing when I deploy. These nodes are unprovsioned when I undeploy. We can assume these nodes are shared as a "default" pool.

#### Story 2

As a kubernetes user, I want a minimum of six nodes with local disk and GPUs. I am OK with my deployment ending up on existing GPU nodes. My deployment would have the following mock up labels:
```yaml
...
nodeProvisioner:
  region: us-centeral1
  initialNodes: "6"
  desiredType: m1-gpu-medium
  localDiskSize: 100Gi
  shared: "true"
  nodeLabels:
    - name: app
      contents: "graphing calculator"
    - name: gpu
      contents: nvidia
...
```

I can then use something like the HPA to also manage the nodes which match those labels.

#### Story 3

As a malicious kubernetes user, I want to provision nodes to mine bitcoins. Because the administrator has declared that the GPU types are forbidden, the expected behavior here is that the workload never deploys because it fails to schedule.

### Notes/Constraints/Caveats (Optional)

### Risks and Mitigations

The on prem K8s cluster tends to not work like this - where nodes are preprovisioned. In this case, the user should completely ignore any of the node labeling and accept the defaults. (Looking for feedback here...)

## Design Details

What do we need more of here?

### Test Plan

- Find some cloud folks willing to give this a spin
- Find some on prem people to make sure nothing is offensive in the implementation

### Graduation Criteria

#### Alpha -> Beta Graduation

- nodes and autoscalers work as expected
- onprem works as expected and nodes can be preprovisioned

#### Beta -> GA Graduation

- Everyone agrees we have thought the potential nodes and configuration through and it meets criteria for all clouds.

### Upgrade / Downgrade Strategy

On Upgrade: feature will be available to use as it already is, but will be promoted to GA.

On downgrade: feature will be available to use when the feature gate is set, but will be moved back to Beta. You will be expected to clean up workers which have been created by this change.

### Version Skew Strategy

Ideally the workers have been recreated to be managed by this strategy so the version skew problem is mitigated.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

- **How can this feature be enabled / disabled in a live cluster?**
  - [ ] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name:
    - Components depending on the feature gate:
  - [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

- **Does enabling the feature change any default behavior?**

No. Worker nodes will be in the "default" nodelabel so they will be available to schedule pods on.

- **What happens if we reenable the feature if it was previously rolled back?**

Nothing.

- **Are there any tests for feature enablement/disablement?**

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

- **How can a rollout fail? Can it impact already running workloads?**

- **What specific metrics should inform a rollback?**

- **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**

- **Is the rollout accompanied by any deprecations and/or removals of features, APIs,
fields of API types, flags, etc.?**

yes: nodepools

### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

- **How can an operator determine if the feature is in use by workloads?**


- **What are the SLIs (Service Level Indicators) an operator can use to determine
the health of the service?**
  - [ ] Metrics
    - Metric name:
    - [Optional] Aggregation method:
    - Components exposing the metric:
  - [ ] Other (treat as last resort)
    - Details:

- **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**


- **Are there any missing metrics that would be useful to have to improve observability
of this feature?**

### Dependencies

_This section must be completed when targeting beta graduation to a release._

- **Does this feature depend on any specific services running in the cluster?**

  - tbd

### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field._

- **Will enabling / using this feature result in any new API calls?**

Yes

- **Will enabling / using this feature result in introducing new API types?**

  Yes

- **Will enabling / using this feature result in any new calls to the cloud
provider?**

  No

- **Will enabling / using this feature result in increasing size or count of
the existing API objects?**

  No

- **Will enabling / using this feature result in increasing time taken by any
operations covered by [existing SLIs/SLOs]?**

  No

- **Will enabling / using this feature result in non-negligible increase of
resource usage (CPU, RAM, disk, IO, ...) in any components?**

  No

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

- **How does this feature react if the API server and/or etcd is unavailable?**

Pods won't be able to be scheduled on specialized node types which don't exist

- **What are other known failure modes?**

- **What steps should be taken if SLOs are not being met to determine the problem?**

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

Original Issue:
First PR with implementation:
Original design doc with solutions considered:
Follow up PR:
Graduation to Beta: 
