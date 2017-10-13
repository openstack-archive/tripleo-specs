
..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
TripleO - Ansible upgrade Worklow with UI integration
==========================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/major-upgrade-workflow

During the Pike cycle the minor update and some parts of the major upgrade
are significantly different to any previous cycle, in that they are *not* being
delivered onto nodes via Heat stack update. Rather, Heat stack update is used
to only collect, but not execute, the relevant ansible tasks defined in each
of the the service manifests_ as upgrade_tasks_ or update_tasks_ accordingly.
These tasks are then written as stand-alone ansible playbooks in the stack
outputs_.

These 'config' playbooks are then downloaded using the *openstack overcloud
config download* utility_ and finally executed to deliver the actual
upgrade or update. See bugs 1715557_ and 1708115_ for more information
(or pointers/reviews) about this mechanism as used during the P cycle.

For Queens and as discussed at the Denver PTG_ we aim to extend this approach
to include the controlplane upgrade too. That is, instead of using HEAT
SoftwareConfig and Deployments_  to invoke_ ansible we should instead collect
the upgrade_tasks for the controlplane nodes into ansible playbooks that can
then be invoked to deliver the actual upgrade.

Problem Description
===================

Whilst it has continually improved in each cycle, complexity and difficulty to
debug or understand what has been executed at any given point of the upgrade
is still one of the biggest complaints from operators about the TripleO
upgrades workflow. In the P cycle and as discussed above, the minor version
update and some part of the 'non-controller' upgrade have already moved to the
model being proposed here, i.e. generate ansible-playbooks with an initial heat
stack update and then execute them.

If we are to use this approach for all parts of the upgrade, including for the
controlplane services then we will also need a mistral workbook that can handle
the download and execution of the ansible-playbook invocations. With this kind
of ansible driven workflow, executed by mistral action/workbook, we can for
the first time consider integration with the UI for upgrade/updates. This
aligns well with the effort_ by the UI team for feature parity in CLI/UI for
Queens. It should also be noted that there is already some work underway to
adding the required mistral actions, at least for the minor update for Pike
deployments in changes 487488_ and 487496_

Implementing a fully ansible-playbook delivered workflow for the entire major
upgrade workflow will offer a number of benefits:

    * very short initial heat stack update to generate the playbooks
    * easier to follow and understand what is happening at a given step of the upgrade
    * easier to debug and re-run any particular step of the upgrade
    * implies full python-tripleoclient and mistral workbook support for the
      ansible-playbook invocations.
    * can consider integrating upgrades/updates into the UI, for the first time

Proposed Change
===============

We will need an initial heat stack update to populate the
upgrade_tasks_playbook into the overcloud stack output (the cli is just a
suggestion):

    * openstack overcloud upgrade --init --init-commands [ "sudo curl -L -o /etc/yum.repos.d/delorean-pike.repo https://trunk.rdoproject.org/centos7-ocata/current/pike.repo",
                                                           "sudo yum install my_package", ... ]

The first step of the upgrade will be used to deliver any required common
upgrade initialisation, such as switching repos to the target version,
installing any new packages required during the upgrade, and populating the upgrades playbooks.

Then the operator will run the upgrade targeting specific nodes:

    * openstack overcloud upgrade --nodes [overcloud-novacompute-0, overcloud-novacompute-1] or
      openstack overcloud upgrade --nodes "Compute"

Download and execute the ansible playbooks on particular specified set of
nodes. Ideally we will make it possible to specify a role name with the
playbooks being invoked in a rolling fashion on each node.

One of the required changes is to convert all the service templates to have
'when' conditionals instead of the current 'stepN'. For Pike we did this in
the client_ to avoid breaking the heat driven upgrade workflow still in use
for the controlplane during the Ocata to Pike upgrade. This will allow us to
use the 'ansible-native' loop_ control we are currently using in the generated
ansible playbooks.


Other End User Impact
---------------------

There will be significant changes to the workflow and cli the operator uses
for the major upgrade as documented above.

Performance Impact
------------------

The initial Heat stack update will not deliver any of the puppet or docker
config to nodes since the DeploymentSteps will be disabled_ as we currently
do for Pike minor update. This will mean a much shorter heat stack update -
exact numbers TBD but 'minutes not hours'.

Developer Impact
----------------

Should make it easier for developers to debug particular parts of the upgrades
workflow.


Implementation
==============

Assignee(s)
-----------
Contributors:
Marios Andreou (marios)
Mathieu Bultel (matbu)
Sofer Athlang Guyot (chem)
Steve Hardy (shardy)
Carlos Ccamacho (ccamacho)
Jose Luis Franco Arza (jfrancoa)
Marius Cornea (mcornea)
Yurii Prokulevych (yprokule)
Lukas Bezdicka (social)
Raviv Bar-Tal (rbartal)
Amit Ugol (amitu)

Work Items
----------

    * Remove steps and add when for all the ansible upgrade tasks, minor
      update tasks, deployment steps, post_upgrade_tasks
    * Need mistral workflows that can invoke the required stages of the
      workflow (--init and --nodes). There is some existing work in this
      direction in 463765_.
    * CLI/python-tripleoclient changes required. Related to the previous
      item there is some work started on this in 463728_.
    * UI work - we will need to collaborate with the UI team for the
      integration. We have never had UI driven upgrade or updates.
    * CI: Implement a simple job (one nodes, just controller, which does the
      heat-setup-output and run ansible --nodes Controller) with keystone
      only upgrade. Then iterate on this as we can add service upgrade_tasks.
    * Docs!

Testing
=======

We will aim to land a 'keystone-only' job asap which will be updated as the various
changes required to deliver this spec are closer to merging. For example we
may deploy only a very small subset of services (e.g. first keystone) and then iterate as changes
related to this spec are proposed.

Documentation Impact
====================

We should also track changes in the documented upgrades workflow since as
described here it is going to change significantly both internally as well as
the interface exposed to an operator.

References
==========
Check the source_ for links

.. _manifests: https://github.com/openstack/tripleo-heat-templates/tree/master/docker/services
.. _upgrade_tasks: https://github.com/openstack/tripleo-heat-templates/blob/211d7f32dc9cda261e96c3f5e0e1e12fc0afdbb5/docker/services/nova-compute.yaml#L147
.. _update_tasks: https://github.com/openstack/tripleo-heat-templates/blob/60f3f10442f3b4cedb40def22cf7b6938a39b391/puppet/services/tripleo-packages.yaml#L59
.. _outputs: https://github.com/openstack/tripleo-heat-templates/blob/3dcc9b30e9991087b9e898e25685985df6f94361/common/deploy-steps.j2#L324-L372
.. _utility: https://github.com/openstack/python-tripleoclient/blob/27bba766daa737a56a8d884c47cca1c003f16e3f/tripleoclient/v1/overcloud_config.py#L26-L154
.. _1715557: https://bugs.launchpad.net/tripleo/+bug/1715557
.. _1708115: https://bugs.launchpad.net/tripleo/+bug/1708115
.. _PTG: https://etherpad.openstack.org/p/tripleo-ptg-queens-upgrades
.. _Deployments: https://github.com/openstack/tripleo-heat-templates/blob/f4730632a51dec2b9be6867d58184fac3b8a11a5/common/major_upgrade_steps.j2.yaml#L132-L173
.. _invoke: https://github.com/openstack/tripleo-heat-templates/blob/f4730632a51dec2b9be6867d58184fac3b8a11a5/puppet/upgrade_config.yaml#L21-L50
.. _effort: http://lists.openstack.org/pipermail/openstack-dev/2017-September/122089.html
.. _487488: https://review.openstack.org/#/c/487488/
.. _487496: https://review.openstack.org/#/c/487496/
.. _client: https://github.com/openstack/python-tripleoclient/blob/4d342826d6c3db38ee88dccc92363b655b1161a5/tripleoclient/v1/overcloud_config.py#L63
.. _loop: https://github.com/openstack/tripleo-heat-templates/blob/fe2acfc579295965b5f39c5ef7a34bea35f3d6bf/common/deploy-steps.j2#L364-L365
.. _disabled: https://review.openstack.org/#/c/487496/21/tripleo_common/actions/package_update.py@63
.. _source: https://raw.githubusercontent.com/openstack/github.com/openstack/tripleo-specs/tree/master/specs/queens/tripleo_ansible_upgrades_workflow.rst
.. _463728: https://review.openstack.org/#/c/463728/
.. _463765: https://review.openstack.org/#/c/463765/
