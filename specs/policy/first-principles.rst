..

========================
TripleO First Principles
========================

The TripleO first principles are a set of principles that guide decision making
around future direction with TripleO. The principles are used to evaluate
choices around changes in direction and architecture. Every impactful decision
does not necessarily have to follow all the principles, but we use them to make
informed decisions about trade offs when necessary.

Problem Description
===================

When evaluating technical direction within TripleO, a better and more
consistent method is needed to weigh pros and cons of choices. Defining the
principles is a step towards addressing that need.


Policy
======


Definitions
-----------

Framework
  The functional implementation which exposes a set of standard enforcing
  interfaces that can be consumed by a service to describe that service's
  deployment and management. The framework includes all functional pieces that
  implement such interfaces, such as CLI's, API's, or libraries.

  Example: tripleoclient/tripleo-common/tripleo-ansible/tripleo-heat-templates

Service
  The unit of deployment. A service will implement the necessary framework
  interfaces in order to describe it's deployment.

  The framework does not enforce a particular service boundary, other than by
  prescribing best practices. For example, a given service implementation could
  deploy both a REST API and a database, when in reality the API and database
  should more likely be deployed as their own services and expressed as
  dependencies.

  Example: Keystone, MariaDB, RabbitMQ

Third party integrations
  Service implementations that are developed and maintained outside of the
  TripleO project. These are often implemented by vendors aiming to add support
  for their products within TripleO.

  Example: Cinder drivers, Neutron plugins

First Principles
----------------

#. [UndercloudMigrate] No Undercloud Left Behind

   #. TripleO itself as the deployment tool can be upgraded. We do
      not immediately propose what the upgrade will look like or the technology
      stack, but we will offer an upgrade path or a migration path.

#. [OvercloudMigrate] No Overcloud Left Behind

   #. An overcloud deployed with TripleO can be upgraded to the next major version
      with either an in place upgrade or migration.

#. [DefinedInterfaces] TripleO will have a defined interface specification.

   #. We will document clear boundaries between internal and external
      (third party integrations) interfaces.
   #. We will document the supported interfaces of the framework in the same
      way that a code library or API would be documented.
   #. Individual services of the framework can be deployed and tested in
      isolation from other services. Service dependencies are expressed per
      service, but do not preclude using the framework to deploy a service
      isolated from its dependencies. Whether that is successful or not
      depends on how the service responds to missing dependencies, and that is
      a behavior of the service and not the framework.
   #. The interface will offer update and upgrade tasks as first class citizens
   #. The interface will offer validation tasks as first class citizens

#. [OSProvisioningSeparation] Separation between operating system provisioning
   and software configuration.

   #. Baremetal configuration, network configuration and base operating system
      provisioning is decoupled from the software deployment.
   #. The software deployment will have a defined set of minimal requirements
      which are expected to be in-place before it begins the software deployment.

      #. Specific linux distributions
      #. Specific linux distribution versions
      #. Password-less access via ssh
      #. Password-less sudo access
      #. Pre-configured network bridges

#. [PlatformAgnostic] Platform agnostic deployment tooling.

   #. TripleO is sufficiently isolated from the platform in a way that allows
      for use in a variety of environments (baremetal/virtual/containerized/OS
      version).
   #. The developer experience is such that it can easily be run in
      isolation on developer workstations

#. [DeploymentToolingScope] The deployment tool has a defined scope

   #. Data collection tool.

      #. Responsible for collecting host and state information and posting to a
         centralized repository.
      #. Handles writes to central repository (e.g. read information from
         repository, do aggregation, post to central repository)

   #. A configuration tool to configure software and services as part of the
      deployment

      #. Manages Software Configuration

         #. Files
         #. Directories
         #. Service (containerized or non-containerized) state
         #. Software packages

      #. Executes commands related to “configuration” of a service
         Example: Configure OpenStack AZ's, Neutron Networks.
      #. Isolated executions that are invoked independently by the orchestration tool
      #. Single execution state management

         #. Input is configuration data/tasks/etc
         #. A single execution produces the desired state or reports failure.
         #. Idempotent

      #. Read-only communication with centralized data repository for configuration data

   #. The deployment process depends on an orchestration tool to handle various
      task executions.

      #. Task graph manager
      #. Task transport and execution tracker
      #. Aware of hosts and work to be executed on the hosts
      #. Ephemeral deployment tooling
      #. Efficient execution
      #. Scale and reliability/durability are first class citizens

#. [CI/CDTooling] TripleO functionality should be considered within the context
   of being directly invoked as part of a CI/CD pipeline.

#. [DebuggableFramework] Diagnosis of deployment/configuration failures within
   the framework should be quick and simple. Interfaces should be provided to
   enable debuggability of service failures.

#. [BaseOSBootstrap] TripleO can start from a base OS and go to full cloud

   #. It should be able to start at any point after base OS, but should be able
      to handle the initial OS bootstrap

#. [PerServiceManagement] TripleO can manage individual services in isolation,
   and express and rely on dependencies and ordering between services.

#. [Predictable/Reproducible/Idempotent] The deployment is predictable

   #. The operator can determine what changes will occur before actually applying
      those changes.
   #. The deployment is reproducible in that the operator can re-run the
      deployment with the same set of inputs and achieve the same results across
      different environments.
   #. The deployment is idempotent in that the operator can re-run the
      deployment with the same set of inputs and the deployment will not change other
      than when it was first deployed.
   #. In the case where a service needs to restart a process, the framework
      will have an interface that the service can use to notify of the
      needed restart. In this way, the restarts are predictable.
   #. The interface for service restarts will allow for a service to describe
      how it should be restarted in terms of dependencies on other services,
      simultaneous restarts, or sequential restarts.

Non-principles
--------------

#. [ContainerImageManagement] The framework does not manage container images.
   Other than using a given container image to start a container, the framework
   does not encompass common container image management to include:

   #. Building container images
   #. Patching container images
   #. Serving or mirroring container images
   #. Caching container images

   Specific tools for container image and runtime management and that need to
   leverage the framework during deployment are expected to be implemented as
   services.

#. [SupportingTooling] Tools and software executed by the framework to deploy
   services or tools required prior to service deployment by the framework are
   not considered part of the framework itself.

   Examples: podman, TCIB, image-serve, nova-less/metalsmith

Alternatives & History
======================

Many, if not all, the principles are already well agreed upon and understood as
core to TripleO. Writing them down as policy makes them more discoverable and
official.

Historically, there have been instances when decisions have been guided by
desired technical implementation or outcomes. Recording the principles does not
necessarily mean those decisions would stop, but it does allow for a more
reasonable way to think about the trade offs.

We do not need to adopt any principles, or record them. However, there is no
harm in doing so.

Implementation
==============

Author(s)
---------

Primary author:
  James Slagle <jslagle@redhat.com>

Other contributors:
  <launchpad-id or None>

Milestones
----------

None.

Work Items
----------

None.

References
==========

None.

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - v0.0.1
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
