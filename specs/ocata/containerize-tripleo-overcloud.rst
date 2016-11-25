===============================
Deploying TripleO in Containers
===============================

https://blueprints.launchpad.net/tripleo/+spec/containerize-tripleo

Ability to deploy TripleO in Containers.

Problem Description
===================

Linux containers are changing how the industry deploys applications by offering
a lightweight, portable and upgradeable alternative to deployments on a physical
host or virtual machine.

Since TripleO already manages OpenStack infrastructure by using OpenStack
itself, containers could be a new approach to deploy OpenStack services. It
would change the deployment workflow but could extend upgrade capabilities,
orchestration, and security management.

Benefits of containerizing the openstack services include:

    * Upgrades can be performed by swapping out containers.
    * Since the entire software stack is held within the container,
      interdependencies do not affect deployments of services.
    * Containers define explicit state and data requirements. Ultimately if we
      moved to kubernetes all volumes would be off the host making the host
      stateless.
    * Easy rollback to working containers if upgrading fails.
    * Software shipped in each container has been proven to work for this service.
    * Mix and match versions of services on the same host.
    * Immutable containers provide a consistent environment upon startup.

Proposed Change
===============

Overview
--------

The intention of this blueprint is to introduce containers as a method of
delivering an OpenStack installation. We currently have a fully functioning
containerized version of the compute node but we would like to extend this to
all services. In addition it should work with the new composable roles work that
has been recently added.

The idea is to have an interface within the heat templates that adds information
for each service to be started as a container. This container format should
closely resemble the Kubernetes definition so we can possibly transition to
Kubernetes in the future. This work has already been started here:

    https://review.openstack.org/#/c/330659/

There are some technology choices that have been made to keep things usable and
practical. These include:

    * Using Kolla containers. Kolla containers are built using the most popular
      operating system choices including CentOS, RHEL, Ubuntu, etc. and are a
      good fit for our use case.
    * We are using a heat hook to start these containers directly via docker.
      This minimizes the software required on the node and maps directly to the
      current baremetal implementation. Also maintaining the heat interface
      keeps the GUI functional and allows heat to drive upgrades and changes to
      containers.
    * Changing the format of container deployment to match Kubernetes for
      potential future use of this technology.
    * Using CentOS in full (not CentOS Atomic) on the nodes to allow users to
      have a usable system for debugging.
    * Puppet driven configuration that is mounted into the container at startup.
      This allows us to retain our puppet configuration system and operate in
      parallel with existing baremetal deployment.

Bootstrapping
-------------

Once the node is up and running, there is a systemd service script that runs
which starts the docker agents container. This container has all of the
components needed to bootstrap the system. This includes:

    * heat agents including os-collect-config, os-apply-config etc.
    * puppet-agent and modules needed for the configuration of the deployment.
    * docker client that connects to host docker daemon.
    * environment for configuring networking on the host.

This containers acts as a self-installing container. Once started, this
container will use os-collect-config to connect back to heat. The heat agents
then perform the following tasks:

    * Set up an etc directory and runs puppet configuration scripts. This
      generates all the config files needed by the services in the same manner
      it would if run without containers. These are copied into a directory
      accessible on the host and by all containerized services.
    * Begin starting containerized services and other steps as defined in the
      heat template.

Currently all containers are implemented using net=host to allow the services to
listen directly on the host network(s). This maintains functionality in terms of
network isolation and IPv6.

Security Impact
---------------

There shouldn't be major security impacts from this change. The deployment
shouldn't be affected negatively by this change from a security standpoint but
unknown issues might be found. SELinux support is implemented in Docker.

End User Impact
---------------

* Debugging of containerized services will be different as it will require
  knowledge about docker (kubernetes in the future) and other tools to access
  the information from the containers.
* Possibly provide more options for upgrades and new versions of services.
* It'll allow for service isolation and better dependency management

Performance Impact
------------------

Very little impact:

    * Runtime performance should remain the same.
    * We are noticing a slightly longer bootstrapping time with containers but that
      should be fixable with a few easy optimizations.

Deployer Impact
---------------

From a deployment perspective very little changes:
    * Deployment workflow remains the same.
    * There may be more options for versions of different services since we do
      not need to worry about interdependency issues with the software stack.

Upgrade Impact
--------------

This work aims to allow for resilent, transparent upgrades from baremetal
overcloud deployments to container based ones.

Initially we need to transition to containers:
    * Would require node reboots.
    * Automated upgrades should be possible as services are the same, just
      containerized.
    * Some state may be moved off nodes to centralized storage. Containers very
      clearly define required data and state storage requirements.

Upgrades could be made easier:
    * Individual services can be upgraded because of reduced interdependencies.
    * It is easier to roll back to a previous version of a service.
    * Explicit storage of data and state for containers makes it very clear what
      needs to be preserved. Ultimately state information and data will likely
      not exist on individual nodes.

Developer Impact
----------------

The developer work flow changes slighly. Instead of interacting with the service
via systemd and log files, you will interact with the service via docker.

Inside the compute node:
    * sudo docker ps -a
    * sudo docker logs <container-name>
    * sudo docker exec -it <container-name> /bin/bash

Implementation
==============

Assignee(s)
  rhallisey
  imain
  flaper87
  mandre

Other contributors:
  dprince
  emilienm

Work Items
----------

* Heat Docker hook that starts containers (DONE)
* Containerized Compute (DONE)
* TripleO CI job (INCOMPLETE - https://review.openstack.org/#/c/288915/)
* Containerized Controller
* Automatically build containers for OpenStack services
* Containerized Undercloud

Dependencies
============

* Composable roles.
* Heat template interface which allows extensions to support containerized
  service definitions.

Testing
=======
TripleO CI would need a new Jenkins job that will deploy an overcloud in
containers by using the selected solution.

Documentation Impact
====================
https://github.com/openstack/tripleo-heat-templates/blob/master/docker/README-containers.md

* Deploying TripleO in containers
* Debugging TripleO containers

References
==========
* https://docs.docker.com/misc/
* https://etherpad.openstack.org/p/tripleo-docker-puppet
* https://docs.docker.com/articles/security/
* http://docs.openstack.org/developer/kolla/
* https://review.openstack.org/#/c/209505/
* https://review.openstack.org/#/c/227295/
