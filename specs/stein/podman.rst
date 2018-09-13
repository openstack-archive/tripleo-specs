..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Podman support for container management
=======================================

Launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/podman-support

There is an ongoing desire to manage TripleO containers with a set of tools
designed to solve complex problems when deploying applications.
The containerization of TripleO started with a Docker CLI implementation
but we are looking at how we could leverage the container orchestration
on a Kubernetes friendly solution.


Problem Description
===================

There are three problems that this document will cover:

* There is an ongoing discussion on whether or not Docker will be
  maintained on future versions of Red Hat platforms. There is a general
  move on OCI (Open Containers Initiative) conformant runtimes, as CRI-O
  (Container Runtime Interface for OCI).

* The TripleO community has been looking at how we could orchestrate the
  containers lifecycle with Kubernetes, in order to bring consistency with
  other projects like OpenShift for example.

* The TripleO project aims to work on the next version of Red Hat platforms,
  therefore we are looking at Docker alternatives in Stein cycle.


Proposed Change
===============

Introduction
------------

The containerization of TripleO has been an ongoing effort since a few releases
now and we've always been looking at a step-by-step approach that tries to
maintain backward compatibility for the deployers and developers; and also
in a way where upgrade from a previous release is possible, without too much
pain. With that said, we are looking at a proposed change that isn't too much
disruptive but is still aligned with the general roadmap of the container
story and hopefully will drive us to manage our containers with Kubernetes.
We use Paunch project to provide an abstraction in our container integration.
Paunch will deal with container configurations formats with backends support.

Integrate Podman CLI
--------------------

The goal of Podman is to allow users to run standalone (non-orchestrated)
containers which is what we have been doing with Docker until now.
Podman also allows users to run groups of containers called Pods where a Pod is
a term developed for the Kubernetes Project which describes an object that
has one or more containerized processes sharing multiple namespaces
(Network, IPC and optionally PID).
Podman doesn't have any daemon which makes it lighter than Docker and use a
more traditional fork/exec model of Unix and Linux.
The container runtime used by Podman is runc.
The CLI has a partial backward compatibility with Docker so its integration
in TripleO shouldn't be that painful.

It is proposed to add support for Podman CLI (beside Docker CLI) in TripleO
to manage the creation, deletion, inspection of our containers.
We would have a new parameter called ContainerCli in TripleO, that if set to
'podman', will make the container provisionning done with Podman CLI and not
Docker CLI.

Because there is no daemon, there are some problems that we needs to solve:

* Automatically restart failed containers.
* Automatically start containers when the host is (re)booted.
* Start the containers in a specific order during host boot.
* Provide an channel of communication with containers.
* Run container healthchecks.

To solve the first 3 problems, it is proposed to use Systemd:

* Use Restart so we can configure a restart policy for our containers.
  Most of our containers would run with Restart=always policy, but we'll
  have to support some exceptions.
* The systemd services will be enabled by default so the containers start
  at boot.
* The ordering will be managed by Wants which provides Implicit Dependencies
  in Systemd. Wants is a weaker version of Requires. It'll allow to make sure
  we start HAproxy before Keepalived for example, if they are on the same host.
  Because it is a weak dependency, they will only be honored if the containers
  are running on the same host.
* The way containers will be managed (start/stop/restart/status) will be
  familiar for our operators used to control Systemd services. However
  we probably want to make it clear that this is not our long term goal to
  manage the containers with Systemd.

The Systemd integration would be:

* complete enough to cover our use-cases and bring feature parity with the
  Docker implementation.
* light enough to be able to migrate our container lifecycle with Kubernetes
  in the future (e.g. CRI-O).


For the fourth problem, we are still investigating the options:

* varlink: interface description format and protocol that aims to make services
  accessible to both humans and machines in the simplest feasible way.
* CRI-O: CI-based implementation of Kubernetes Container Runtime Interface
  without Kubelet. For example, we could use a CRI-O Python binding to
  communicate with the containers.
* A dedicated image which runs the rootwrap daemon, with rootwrap filters to only run the allowed
  commands.  The controlling container will have the rootwrap socket mounted in so that it can
  trigger allowed calls in the rootwrap container.  For pacemaker, the rootwrap container will allow
  image tagging. For neutron, the rootwrap container will spawn the processes inside the container,
  so it will need to be a long-lived container that is managed outside paunch.

             +---------+     +----------+
             |         |     |          |
             | L3Agent +-----+ Rootwrap |
             |         |     |          |
             +---------+     +----------+

  In this example, the L3Agent container has mounted in the rootwrap daemon socket so that it can
  run allowed commands inside the rootwrap container.

Finally, the fifth problem is still an ongoing question.
There are some plans to support healthchecks in Podman but nothing has been
done as of today. We might have to implement something on our side with
Systemd.

Alternatives
============

Two alternatives are proposed.

CRI-O Integration
-----------------

CRI-O is meant to provide an integration path between OCI conformant runtimes
and the kubelet. Specifically, it implements the Kubelet Container Runtime
Interface (CRI) using OCI conformant runtimes. Note that the CLI utility for
interacting with CRI-O isn't meant to be used in production, so managing
the containers lifecycle with a CLI is only possible with Docker or Podman.

So instead of a smooth migration from Docker CLI to Podman CLI, we could go
straight to Kubernetes integration and convert our TripleO services to work
with a standalone Kubelet managed by CRI-O.
We would have to generate YAML files for each container in a Pod format,
so CRI-O can manage them.
It wouldn't require Systemd integration, as the containers will be managed
by Kubelet.
The operator would control the container lifecycle by using kubectl commands
and the automated deployment & upgrade process would happen in Paunch with
a Kubelet backend.

While this implementation will help us to move to a multi-node Kubernetes
friendly environment, it remains the most risky option in term of the
quantity of work that needs to happen versus the time that we have to design,
implement, test and ship the next tooling before the end of Stein cycle.

We also need to keep in mind that CRI-O and Podman share containers/storage
and containers/image libraries, so the issues that we have had with Podman
will be hit with CRI-O as well.

Keep Docker
-----------

We could keep Docker around and do not change anything in the way we manage
containers. We could also keep Docker and make it work with CRI-O.
The only risk here is that Docker tooling might not be supported in the future
by Red Hat platforms and we would be on our own if any issue with Docker.
The TripleO community is always seeking for an healthy and long term
collaboration between us and the projects communities that we are interracting
with.

Proposed roadmap
================

In Stein:

* Make Paunch support Podman as an alternative to Docker.
* Get our existing services fully deployable on Podman, with parity to
  what we had with Docker.
* If we have time, add Podman pod support to Paunch

In "T" cycle:

* Rewrite all of our container yaml to the pod format.
* Add a Kubelet backend to Paunch (or change our agent tooling to call
  Kubelet directly from Ansible).
* Get our existing service fully deployable via Kublet, with parity to
  what we had with Podman / Docker.
* Evaluate switching to Kubernetes proper.


Security Impact
===============

The TripleO containers will rely on Podman security.
If we don't use CRI-O or varlink to communicate with containers, we'll have
to consider running some containers in privileged mode and mount
/var/lib/containers into the containers. This is a security concern and
we'll have to evaluate it.
Also, we'll have to make the proposed solution with SELinux in Enforcing mode.

Docker solution doesn't enforce selinux separation between containers.
Podman does, and there's currently no easy way to deactivate that globally.
So we'll basically get a more secure containers with Podman, as we have to
support separation from the very beginning.

Upgrade Impact
==============

The containers that were managed by Docker Engine will be removed and
provisioned into the new runtime. This process will happen when Paunch
generates and execute the new container configuration.
The operator shouldn't have to do any manual action and the migration will be
automated, mainly by Paunch.
The Containerized Undercloud upgrade job will test the upgrade of an Undercloud
running Docker containers on Rocky and upgrade to Podman containers on Stein.
The Overcloud upgrade jobs will also test.

Note: as the docker runtime doesn't have the selinux separation,
some chcon/relabelling might be needed prior the move to podman runtime.

End User Impact
===============

The operators won't be able to run Docker CLI like before and instead will
have to use Podman CLI, where some backward compatibility is garanteed.

Performance Impact
==================

There are different aspects of performances that we'll need to investigate:

* Container performances (relying on Podman).
* How Systemd + Podman work together and how restart work versus Docker engine.

Deployer Impact
===============

There shouldn't be much impact for the deployer, as we aim to make this change
the most transparent as possible. The only option (so far) that will be
exposed to the deployer will be "ContainerCli", where only 'docker' and
'podman' will be supported. If 'podman' is choosen, the transition will be
automated.

Developer Impact
================

There shouldn't be much impact for the developer of TripleO services, except
that there are some things in Podman that slightly changed when comparing
with Docker. For example Podman won't create the missing directories when
doing bind-mount into the containers, while Docker create them.

Implementation
==============

Contributors
------------

* Bogdan Dobrelya
* Cédric Jeanneret
* Emilien Macchi
* Steve Baker

Work Items
----------

* Update TripleO services to work with Podman (e.g. fix bind-mounts issues).
* SELinux separation (relates to bind-mounts rights + some other issues when
  we're calling iptables/other host command from a containe)
* Systemd integration.
* Healthcheck support.
* Socket / runtime: varlink? CRI-O?
* Upgrade workflow.
* Testing.
* Documentation for operators.


Dependencies
============

* The Podman integration depends a lot on how stable is the tool and how
  often it is released and shipped so we can test it in CI.
* The Healthchecks interface depends on Podman's roadmap.

Testing
=======

First of all, we'll switch the Undercloud jobs to use Podman and this work
should be done by milestone-1. Both the deployment and upgrade jobs should
be switched and actually working.
The overcloud jobs should be switched by milestone-2.

We'll keep Docker testing support until we keep testing running on CentOS7
platform.

Documentation Impact
====================

We'll need to document the new commands (mainly the same as Docker), and
the differences of how containers should be managed (Systemd instead of Docker
CLI for example).


References
==========

* https://www.projectatomic.io/blog/2018/02/reintroduction-podman/
* https://github.com/kubernetes-sigs/cri-o
* https://github.com/kubernetes/community/blob/master/contributors/devel/container-runtime-interface.md
* https://varlink.org/
* https://github.com/containers/libpod/blob/master/transfer.md
* https://etherpad.openstack.org/p/tripleo-standalone-kubelet-poc
