===========================================
Container Healthchecks for TripleO Services
===========================================

https://blueprints.launchpad.net/tripleo/+spec/container-healthchecks

An OpenStack deployment involves many services spread across many
hosts. It is important that we provide tooling and APIs that make it
as easy as possible to monitor this large, distributed environment.
The move to containerized services in the overcloud [1]
brings with it many opportunities, such as the ability to bundle
services with their associated health checks and provide a standard
API for assessing the health of the service.

[1]: https://blueprints.launchpad.net/tripleo/+spec/containerize-tripleo

Problem Description
===================

The people who are in the best position to develop appropriate health
checks for a service are generally those people responsible for
developing the service.  Unfortunately, the task of setting up
monitoring generally ends up in the hands of cloud operators or some
intermediary.

I propose that we take advantage of the bundling offered by
containerized services and create a standard API with which an
operator can assess the health of a service.  This makes life easier
for the operator, who can now provide granular service monitoring
without requiring detailed knowledge about every service, and it
allows service developers to ensure that services are monitored
appropriately.

Proposed Change
===============

Overview
--------

The Docker engine (since version 1.12), as well as most higher-level
orchestration frameworks, provide a standard mechanism for validating
the health of a container.  Docker itself provides the
HEALTHCHECK_ directive, while Kubernetes has explicit
support for `liveness and readiness probes`_.  Both
mechanisms work by executing a defined command inside the container,
and using the result of that executing to determine whether or not the
container is "healthy".

.. _liveness and readiness probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
.. _healthcheck: https://docs.docker.com/engine/reference/builder/#healthcheck

I propose that we explicitly support these interfaces in containerized
TripleO services through the following means:

1. Include in every container a `/openstack/healthcheck` command that
   will check the health of the containerized service, exit with
   status ``0`` if the service is healthy or ``1`` if not, and provide
   a message on ``stdout`` describing the nature of the error.

2. Include in every Docker image an appropriate ``HEALTHCHECK``
   directive to utilize the script::

      HEALTHCHECK CMD /openstack/healthcheck

3. If Kubernetes becomes a standard part of the TripleO deployment
   process, we may be able to implement liveness or readiness probes
   using the same script::

      livenessProbe:
        exec:
          command:
            - /openstack/healthcheck

Alternatives
------------

The alternative is the status quo: services do not provide a standard
healthcheck API, and service monitoring must be configured
individually by cloud operators.

Security Impact
---------------

N/A

Other End User Impact
---------------------

Users can explicitly run the healthcheck script to immediately assess
the state of a service.

Performance Impact
------------------

This proposal will result in the periodic execution of tasks on the
overcloud hosts.  When designing health checks, service developers
should select appropriate check intervals such that there is minimal
operational overhead from the health checks.

Other Deployer Impact
---------------------

N/A

Developer Impact
----------------

Developers will need to determine how best to assess the health of a
service and provide the appropriate script to perform this check.

Implementation
==============

Assignee(s)
-----------

N/A

Work Items
----------

N/A

Dependencies
============

- This requires that we implement `containerize-tripleo-overcloud`_
  blueprint.

.. _containerize-tripleo-overcloud: https://specs.openstack.org/openstack/tripleo-specs/specs/ocata/containerize-tripleo-overcloud.html

Testing
=======

TripleO CI jobs should be updated to utilize the healthcheck API to
determine if services are running correctly.

Documentation Impact
====================

Any documentation describing the process of containerizing a service
for TripleoO must be updated to describe the healthcheck API.

References
==========

N/A

