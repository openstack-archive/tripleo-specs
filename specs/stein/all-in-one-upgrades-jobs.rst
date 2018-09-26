..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================
Improve upgrade_tasks CI coverage with the standalone installer
===============================================================

https://blueprints.launchpad.net/tripleo/+spec/upgrades-ci-standalone

The main goal of this work is to improve coverage of service upgrade_tasks in
tripleo ci upgrades jobs, by making use of the Standalone_installer_work_.
Using a standalone node as a single node 'overcloud' allows us to exercise
both controlplane and dataplane services in the same job and within current
resources of 2 nodes and 3 hours. Furthermore and once proven successful
this approach can be extended to include even single service upgrades testing
to vastly improve on the current coverage with respect to all the service
upgrade_tasks defined in the tripleo-heat-templates (which is currently minimal).

Traditionally upgrades jobs have been restricted by resource constraints
(nodes and walltime). For example the undercloud and overcloud upgrade are
never exercised in the same job, that is an overcloud upgrade job uses an undercloud that is already on the target version (so called mixed version deployment).

A further example is that upgrades jobs have typically exercised either
controlplane or dataplane upgrades (i.e. controllers only, or compute only)
and never both in the same job, again because constraints. The currently running
tripleo-ci-centos-7-scenario000-multinode-oooq-container-upgrades_ job for
example has 2 nodes, where one is undercloud and one is overcloud controller.
The workflow *is* being exercised, but controller only. Furthermore, whilst
the current_upgrade_ci_scenario_ is only exercising a small subset of the
controlplane services, it is still running at well over 140 minutes. So there
is also very little coverage with respect to the upgrades_tasks across the
many different service templates defined in the tripleo-heat-templates.

Thus the main goal of this work is to use the standalone installer to define
ci jobs that test the service upgrade_tasks for a one node 'overcloud' with
both controlplane and dataplane services. This approach is composable as the
services in the stand-alone are fully configurable. Thus after the first
iteration of compute/control, we can also define per-service ci jobs and over
time hopefully reach coverage for all the services deployable by TripleO.

Finally it is worth emphasising that the jobs defined as part of this work will not
be testing the TripleO upgrades *workflow* at all. Rather this is about testing
the service upgrades_tasks specifically. The workflow instead will be tested
using the existing ci upgrades job (tripleo-ci-centos-7-scenario000-multinode-oooq-container-upgrades_) subject to modifications to strip it down to a bare
minimum required (e.g. hardly any services). There are more pointers to this
from the discussion at the TripleO-Stein-PTG_ but ultimately we will have two
approximations of the upgrade tested in ci - the service upgrade_tasks as
described by this spec, and the workflow itself using a different ci job or
modifying the existing one.

.. _Standalone_installer_work: http://lists.openstack.org/pipermail/openstack-dev/2018-June/131135.html
.. _tripleo-ci-centos-7-scenario000-multinode-oooq-container-upgrades: https://github.com/openstack-infra/tripleo-ci/blob/4101a393f29c18a84f64cd95a28c41c8142c5b05/zuul.d/multinode-jobs.yaml#L384
.. _current_upgrade_ci_scenario: https://github.com/openstack/tripleo-heat-templates/blob/9f1d855627cf54d26ee540a18fc8898aaccdda51/ci/environments/scenario000-multinode-containers.yaml#L21
.. _TripleO-Stein-PTG: https://etherpad.openstack.org/p/tripleo-ptg-stein

Problem Description
===================

As described above we have not been able to have control and dataplane
services upgraded as part of the same tripleo ci job. Such a job would
have to be 3 nodes for starters (undercloud,controller,compute).

A *full* upgrade workflow would need the following steps:

  * deploy undercloud, deploy overcloud
  * upgrade undercloud
  * upgrade prepare the overcloud (heat stack update generates playbooks)
  * upgrade run controllers (ansible-playbook via mistral workflow)
  * upgrade run computes/storage etc (repeat until all done)
  * upgrade converge (heat stack update).

The problem being solved here is that we can run only some approximation of
the upgrade workflow, specifically the upgrade_tasks, for a composed set
of services and do so within the ci timeout. The first iteration will focus on
modelling a one node 'overcloud' with both controller and compute services. If
we prove this to be successful we can also consider single-service upgrades
jobs (a job for testing just nova,or glance upgrade tasks for example) for
each of services that we want to test the upgrades tasks. Thus even though
this is just an approximation of the upgrade (upgrade_tasks only, not the full
workflow), it can hopefully allow for a wider coverage of services in ci
than is presently possible.

One of the early considerations when writing this spec was how we could enforce
a separation of services with respect to the upgrade workflow. That is, enforce
that controlplane upgrade_tasks and deploy_steps are executed first and then
dataplane compute/storage/ceph as is usually the case with the upgrade workflow.
However review comments on this spec as well as PTG discussions around it, in
particular that this is just some approximation of the upgrade (service
upgrade tasks, not workflow) in which case it may not be necessary to artificially
induce this control/dataplane separation here. This may need to be revisited
once implementation begins.

Another core challenge that needs solving is how to collect ansible playbooks
from the tripleo-heat-templates since we don't have a traditional undercloud
heat stack to query. This will hopefully be a lesser challenge assuming we can
re-use the transient heat process used to deploy the standalone node. Futhermore
discussion around this point at the TripleO-Stein-PTG_ has informed us of a way
to keep the heat stack after deployment with keep-running_ so we could just
re-use it as we would with a 'normal' deployment.

Proposed Change
===============

Overview
--------

We will need to define a new ci job in the tripleo-ci_zuul.d_standalone-jobs_
(preferably following the currently ongoing ci_v3_migrations_ define this as
v3 job).

For the generation of the playbooks themselves we hope to use the ephemeral
heat service that is used to deploy the stand-alone node, or use the keep-running_
option to the stand-alone deployment to keep the stack around after deployment.

As described in the problem statement we hope to avoid the task of having to
distinguish between control and dataplane services in order to enforce that
controlplane services are upgraded first.

.. _tripleo-ci_zuul.d_standalone-jobs: https://github.com/openstack-infra/tripleo-ci/blob/4101a393f29c18a84f64cd95a28c41c8142c5b05/zuul.d/standalone-jobs.yaml
.. _ci_v3_migrations: https://review.openstack.org/#/c/578432/8
.. _keep-running: https://github.com/openstack/python-tripleoclient/blob/a57531382535e92e2bfd417cee4b10ac0443dfc8/tripleoclient/v1/tripleo_deploy.py#L911

Alternatives
------------

Add another node and have 3 node upgrades jobs together with increasing the
walltime but this is not scalable in the long term assuming limited
resources!


Security Impact
---------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

More coverage of services should mean less breakage because of upgrades
incompatible things being merged.

Developer Impact
----------------

Might be easier for developers too who may have limited access to resources
to take the reproducer script with the standalone jobs and get a dev env for
testing upgrades.

Implementation
==============

Assignee(s)
-----------

tripleo-ci and upgrades squads

Work Items
----------

First we must solve the problem of generating the ansible playbooks, that
will include all the latest configuration from the tripleo-heat-templates at
the time of upgrade (including all upgrade_tasks etc) when there is no
undercloud Heat stack to query.

We might consider some non-heat solution by parsing the tripleo-heat-templates
but I don't think that is a feasible solution (re-inventing wheels). There is
ongoing work to transfer tasks to roles which is promising and that is another
area to explore.

One obvious mechanism to explore given the current tools is to re-use the
same ephemeral heat process that the stand-alone uses in deploying the
overcloud, but setting the usual 'upgrade-init' environment files for a short
stack 'update'. This is not tested at all yet so needs to be investigated
further. As identified earlier there is now in fact a keep-running_ option to the
tripleoclient that will keep this heat process around

For the first iteration of this work we will aim to use the minimum possible combination
of services to implement a 'compute'/'control' overcloud. That is, using the existing
services from the current current_upgrade_ci_scenario_ with the addition of nova-compute
and any dependencies.

Finally a third major consideration is how to execute this service upgrade, that
is how to invoke the playbook generation and then run the resulting playbooks
(it probably doesn't need to converge if we are just interested in the upgrades
tasks). One consideration might be to re-use the existing python-tripleoclient
"openstack overcloud upgrade" prepare and run sub-commands. However the first
and currently favored approach will be to use the existing stand-alone client
commands (tripleo_upgrade_ tripleo_deploy_). So one work item is to try these
and discover any modifications we might need to make them work for us.

Items:
  * Work out/confirm generation the playbooks for the standalone upgrade tasks.
  * Work out any needed changes in the client/tools to execute the ansible playbooks
  * Define new ci job in the tripleo-ci_zuul.d_standalone-jobs_ with control and
    compute services, that will exercise upgrade_tasks, deployment_tasks and
    post_upgrade_tasks playbooks.

Once this first iteration is complete we can then consider defining multiple
jobs for small subsets of services, or even for single services.

.. _tripleo_upgrade: https://github.com/openstack/python-tripleoclient/blob/6b0f54c07ae8d0dd372f16684c863efa064079da/tripleoclient/v1/tripleo_upgrade.py#L33
.. _tripleo_deploy: https://github.com/openstack/python-tripleoclient/blob/6b0f54c07ae8d0dd372f16684c863efa064079da/tripleoclient/v1/tripleo_deploy.py#L80

Dependencies
============

This obviously depends on stand-alone installer

Testing
=======

There will be at least one new job defined here

Documentation Impact
====================

None

References
==========
