..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Composable Service Upgrades
===========================

https://blueprints.launchpad.net/tripleo/+spec/overcloud-upgrades-per-service

In the Newton release TripleO delivered a new capability to deploy arbitrary
custom roles_ (groups of nodes) with a lot of flexibility of which services
are placed on which roles (using roles_data.yaml_). This means we can no
longer make the same assumptions about a specific service running on a
particular role (e.g Controller).

The current upgrades workflow_ is organised around the node role determining
the order in which that given node and services deployed therein are upgraded.
The workflow dictates "swifts", before "controllers", before "cinders", before
"computes", before "cephs". The reasons for this ordering are beyond the scope
here and ultimately inconsequential, since the important point to note is
there is a hard coded relationship between a given service and a given node
with respect to upgrading that service (e.g. a script that upgrades all
services on "Compute" nodes).  For upgrades from Newton to Ocata we can no
longer make these assumptions about services being tied to a specific role,
so a more composable model is needed.

Consensus after the initial discussion during the Ocata design summit session_
was that:

    * Re-engineering the upgrades workflow for Newton to Ocata is necessary
      because 'custom roles'
    * We should start by moving the upgrades logic into the composable service
      templates in the tripleo-heat-templates (i.e. into each service)
    * There is still a need for an over-arching workflow - albeit service
      rather than role oriented.
    * It is TBD what will drive that workflow. We will use whatever will be
      'easier' for a first iteration, especially given the Ocata development
      time contraints.

Problem Description
===================

As explained in the introduction above, the current upgrades workflow_ can no
longer work for composable service deployments. Right now the upgrade scripts
are organised around and indeed targetted at specific nodes: the upgrade
script for swifts_ is different to that for computes_ or for controllers (split
across a number_ of_ steps_) cinders_ or cephs_. These scripts are invoked
as part of a worfklow where each step is either a heat stack update or
invocation of the upgrade-non-controller.sh_ script to execute the node
specific upgrade script (delivered as one of the earlier steps in the workflow)
on non controllers.

One way to handle this problem is to decompose the upgrades logic
from those monolithic per-node upgrade scripts into per-service upgrades logic.
This should live in the tripleo-heat-templates puppet services_ templates for
each service. For the upgrade of a give service we need to express:

    * any pre-upgrade requirements (run a migration, stop a service, pin RPC)
    * any post upgrade (migrations, service starts/reload config)
    * any dependencies on other services (upgrade foo only after bar)

If we organise the upgrade logic in this manner the idea is to gain the
flexibility to combine this dynamically into the new upgrades workflow.
Besides the per-service upgrades logic the worklow will also need to handle
and provide for any deployment wide upgrades related operations such as
unpin of the RPC version once all services are successfully running Ocata, or
upgrading of services that aren't directly managed or configured by the
tripleo deployment (like openvswitch as just one example), or even the delivery
of a new kernel which will require a reboot on the given service node after
all services have been upgraded.


Proposed Change
===============

The first step is to work out where to add upgrades related configuration to
each service in the tripleo-heat-templates services_ templates. The exact
format will depend on what we end up using to drive the workflow. We could
include them in the *outputs* as 'upgrade_config', like::

    outputs:
      role_data:
        description: Role data for the Nova Compute service.
        value:
          service_name: nova_compute
          ...
        upgrade_tasks:
            - name: RPC pin nova-compute
              exec: "crudini --set /etc/nova/nova.conf upgrade_levels compute $upgrade_level_nova_compute"
              tags: step1
            - name: stop nova-compute
              service: name=openstack-nova-compute state=stopped
              tags: step2
            - name: update heat database
              command: nova-manage db_sync
              tags: step3
            - name: start nova-compute
              service: name=openstack-nova-compute state=started
              tags: step4
            ...

The current proposal is for the upgrade snippets to be expressed in Ansible.
The initial focus will be to drive the upgrade via the existing tripleo
tooling, e.g heat applying ansible similar to how heat applies scripts for
the non composable implementation.  In future it may also be possible to
expose the per-role ansible playbooks to enable advanced operators to drive
the upgrade workflow directly, perhaps used in conjunction with the dynamic
inventory provided for tripleo validations.

One other point of note that was brought up in the Ocata design summit
session_ and which should factor into the design here is that operators may
wish to run the upgrade in stages rather than all at once. It could still be
the case that the new workflow can differentiate between 'controlplane'
vs 'non-controlplane' services. The operator could then upgrade controlplane
services as one stand-alone upgrade step and then later start to roll out the
upgrade of non-controlplane services.

Alternatives
------------

One alternative is to have a stand-alone upgrades workflow driven by ansible.
Some early work and prototyping was done as well as a (linked from the
Ocata design summit session_). Ultimately the proposal was abandoned but it is
still possible that we will use ansible for the upgrade logic as described
above. We could also explore exposing the resulting ansible playbooks for
advanced operators to invoke as part of their own tooling.

Other End User Impact
---------------------
Significant change in the tripleo upgrades workflow.

Implementation
==============

Assignee(s)
-----------

Primary assignee: shardy

Other contributors: marios, emacchi, matbu, chem, lbezdick,


Work Items
----------
Some prototyping by shardy at
"WIP prototyping composable upgrades with Heat+Ansible" at
I39f5426cb9da0b40bec4a7a3a4a353f69319bdf9_

    * Decompose the upgrades logic into each service template in the tht
    * Design a workflow that incorporates migrations, the per-service upgrade
      scripts and any deployment wide upgrades operations.
    * Decide how this workflow is to be invoked (mistral? puppet? bash?)
    * profit!


Dependencies
============



Testing
=======

Hopefully we can use the soon to be added upgrades job_ to help with the
development and testing of this feature and obviously guard against changes
that break upgrades. Ideally we will expand that to include jobs for each of
the stable branches (upgrade M->N and N->O). The M->N would exercise the
previous upgrades workflow whereas N->O would be exercising the work developed
as part of this spec.


Documentation Impact
====================


References
==========


.. _roles: https://blueprints.launchpad.net/tripleo/+spec/custom-roles
.. _roles_data.yaml: https://github.com/openstack/tripleo-heat-templates/blob/78500bc2e606bd1f80e05d86bf7da4d1d27f77b1/roles_data.yaml
.. _workflow: http://docs.openstack.org/developer/tripleo-docs/post_deployment/upgrade.html
.. _session: https://etherpad.openstack.org/p/ocata-tripleo-upgrades
.. _swifts: https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/extraconfig/tasks/major_upgrade_object_storage.sh
.. _computes: https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/extraconfig/tasks/major_upgrade_compute.sh
.. _number: https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/extraconfig/tasks/major_upgrade_controller_pacemaker_1.sh
.. _of: https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/extraconfig/tasks/major_upgrade_controller_pacemaker_2.sh
.. _steps: https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/extraconfig/tasks/major_upgrade_controller_pacemaker_3.sh
.. _cinders: https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/extraconfig/tasks/major_upgrade_block_storage.sh
.. _cephs: https://github.com/openstack/tripleo-heat-templates/blob/stable/newton/extraconfig/tasks/major_upgrade_ceph_storage.sh
.. _upgrade-non-controller.sh: https://github.com/openstack/tripleo-common/blob/01b68d0b0cdbd0323b7f006fbda616c12cbf90af/scripts/upgrade-non-controller.sh
.. _services: https://github.com/openstack/tripleo-heat-templates/tree/master/puppet/services
.. _I39f5426cb9da0b40bec4a7a3a4a353f69319bdf9 : https://review.openstack.org/#/c/393448/
.. _job: https://bugs.launchpad.net/tripleo/+bug/1583125
