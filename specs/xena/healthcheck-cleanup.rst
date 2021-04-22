..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Cleaning container healthchecks
===============================

https://blueprints.launchpad.net/tripleo/+spec/clean-container-healthchecks

We don't rely on the `container healthcheck`_ results for anything in the
infrastructure. They are time and resource consuming, and their maintenance is
mostly random. We can at least remove the ones that aren't hitting an actual
API healthcheck endpoint.

This proposal was discussed during a `session at the Xena PTG`_

Problem Description
===================

Since we moved the services to container, first with the docker engine, then
with podman, container healthchecks have been implemented and used.

While the very idea of healthchecks isn't bad, the way we (TripleO) are
making and using them is mostly wrong:

* no action is taken upon healthcheck failure
* some (most) aren't actually checking if the service is working, but merely
  that the service container is running

The healthchecks such as `healthcheck_port`_, `healthcheck_listen`_,
`healthcheck_socket`_ as well as most of the scripts calling
`healthcheck_curl`_ are mostly NOT doing anything more than ensuring a
service is running - and we already have this info when the container is
"running" (good), "restarting" (not good) or "exited" (with a non-0 code
- bad).

Also, the way podman implements healthchecks is relying on systemd and its
transient service and `timers`_. Basically, for each container, a new systemd
unit is created and injected, as well as a new timer - meaning systemd calls
podman. This isn't really good for the hosts, especially the ones having
heavy load due to their usage.

Proposed Change
===============

Overview
--------

A deep cleaning of the current healthcheck is needed, such as the
`healthcheck_socket`_, `healthcheck_port`_, and `healthcheck_curl`_
that aren't calling an actual API healthcheck endpoint. This list isn't
exhaustive.

This will drastically reduce the amount of "podman" calls, leading
to less resource issues, and provide a better comprehension when we list
the processes or services.

In case an Operator wants to get some status information, they can leverage
an existing validation::

  openstack tripleo validator run --validation service-status

This validation can be launched from the Undercloud directly, and will gather
remote status for every OC nodes, then provide a clear summary.

Such a validation could also be launched from a third-party monitoring
instance, provided it has the needed info (mostly the inventory).

Alternatives
------------

There are multiple alternatives we can even implement as a step-by-step
solution, though any of them would more than probably require their own
specifications and discussions:

Replace the listed healthchecks by actual service healthchecks
..............................................................

Doing so would allow to get a better understanding of the stack health, but
will not solve the issue with podman calls (hence resource eating and related
things).
Such healchecks can be launched from an external tool, for instance based
on a host's cron, or an actual service.

Call the healthchecks from an external tool
...........................................

Doing so would prevent the potential resource issues with the "podman exec"
calls we're currently seeing, while allowing a centralization for the results,
providing a better way to get metrics and stats.

Keep things as-is
.................

Because we have to list this one, but there are hints this isn't the right
thing to do (hence the current spec).

Security Impact
---------------

No real Security impact. Less services/calls might lead to smaller attack
surface, and it might prevent some *denial of service* situations.

Upgrade Impact
--------------

No Upgrade impact.

Other End User Impact
---------------------

The End User doesn't have access to the healthcheck anyway - that's more for
the operator.

Performance Impact
------------------

The systems will be less stressed, and this can improve the current situation
regarding performances and stability.

Other Deployer Impact
---------------------

There is no "deployer impact" if we don't consider they are the operator.

For the latter, there's a direct impact: ``podman ps`` won't be able to show
the health status anymore or, at least, not for the containers without such
checks.

But the operator is able to leverage the service-status validation instead -
this validation will even provide more information since it takes into account
the failed containers, a thing ``podman ps`` doesn't show without the proper
option, and even with it, it's not that easy to filter.

Developer Impact
----------------

In order to improve the healthchecks, especially for the API endpoints, service
developers will need to implement specific tests in the app.

Once it's existing, working and reliable, they can push it to any healthcheck
tooling at disposition - being the embedded container healthcheck, or some
dedicated service as described in the third step.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  cjeanner

Work Items
----------

#. Triage existing healthcheck, and if they aren't calling actual endpoint,
   deactive them in tripleo-heat-templates
#. Ensure the stack stability isn't degraded by this change, and properly
   document the "service-status" validation with the Validation Framework Team

The second work item is more an empirical data on the long term - we currently
don't have actual data, appart a `Launchpad issue`_ pointing to a problem
maybe caused by the way healthchecks are launched.

Possible future work items
..........................

#. Initiate a discussion with CloudOps (metrics team) regarding an dedicated
   healthcheck service, and how to integrate it properly within TripleO
#. Initiate a cross-Team work toward actual healthcheck endpoints for the
   services in need

Those are just here for the sake of evolution. Proper specs will be needed
in order to frame the work.

Dependencies
============

For step 1 and 2, no real dependencies are needed.

Testing
=======

Testing will require different things:

* Proper metrics in order to ensure there's no negative impact - and that any
  impact is measurable
* Proper insurance the removal of the healthcheck doesn't affect the services
  in a negative way
* Proper testing of the validations, especially "service-status" in order to
  ensure it's reliable enough to be considered as a replacement at some point

Documentation Impact
====================

A documentation update will be needed regarding the overall healthcheck topic.

References
==========

* `Podman Healthcheck implementation and usage`_


.. _container healthcheck: https://opendev.org/openstack/tripleo-common/src/branch/master/healthcheck
.. _healthcheck_port: https://opendev.org/openstack/tripleo-common/src/commit/a072a7f07ea75933a2372b1a95ae960095a3250e/healthcheck/common.sh#L49
.. _healthcheck_listen: https://opendev.org/openstack/tripleo-common/src/commit/a072a7f07ea75933a2372b1a95ae960095a3250e/healthcheck/common.sh#L85
.. _healthcheck_socket: https://opendev.org/openstack/tripleo-common/src/commit/a072a7f07ea75933a2372b1a95ae960095a3250e/healthcheck/common.sh#L95
.. _healthcheck_curl: https://opendev.org/openstack/tripleo-common/src/commit/a072a7f07ea75933a2372b1a95ae960095a3250e/healthcheck/common.sh#L28
.. _session at the Xena PTG: https://etherpad.opendev.org/p/tripleo-xena-drop-healthchecks
.. _timers: https://www.freedesktop.org/software/systemd/man/systemd.timer.html
.. _Podman Healthcheck implementation and usage: https://developers.redhat.com/blog/2019/04/18/monitoring-container-vitality-and-availability-with-podman/
.. _Launchpad issue: https://bugs.launchpad.net/tripleo/+bug/1923607
