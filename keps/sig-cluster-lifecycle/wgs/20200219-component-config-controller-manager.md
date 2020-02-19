---
title: Component Config for Kube Controller Manager
# Add more ? Get someone from api-machinery?
authors:
  - "@obitech"
  - "@mtaufen"
# Different SIG?
owning-sig: sig-api-machinery
participating-sigs:
  - sig-cluster-lifecyle
reviewers:
  - "@deads2k"
  - "@liggitt"
  - "@luxas"
  - "@stewart-yu"
  - "@sttts"
  - "@thockin"
  - "@lavalamp"
approvers:
  - "@deads2k"
  - "@liggitt"
  - "@luxas"
  - "@sttts"
  - "@thockin"
editor: "@obitech"
creation-date: 2019-02-19
last-updated: 2019-02-19
status: provisional
# see-also:
#   - "/keps/sig-aaa/20190101-we-heard-you-like-keps.md"
#   - "/keps/sig-bbb/20190102-everyone-gets-a-kep.md"
# replaces:
#   - "/keps/sig-ccc/20181231-replaced-kep.md"
# superseded-by:
#   - "/keps/sig-xxx/20190104-superceding-kep.md"
---

# Component Config for Kube Controller Manager

## Table of Contents

<!-- toc -->
- [Component Config for Kube Controller Manager](#component-config-for-kube-controller-manager)
  - [Table of Contents](#table-of-contents)
  - [Release Signoff Checklist](#release-signoff-checklist)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [Desired Properties](#desired-properties)
    - [Current architecture](#current-architecture)
    - [Option 1](#option-1)
    - [Option 2](#option-2)
    - [Proposed way forward](#proposed-way-forward)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
    - [Test Plan](#test-plan)
    - [Graduation Criteria](#graduation-criteria)
      - [Examples](#examples)
        - [Alpha -> Beta Graduation](#alpha---beta-graduation)
        - [Beta -> GA Graduation](#beta---ga-graduation)
        - [Removing a deprecated flag](#removing-a-deprecated-flag)
    - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Version Skew Strategy](#version-skew-strategy)
  - [Implementation History](#implementation-history)
  - [Drawbacks [optional]](#drawbacks-optional)
  - [Alternatives [optional]](#alternatives-optional)
  - [Infrastructure Needed [optional]](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core Kubernetes i.e., [kubernetes/kubernetes], we require the following Release Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

**Note:** Any PRs to move a KEP to `implementable` or significant changes once it is marked `implementable` should be approved by each of the KEP approvers. If any of those approvers is no longer appropriate than changes to that list should be approved by the remaining approvers and/or the owning SIG (or SIG-arch for cross cutting KEPs).

**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://github.com/kubernetes/enhancements/issues
[kubernetes/kubernetes]: https://github.com/kubernetes/kubernetes
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

<!-- Summarise why we need this -->

<!-- Link to other resourecs & previous work
My design doc: https://docs.google.com/document/d/1Q9kzmj0v_dPxXoNO9F6wiTX-hWnnrnyuMPbWVbVuO5Y/edit#

Luxas' design doc: https://docs.google.com/document/d/1tNnsH_uQU7uB3oUul0aX8R8SXUlKjGHVxJc_wHA9BMM/edit#heading=h.gb7lws4ts2j9

Enhancement issue: https://github.com/kubernetes/enhancements/issues/786

Stewart's KEP: https://github.com/kubernetes/enhancements/pull/1052
 -->

ComponentConfig is an ongoing effort to make Kubernetes-style config files the
preferred way to configure core Kubernetes components, instead of command-line
flags. An overview of and motivation for ComponentConfig, in general, can be
found in the [Versioned Component Configuration Files][component_configs]
doc (access is available to all members of `kubernetes-dev@googlegroups.com`).

This KEP is an effort to revive the conversation about bringing a component
config to the Controller Manager and by extension to Kubernetes controlers
in general. Previous work has been:

- The initial, work-in-progrss design document by @luxas ([#](https://docs.google.com/document/d/1tNnsH_uQU7uB3oUul0aX8R8SXUlKjGHVxJc_wHA9BMM))
- The initial enhancement issue that started the work ([#](https://github.com/kubernetes/enhancements/issues/786))
- The initial KEP by @stewart-yu, now stale ([#][kep_stuart])
- The work-tracking document by @obitech ([#](https://docs.google.com/document/d/1Q9kzmj0v_dPxXoNO9F6wiTX-hWnnrnyuMPbWVbVuO5Y))

Stewart's initial KEP has brought up a lot of valuable points of discussion but
unfortunately the work on it has stalled. This KEP aims to build upon these
points and provide ideas on how to resolve them. The primary goal of this KEP
is to open up the discussion again, as well as decide on a way forward to
implement a ComponentConfig design that works for the controller manager, as
well as other controller implementations.

## Motivation

As outline in the [original document][component_configs], component configs
offer a way to configure Kubernetes core components through configuration files
instead of command line flags. Many other components such as the kubelet,
kube-scheduler or kube-proxy can already consume a component config. A
[KubeControllerManagerConfiguration][kcm_config]
API exists and while technically there isn't anything preventing serializing a
it from file, there are some open question about implementing it in a
"sustainable" way:

- Do we need to version each individual controller? If yes, how is it implemented?
- How to handle fields that are shared between controllers?
- How can we handle future controllers and those, that are started outside of
  the context of the controller manager?

### Goals

- Discuss & decide on a design to implement component config(s) for Kubernetes
  controllers and the controller manager.
- Implement the proposed design through refactoring the existing API(s)

### Non-Goals

Lorem

## Proposal

First we're looking at properties we want an implementation of component
config for Kubernetes Controller to have. We will then look at the current
state of [KubeControllerManagerConfiguration][kcm_config],
as well as possible alternatives, against the background of our desired
properties. Lastly we will propose an architecture to move forward with.

### Desired Properties

1. Be able to configure and start controllers outside of the context
  of the controller manager (be able to start controllers from different
  binaries).
2. Be able to load configuration from config map.
3. Be able to share configuration fields between controllers.

### Current architecture

The Kubernetes controller manager is a "binary host" that is responsible
for starting Kubernetes controllers. All configuration is done via command line
flags passed to the component. Like that individual controllers can be enabled
or disabled, as well as configured.

Internally, each controller has its own configuration struct that is "bundled"
into the container type [KubeControllerManagerConfiguration][kcm_config]. Only
the container type has a `TypeMeta` field and is versioned.

The controller manager itself is configured via the
[GenericControllerManagerConfiguration](https://github.com/kubernetes/kubernetes/blob/v1.17.3/staging/src/k8s.io/kube-controller-manager/config/v1alpha1/types.go#L159) and
right now the only fields that are shared (consumed by more than one
controller) are found in the [KubeCloudSharedConfiguration](https://github.com/kubernetes/kubernetes/blob/v1.17.3/staging/src/k8s.io/kube-controller-manager/config/v1alpha1/types.go#L186)
struct.

Right now serializing the configuration from file is possible, as [this PR](https://github.com/kubernetes/kubernetes/pull/87370)
demonstrates:

```yaml
kind: KubeControllerManagerConfiguration
apiVersion: kubecontrollermanager.config.k8s.io/v1alpha1
Generic: 					        # GenericControllerManagerConfiguration
  Port: 1234
  Address: 0.0.0.0
  ClientConnection:			  # ClientConnectionConfiguration
    burst: 10
ResourceQuotaController:	# ResourceQuotaControllerConfiguration
  ConcurrentResourceQuotaSyncs: 15
```

However with the configuration of individual controllers being bound to the
container type, it's not practical to launch individual individual controllers
outside of the context of the controller manager, as it would require
serializing the whole configuration struct. This also prevents versioning
controllers individually, as right now they are all versioned
according to the container type.

### Option 1

Lorem

### Option 2

Lorem

### Proposed way forward

Lorem

### Risks and Mitigations

Lorem

## Design Details

Lorem

### Test Plan

Lorem

### Graduation Criteria

Lorem

#### Examples

These are generalized examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

##### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

##### Beta -> GA Graduation

- N examples of real world usage
- N installs
- More rigorous forms of testing e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least 2 releases between beta and GA/stable, since there's no opportunity for user feedback, or even bug reports, in back-to-back releases.

##### Removing a deprecated flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality which deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include [conformance tests].**

[conformance tests]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to make use of the enhancement?

### Version Skew Strategy

If applicable, how will the component handle version skew with other components? What are the guarantees? Make sure
this is in the test plan.

Consider the following in developing a version skew strategy for this enhancement:
- Does this enhancement involve coordinating behavior in the control plane and in the kubelet? How does an n-2 kubelet without this feature available behave when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI or CNI may require updating that component before the kubelet.

## Implementation History

Major milestones in the life cycle of a KEP should be tracked in `Implementation History`.
Major milestones might include

- the `Summary` and `Motivation` sections being merged signaling SIG acceptance
- the `Proposal` section being merged signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded

## Drawbacks [optional]

Why should this KEP _not_ be implemented.

## Alternatives [optional]

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other possible approaches to delivering the value proposed by a KEP.

## Infrastructure Needed [optional]

Use this section if you need things from the project/SIG.
Examples include a new subproject, repos requested, github details.
Listing these here allows a SIG to get the process for these resources started right away.

[kep_stewart]: https://github.com/kubernetes/enhancements/pull/1052
[component_configs]: https://docs.google.com/document/d/1FdaEJUEh091qf5B98HM6_8MS764iXrxxigNIdwHYW9c
[kcm_config]: https://github.com/kubernetes/kubernetes/blob/v1.17.3/staging/src/k8s.io/kube-controller-manager/config/v1alpha1/types.go#L84