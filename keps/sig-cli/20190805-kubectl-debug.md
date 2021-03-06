---
title: kubectl debug
authors:
  - "@verb"
owning-sig: sig-cli
participating-sigs:
  - sig-cli
  - sig-node
reviewers:
  - "@aylei"
  - "@soltysh"
approvers:
  - "@pwittrock"
  - "@soltysh"
editor: TBD
creation-date: 2019-08-05
last-updated: 2019-08-06
status: implementable
see-also:
  - "/keps/sig-node/20190212-ephemeral-containers.md"
  - "/keps/sig-release/20190316-rebase-images-to-distroless.md"
---

# Pod Troubleshooting

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Use Cases](#use-cases)
    - [Distroless Containers](#distroless-containers)
    - [Kubernetes System Images](#kubernetes-system-images)
    - [Operations and Support](#operations-and-support)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Pod Troubleshooting with Ephemeral Debug Container](#pod-troubleshooting-with-ephemeral-debug-container)
    - [Debug Container Naming](#debug-container-naming)
    - [Container Namespace Targeting](#container-namespace-targeting)
    - [Interactive Troubleshooting and Automatic Attaching](#interactive-troubleshooting-and-automatic-attaching)
    - [Proposed Ephemeral Debug Arguments](#proposed-ephemeral-debug-arguments)
  - [Pod Troubleshooting by Copy](#pod-troubleshooting-by-copy)
    - [Creating a Debug Container by copy](#creating-a-debug-container-by-copy)
    - [Modify Application Image by Copy](#modify-application-image-by-copy)
  - [Node Troubleshooting with Privileged Containers](#node-troubleshooting-with-privileged-containers)
  - [User Stories](#user-stories)
    - [Operations](#operations)
    - [Debugging](#debugging)
    - [Automation](#automation)
    - [Technical Support](#technical-support)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

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

This proposal adds a command to `kubectl` to improve the user experience of
troubleshooting. Current `kubectl` commands such as `exec` and `port-forward`
allow troubleshooting at the container and network level. `kubectl debug`
extends these capabilities to include Kubernetes abstractions such as Pod, Node
and [Ephemeral Containers].

User journeys supported by the initial release of `kubectl debug` are:

1. Create an Ephemeral Container in a running Pod to attach debugging tools
   to a distroless container image. (See [*Pod Troubleshooting with Ephemeral
   Containers*](#pod-troubleshooting-with-ephemeral-debug-container))
2. Restart a pod with a modified `PodSpec`, to allow in-place troubleshooting
   using different container images or permissions. (See [Pod Troubleshooting by
   Copy](#pod-troubleshooting-by-copy))
3. Start and attach to a privileged container in the host namespace. (See [*Node
   Troubleshooting with Privileged Containers*](
   #node-troubleshooting-with-privileged-containers))

[Ephemeral Containers]: https://git.k8s.io/enhancements/keps/sig-node/20190212-ephemeral-containers.md

## Motivation

### Use Cases

#### Distroless Containers

Many developers of native Kubernetes applications wish to treat Kubernetes as an
execution platform for custom binaries produced by a build system. These users
can forgo the scripted OS install of traditional Dockerfiles and instead `COPY`
the output of their build system into a container image built `FROM scratch` or
a [distroless container image]. This confers several advantages:

1.  **Minimal images** lower operational burden and reduce attack vectors.
2.  **Immutable images** improve correctness and reliability.
3.  **Smaller image size** reduces resource usage and speeds deployments.

The disadvantage of using containers built `FROM scratch` is the lack of system
binaries provided by a Linux distro image makes it difficult to
troubleshoot running containers. Kubernetes should enable one to troubleshoot
pods regardless of the contents of the container images.

#### Kubernetes System Images

Kubernetes itself is migrating to [distroless for k8s system images] such as
`kube-apiserver` and `kube-dns`. This led to the creation of a [scratch
debugger] to copy debugging tools into the running container, but this script
has some downsides:

1. Since it's not possible to install the debugging tools using only Kubernetes,
   the script issues docker commands directly and so isn't portable to other
   runtimes.
2. The script requires broad administrative access to the node to run docker
   commands.
3. Installing tools into the running container requires modifying the image
   being debugged.

`kubectl debug` would replace the scratch debugger script with a method
idiomatic to Kubernetes.

#### Operations and Support

As Kubernetes gains in popularity, it's becoming the case that a person
troubleshooting an application is not necessarily the person who built it.
Operations staff and Support organizations want the ability to attach a "known
good" or automated debugging environment to a pod.

### Goals

Make available to users of Kubernetes a troubleshooting facility that:

1. works out of the box as a first-class feature of the platform.
2. does not depend on tools having already been included in container images.
3. does not require administrative access to the node. (Administrative access
   via the Kubernetes API is acceptable.)

New `kubectl` concepts increase cognitive burden for all users of Kubernetes.
This KEP seeks to minimize this burden by mirroring the existing `kubectl`
workflows where possible.

### Non-Goals

Ephemeral containers are supported on Windows, but it's not the recommended
debugging facility. The other troubleshooting workflows described here are
equally as useful to Windows containers. We will not attempt to create
separate debugging facilities for Windows containers.

[distroless container image]: https://github.com/GoogleCloudPlatform/distroless
[distroless for k8s system images]: https://git.k8s.io/enhancements/keps/sig-release/20190316-rebase-images-to-distroless.md
[scratch debugger]: https://github.com/kubernetes-retired/contrib/tree/master/scratch-debugger

## Proposal

### Pod Troubleshooting with Ephemeral Debug Container

We will add a new command `kubectl debug` that will:

1. Construct a `v1.Container` for the debug container based on command line
   arguments. This will include an optional container for namespace targeting
   (described below).
2. Fetch the specified pod's existing ephemeral containers using
   `GetEphemeralContainers()` in the generated pod client.
3. Append the new debug container to the pod's ephemeral containers and call
   `UpdateEphemeralContainers()`.
4. Watch pod for updates to the debug container's `ContainerStatus` and
   automatically attach once the container is running. *(optional based on
   command line flag)*

We will attempt to make `kubectl debug` useful with a minimal of arguments
by using reasonable defaults where possible.

#### Debug Container Naming

Currently, there is no support for deleting or recreating ephemeral containers.
In cases where the user does not specify a name, `kubectl` should generate a
unique name and display it to the user.

#### Container Namespace Targeting

In order to see processes running in other containers in the pod, [Process
Namespace Sharing] should be enabled for the pod. In cases where process
namespace sharing isn't enabled for the pod, `kubectl` will set
`TargetContainer` in the `EphemeralContainer`. This will cause the ephemeral
container to be created in the namespaces of the target container in runtimes
that support container namespace targeting.

[Process Namespace Sharing]: https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace

#### Interactive Troubleshooting and Automatic Attaching

Since the primary use case for `kubectl debug` is interactive troubleshooting,
`kubectl debug` will automatically attach to the console of the newly created
ephemeral container and will default to creating containers with `Stdin` and
`TTY` enabled.

These may be disabled via command line flags.

#### Proposed Ephemeral Debug Arguments

```
% kubectl help debug
Execute a container in a pod.

Examples:
  # Start an interactive debugging session with a debian image
  kubectl debug mypod --image=debian

  # Run a debugging session in the same namespace as target container 'myapp'
  # (Useful for debugging other containers when shareProcessNamespace is false)
  kubectl debug mypod --target=myapp

Options:
  -a, --attach=true: Automatically attach to container once created
  -c, --container='': Container name.
  -i, --stdin=true: Pass stdin to the container
  --image='': Required. Container image to use for debug container.
  --target='': Target processes in this container name.
  -t, --tty=true: Stdin is a TTY

Usage:
  kubectl debug (POD | TYPE/NAME) [-c CONTAINER] [flags] -- COMMAND [args...] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```

### Pod Troubleshooting by Copy

Pod troubleshooting via Ephemeral Container relies on an alpha feature which is
unlikely to be enabled on production clusters. In order to support these
clusters, and because it would be generally useful, we will support a mode of
Pod troubleshooting that behaves similar to Pod Troubleshooting with Ephemeral
Debug Container but operates instead on a copy of the target pod.

The following additional options will cause a copy of the target pod to be
created:

```
Options:
  --copy-to='': Create a copy of the target Pod with this name.
  --copy-labels=false: When used with `--copy-to`, specifies whether labels
                       should also be copied. Note that copying labels may cause
                       the copy to receive traffic from a service or a replicaset
                       to kill other Pods.
  --delete-old=false: When used with `--copy-to`, delete the original Pod.
  --edit=false: Open an editor to modify the generated Pod prior to creation.
  --same-node=false: Schedule the copy of target Pod on the same node.
  --share-processes=true: When used with `--copy-to`, enable process namespace
                          sharing in the copy.
```

The modification `kubectl debug` makes to `Pod.Spec.Containers` depends on the
value of the `--container` flag.

#### Creating a Debug Container by copy

If a user does not specify a `--container` or specifies one that does not exist,
then the user is instructing `kubectl debug` to create a new Debug Container in
the Pod copy.

```
Examples:
  # Create a copy of 'mypod' with a new debugging container and attach to it
  kubectl debug mypod --copy-to=mypod-debug --image=debian --attach -- bash
```

#### Modify Application Image by Copy

If a user specifies a `--container`, then they are instructing `kubectl debug` to
create a copy of the target pod with a new image for one of the containers.

```
Examples:
  # Create a copy of 'mypod' with the debugging image for container 'app'
  kubectl debug mypod --copy-to=mypod-debug --image=myapp-image:debug --container=myapp -- myapp --debug=5
```

Note that the Pod API allows updates of container images in-place, so
`--copy-to` is not necessary for this operation. `kubectl debug` isn't necessary
to achieve this -- it can be done today with patch -- but `kubectl debug` could
implement it as well for completeness.

### Node Troubleshooting with Privileged Containers

When invoked with a node as a target, `kubectl debug` will create a new
pod with the following fields set:

* `nodeName: $target_node`
* `hostIPC: true`
* `hostNetwork: true`
* `hostPID: true`
* `restartPolicy: Never`

Additionally, `/` on the node will be mounted as a HostPath volume.

```
Examples:
  # Start an interactive debugging session on mynode with a debian image
  kubectl debug node/mynode --image=debian

Options:
  -a, --attach=true: Automatically attach to container once created
  -c, --container='': Container name.
  -i, --stdin=true: Pass stdin to the container
  --image='': Required. Container image to use for debug container.
  -t, --tty=true: Stdin is a TTY
```

### User Stories

#### Operations

Alice runs a service "neato" that consists of a statically compiled Go binary
running in a minimal container image. One of its pods is suddenly having
trouble connecting to an internal service. Being in operations, Alice wants to
be able to inspect the running pod without restarting it, but she doesn't
necessarily need to enter the container itself. She wants to:

1.  Inspect the filesystem of target container
1.  Execute debugging utilities not included in the container image
1.  Initiate network requests from the pod network namespace

This is achieved by running a new "debug" container in the pod namespaces. Her
troubleshooting session might resemble:

```
% kubectl debug -it --image debian neato-5thn0 -- bash
root@debug-image:~# ps x
  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 /pause
   13 ?        Ss     0:00 bash
   26 ?        Ss+    0:00 /neato
  107 ?        R+     0:00 ps x
root@debug-image:~# cat /proc/26/root/etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.155.240.10
options ndots:5
root@debug-image:~# dig @10.155.240.10 neato.svc.cluster.local.

; <<>> DiG 9.9.5-9+deb8u6-Debian <<>> @10.155.240.10 neato.svc.cluster.local.
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
```

Alice discovers that the cluster's DNS service isn't responding.

#### Debugging

Bob is debugging a tricky issue that's difficult to reproduce. He can't
reproduce the issue with the debug build, so he attaches a debug container to
one of the pods exhibiting the problem:

```
% kubectl debug -it --image=gcr.io/neato/debugger neato-5x9k3 -- sh
Defaulting container name to debug.
/ # ps x
PID   USER     TIME   COMMAND
    1 root       0:00 /pause
   13 root       0:00 /neato
   26 root       0:00 sh
   32 root       0:00 ps x
/ # gdb -p 13
...
```

He discovers that he needs access to the actual container, which he can achieve
by installing busybox into the target container:

```
root@debug-image:~# cp /bin/busybox /proc/13/root
root@debug-image:~# nsenter -t 13 -m -u -p -n -r /busybox sh


BusyBox v1.22.1 (Debian 1:1.22.0-9+deb8u1) built-in shell (ash)
Enter 'help' for a list of built-in commands.

/ # ls -l /neato
-rwxr-xr-x    2 0        0           746888 May  4  2016 /neato
```

Note that running the commands referenced above requires `CAP_SYS_ADMIN` and
`CAP_SYS_PTRACE`.

This scenario also requires process namespace sharing which is not available
on Windows.

#### Automation

Carol is a security engineer tasked with running security audits across all of
her company's running containers. Even though her company has no standard base
image, she's able to audit all containers using:

```
% for pod in $(kubectl get -o name pod); do
    kubectl debug --image gcr.io/neato/security-audit -p $pod /security-audit.sh
  done
```

#### Technical Support

Dan's team provides support for his company's multi-tenant cluster. He can
access the Kubernetes API (as a viewer) on behalf of the users he's supporting,
but he does not have administrative access to nodes or a say in how the
application image is constructed. When someone asks for help, Dan's first step
is to run his team's autodiagnose script:

```
% kubectl debug --image=k8s.gcr.io/autodiagnose nginx-pod-1234
```

### Implementation Details/Notes/Constraints

1.  There's an output stream race inherent to creating then attaching a
    container which causes output generated between the start and attach to go
    to the log rather than the client. This is not specific to Ephemeral
    Containers and exists because Kubernetes has no mechanism to attach a
    container prior to starting it. This larger issue will not be addressed by
    Ephemeral Containers, but Ephemeral Containers would benefit from future
    improvements or work arounds.

### Risks and Mitigations

1.  There are no guaranteed resources for ad-hoc troubleshooting. If
    troubleshooting causes a pod to exceed its resource limit it may be evicted.
    This risk can be removed once support for pod resizing has been implemented.

## Design Details

### Test Plan

In addition to standard unit tests for `kubectl`, the `debug` command will be
released as a `kubectl alpha` subcommand, signaling users to expect instability.
During the alpha phase we will gather feedback from users that we expect will
improve the design of `debug` and identify the Critical User Journeys we should
test prior to Alpha -> Beta graduation.

### Graduation Criteria

#### Alpha -> Beta Graduation

- [ ] Ephemeral Containers API has graduated to Beta
- [ ] A task on https://kubernetes.io/docs/tasks/ describes how to troubleshoot
  a running pod using Ephemeral Containers.
- [ ] A survey sent to early adopters doesn't reveal any major shortcomings.
- [ ] Test plan is amended to address the most common user journeys.

#### Beta -> GA Graduation

- [ ] Ephemeral Containers are GA

## Implementation History

- *2019-08-06*: Initial KEP draft
- *2019-12-05*: Updated KEP for expanded debug targets.
- *2020-01-09*: Updated KEP for debugging nodes and mark implementable.
- *2020-01-15*: Added test plan.

## Alternatives

An exhaustive list of alternatives to ephemeral containers is included in the
[Ephemeral Containers KEP].

[Ephemeral Containers KEP]: https://git.k8s.io/enhancements/keps/sig-node/20190212-ephemeral-containers.md
