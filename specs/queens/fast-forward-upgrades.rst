..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Fast-forward upgrades
=====================

https://blueprints.launchpad.net/tripleo/+spec/fast-forward-upgrades

Fast-forward upgrades are upgrades that move an environment from release `N` to
`N+X` in a single step, where `X` is greater than `1` and for fast-forward
upgrades is typically `3`. This spec outlines how such upgrades can be
orchestrated by TripleO between the Newton and Queens releases.

Problem Description
===================

OpenStack upgrades are often seen by operators as problematic [1]_ [2]_.
Whilst TripleO upgrades have improved greatly over recent cycles many operators
are still reluctant to upgrade with each new release.

This often leads to a situation where environments remain on the release used
when first deployed. Eventually this release will come to the end of its
supported life (EOL), forcing operators to upgrade to the next supported
release. There can also be restrictions imposed on an environment that simply
do not allow upgrades to be performed ahead of the EOL of a given release,
forcing operators to again wait until the release hits EOL.

While it is possible to then linearly upgrade to a supported release with the
cadence of upstream releases, downstream distributions providing long-term
support (LTS) releases may not be able to provide the same path once the
initially installed release reaches EOL. Operators in such a situation may also
want to avoid running multiple lengthy linear upgrades to reach their desired
release.

Proposed Change
===============

Overview
--------

TripleO support for fast-forward upgrades will first target `N` to `N+3`
upgrades between the Newton and Queens releases:

.. code-block:: bash

    Newton    Ocata     Pike       Queens
    +-----+   +-----+   +-----+    +-----+
    |     |   | N+1 |   | N+2 |    |     |
    |  N  | ---------------------> | N+3 |
    |     |   |     |   |     |    |     |
    +-----+   +-----+   +-----+    +-----+


This will give the impression of the Ocata and Pike releases being skipped with
the fast-forward upgrade moving the environment from Newton to Queens. In
reality as OpenStack projects with the `supports-upgrade` tag are only required
to support `N` to `N+1` upgrades [3]_ the upgrade will still need to move
through each release, completing database migrations and limited other tasks.

Caveats
-------

Before outlining the suggested changes to TripleO it is worth highlighting the
following caveats for fast-forward upgrades:

* The control plane is inaccessible for the duration of the upgrade
* The data plane and active workloads must remain available for the duration of
  the upgrade.

Prerequisites
-------------

Prior to the overcloud fast-forward upgrade starting the following prerequisite
tasks must be completed:

* Rolling minor update of the overcloud on `N`

This is a normal TripleO overcloud update [4]_ and should bring each node in
the environment up to the latest supported version of the underlying OS and
pulling in the latest packages. Operators can then reboot the nodes as
required. The reboot ensuring that the latest kernel, openvswitch, QEMU and any
other reboot dependant package is reloaded before proceeding with the upgrade.
This can happen well in advance of the overcloud fast-forward upgrade and
should remove the need for additional reboots during the upgrade.

* Upgrade undercloud from `N` to `N+3`

The undercloud also needs to be upgraded to `N+3` ahead of any overcloud
upgrade. Again this can happen well in advance of the overcloud upgrade. For
the time being this is a traditional, linear upgrade between `N` and `N+1`
until we reach the target `N+3` Queens release.

* Container images uploaded prior to the start of the upgrade

With the introduction of containerised TripleO overclouds in Pike operators
will need to upload the required container images prior to the fast-forward
upgrade if they wish to end up with a constainerised Queens overcloud.

High level flow
---------------

At a high level the following actions will be carried out by the fast-forward
upgrade to move the overcloud from `N` to `N+3`:

* Stop all control and compute services across all roles

This should bring down the control plane but allow any workloads to continue
running without interruption.

* Upgrade a single host from `N` to `N+1` then `N+1` to `N+2`

As alluded to earlier, OpenStack projects currently only support `N` to `N+1`
upgrades and so fast-forward upgrades still need to cycle through each release in
order to complete data migrations and any other tasks that are required before
these migrations can be completed. This part of the upgrade is limited to a
single host to ensure this is completed as quickly as possible.

By default a single node of role `controller` will be selected for this part of
the upgrade however this should be fully configurable by the operator driving
the upgrade.

* Upgrade and deployment of all roles to `N+3`

The final action in the fast-forward upgrade will be a traditional `N` to `N+1`
migration between `N+2` and `N+3` followed by the deployment of all roles on
`N+3`. The final deployment ensuring the overcloud is containerised.

Implementation
--------------

As with updates [5]_ and upgrades [6]_ specific fast-forward upgrade Ansible
tasks associated with the first two actions above will be introduced into the
`tripleo-heat-template` service templates for each service as `role_data`
outputs under `fast_forward_upgrade_tasks`.

As with `upgrade_tasks` these tasks will be broken into various steps:

1) Stop all control-plane services
2) Quiesce the control-plane
3) Perform a package update and install new packages
4) Add and potentially start services needed for migration tasks
5) Perform any migration tasks, e.g DB sync commands
6) Optionally validate and cleanup after migration tasks complete

As with `update_tasks` each task will use a simple `when` conditional to
identify which step and release(s) it is associated with, ensuring these tasks
are executed at the correct point in the upgrade.

For example, a step 2 `fast_forward_upgrade_task` task on Ocata is listed below:

.. code-block:: yaml

   fast_forward_upgrade_tasks:
     - name: Example Ocata step 2 task
       command: /bin/foo bar
       when:
         - step == 2
         - release == 'ocata'


These tasks will then be collated into role specific Ansible playbooks via the
RoleConfig output of the `overcloud` heat template, with step and release
variables being fed in to ensure tasks are executed in the correct order. These
playbooks should also include the required `upgrade_tasks` and `deploy_tasks`
as highlighted above to ensure baremetal overclouds are migrated to containers.

As with `major upgrades` [8] a new mistral workflow and tripleoclient command
will be introduced to generate and execute the associated Ansible tasks.

.. code-block:: bash

    openstack overcloud fast-forward-upgrade --templates [..path to latest THT..] \
                               [..original environment arguments..] \
                               [..new container environment agruments..]

Operators will also be able to generate [7]_ , download and review the
playbooks ahead of time using the latest version of `tripleo-heat-templates`
with the following commands:

.. code-block:: bash

    openstack overcloud deploy --templates [..path to latest THT..] \
                               [..original environment arguments..] \
                               [..new container environment agruments..] \
                               --setup-heat-outputs
    openstack overcloud config download --config-dir ~/overcloud-config


Dev workflow
------------

A new tripleo-quickstart-extras `libvirt/snapshot` Ansible role will be
introduced to snapshot the initial undercloud and overcloud domains when these
are deployed using libvirt.

A new tripleo-fast-forward-upgrade Ansible role will be introduced to automate
the fast-forward upgrade process, including the initial overcloud minor update,
undercloud upgrade to `N+3` and fast-forward upgrade itself. Again, snapshots
of the undercloud and overcloud domains will be taken between each step when
these are deployed using libvirt to hopefully make development less painful.

Developers working on fast_forward_upgrade_tasks will be able to utilise the
above roles to snapshot their environments ahead of testing new or modified
tasks. Further as these tasks can be rendered as Ansible playbooks developers
can also run a subset of these tasks against specific nodes using custom
playbooks during their development work. Examples of how to do this will be
documented hopefully ensuring a smooth development experience for anyone
looking to contribute tasks for specific services.

Alternatives
------------

* Continue to force operators to upgrade linearly through each major release
* Parallel cloud migrations.

Security Impact
---------------

N/A

Other End User Impact
---------------------

* The control plane will be down for the duration of the upgrade
* The data plane and workloads will remain up.

Performance Impact
------------------

N/A

Other Deployer Impact
---------------------

N/A

Developer Impact
----------------

* Third party service template providers will need to provide
  fast_forward_upgrade_steps in their THT service configurations.

Implementation
==============

Assignee(s)
-----------

Primary assignees:
* lyarwood
* lbezdick

Other contributors:
* shardy


Work Items
----------

* Introduce fast_forward_upgrades_playbook.yaml to RoleConfig
* Introduce fast_forward_upgrade_tasks in each service template

Dependencies
============

N/A

Testing
=======

* Third party CI jobs will need to be created to test Newton to Queens using
  RDO given the upstream EOL of stable/newton with the release of Pike.

* An additional third party CI job will be required to verify that a Queens
  undercloud can correctly manage a Newton overcloud, allowing the separation
  of the undercloud upgrade and fast-forward upgrade discussed under
  prerequisites.


Documentation Impact
====================

* This will require extensive developer and user documentation to be written,
  most likely in a new section of the docs specifically detailing the
  fast-forward upgrade flow.

References
==========
.. [1] https://etherpad.openstack.org/p/MEX-ops-migrations-upgrades
.. [2] https://etherpad.openstack.org/p/BOS-forum-skip-level-upgrading
.. [3] https://governance.openstack.org/tc/reference/tags/assert_supports-upgrade.html
.. [4] http://tripleo.org/install/post_deployment/package_update.html
.. [5] https://github.com/openstack/tripleo-heat-templates/blob/master/puppet/services/README.rst#update-steps
.. [6] https://github.com/openstack/tripleo-heat-templates/blob/master/puppet/services/README.rst#upgrade-steps
.. [7] https://review.openstack.org/#/c/495658/
.. [8] https://review.openstack.org/#/q/topic:major-upgrade+(status:open+OR+status:merged)
