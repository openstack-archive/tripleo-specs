..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Make tripleo third party ci toolset tripleo-quickstart
======================================================

https://blueprints.launchpad.net/tripleo/+spec/use-tripleo-quickstart-and-tripleo-quickstart-extras-for-the-tripleo-ci-toolset

Devstack being the reference CI deployment of OpenStack does a good job at
running both in CI and locally on development hardware.
TripleO-Quickstart (TQ)`[3]`_ and TripleO-QuickStart-Extras (TQE) can provide
an equivalent experience like devstack both in CI and on local development
hardware. TQE does a nice job of breaking down the steps required to install an
undercloud and deploy and overcloud step by step by creating bash scripts on the
developers system and then executing them in the correct order.


Problem Description
===================

Currently there is a population of OpenStack developers that are unfamiliar
with TripleO and our TripleO CI tools. It's critical that this population have
a tool which can provide a similar user experience that devstack currently
provides OpenStack developers.

Recreating a deployment failure from TripleO-CI can be difficult for developers
outside of TripleO. Developers may need more than just a script that executes
a deployment. Ideally developers have a tool that provides a high level
overview, a step-by-step install process with documentation, and a way to inject
their local patches or patches from Gerrit into the build.

Additionally there may be groups outside of TripleO that want to integrate
additional code or steps to a deployment. In this case the composablity of the
CI code is critical to allow others to plugin, extend and create their own steps
for a deployment.


Proposed Change
===============

Overview
--------

Replace the tools found in openstack-infra/tripleo-ci that drive the deployment
of tripleo with TQ and TQE.

Alternatives
------------

One alternative is to break down TripleO-CI into composable shell scripts, and
improve the user experience `[4]`_.

Security Impact
---------------

No known additional security vulnerabilities at this time.

Other End User Impact
---------------------

We expect that newcomers to TripleO will have an enhanced experience
reproducing results from CI.

Performance Impact
------------------

Using an undercloud image with preinstalled rpms should provide a faster
deployment end-to-end.

Other Deployer Impact
---------------------

None at this time.

Developer Impact
----------------

This is the whole point really and discussed elsewhere in the spec. However,
this should provide a quality user experience for developers wishing to setup
TripleO.

TQE provides a step-by-step, well documented deployment of TripleO.
Furthermore, and is easy to launch and configure::

 bash quickstart.sh -p quickstart-extras.yml -r quickstart-extras-requirements.txt --tags all <development box>

Everything is executed via a bash shell script, the shell scripts are customized
via jinja2 templates. Users can see the command prior to executing it when
running it locally. Documentation of what commands were executed are
automatically generated per execution.

Node registration and introspection example:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Bash script::

    https://ci.centos.org/artifacts/rdo/jenkins-tripleo-quickstart-promote-newton-delorean-minimal-31/undercloud/home/stack/overcloud-prep-images.sh


* Execution log::

   https://ci.centos.org/artifacts/rdo/jenkins-tripleo-quickstart-promote-newton-delorean-minimal-31/undercloud/home/stack/overcloud_prep_images.log.gz

* Generated rst documentation::

   https://ci.centos.org/artifacts/rdo/jenkins-tripleo-quickstart-promote-newton-delorean-minimal-31/docs/build/overcloud-prep-images.html

Overcloud Deployment example:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Bash script::

   https://ci.centos.org/artifacts/rdo/jenkins-tripleo-quickstart-promote-newton-delorean-minimal_pacemaker-31/undercloud/home/stack/overcloud-deploy.sh.gz

* Execution log::

   https://ci.centos.org/artifacts/rdo/jenkins-tripleo-quickstart-promote-newton-delorean-minimal_pacemaker-31/undercloud/home/stack/overcloud_deploy.log.gz

* Generated rST documentation::

   https://ci.centos.org/artifacts/rdo/jenkins-tripleo-quickstart-promote-master-current-tripleo-delorean-minimal-37/docs/build/overcloud-deploy.html

Step by Step Deployment:
^^^^^^^^^^^^^^^^^^^^^^^^

There are times when a developer will want to walk through a deployment step-by-step,
run commands by hand, and try to figure out what exactly is involved with
a deployment. A developer may also want to tweak the settings or add a patch.
To do the above the deployment can not just run through end to end.

TQE can setup the undercloud and overcloud nodes, and then just add add already
configured scripts to install the undercloud and deploy the overcloud
successfully. Essentially allowing the developer to ssh to the undercloud and
drive the installation from there with prebuilt scripts.

* Example::

  ./quickstart.sh  --no-clone --bootstrap --requirements quickstart-extras-requirements.txt --playbook quickstart-extras.yml --skip-tags undercloud-install,undercloud-post-install,overcloud-deploy,overcloud-validate --release newton <development box>

Composability:
^^^^^^^^^^^^^^

TQE is not a single tool, it's a collection of composable Ansible roles. These
Ansible roles can coexist in a single Git repository or be distributed to many
Git repositories. See "Additional References."

Why have two projects? Why risk adding complexity?
One of the goals of the TQ and TQE is to not assume we are
writing code that works for everyone, on every deployment type, and in any
kind of infrastructure. To ensure that TQE developers can not block outside
contributions (roles, additions, and customization to either TQ or TQE),
it was thought best to uncouple as well and make it as composable
as possible. Ansible playbooks after all, are best used as a method to just
call roles so that anyone can create playbooks with a variety of roles in the
way that best suits their purpose.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  - weshayutin

Other contributors:
  - trown
  - sshnaidm
  - gcerami
  - adarazs
  - larks

Work Items
----------

- Enable third party testing `[1]`_
- Enable TQE to run against the RH2 OVB OpenStack cloud `[2]`_
- Move the TQE roles into one or many OpenStack Git Repositories, see the roles listed
  in the "Additional References"


Dependencies
============

- A decision needs to be made regarding `[1]`_
- The work to enable third party testing in rdoproject needs to be completed

Testing
=======

There is a work in progress testing TQE against the RH2 OVB cloud atm `[2]`_. TQE
has been vetted for quite some time with OVB on other clouds.


Documentation Impact
====================

What is the impact on the docs? Don't repeat details discussed above, but
please reference them here.


References
==========
* `[1]`_ -- http://lists.openstack.org/pipermail/openstack-dev/2016-October/105248.html
* `[2]`_ -- https://review.openstack.org/#/c/381094/
* `[3]`_ -- https://etherpad.openstack.org/p/tripleo-third-party-ci-quickstart
* `[4]`_ -- https://blueprints.launchpad.net/tripleo/+spec/make-tripleo-ci-externally-consumable

.. _[1]: http://lists.openstack.org/pipermail/openstack-dev/2016-October/105248.html
.. _[2]: https://review.openstack.org/#/c/381094/
.. _[3]: https://etherpad.openstack.org/p/tripleo-third-party-ci-quickstart
.. _[4]: https://blueprints.launchpad.net/tripleo/+spec/make-tripleo-ci-externally-consumable

Additional References
=====================

TQE Ansible role library
------------------------

* Undercloud roles:

 * https://github.com/redhat-openstack/ansible-role-tripleo-baremetal-virt-undercloud
 * https://github.com/redhat-openstack/ansible-role-tripleo-pre-deployment-validate ( under development )

* Overcloud roles:

 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-prep-config
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-prep-flavors
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-prep-images
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-prep-network
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud
 * https://github.com/redhat-openstack/ansible-role-tripleo-ssl  ( under development )

* Utility roles:

 * https://github.com/redhat-openstack/ansible-role-tripleo-cleanup-nfo
 * https://github.com/redhat-openstack/ansible-role-tripleo-collect-logs
 * https://github.com/redhat-openstack/ansible-role-tripleo-gate
 * https://github.com/redhat-openstack/ansible-role-tripleo-provision-heat
 * https://github.com/redhat-openstack/ansible-role-tripleo-image-build

* Post Deployment roles:

 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-upgrade
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-scale-nodes
 * https://github.com/redhat-openstack/ansible-role-tripleo-tempest
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-validate
 * https://github.com/redhat-openstack/ansible-role-tripleo-validate-ipmi
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-validate-ha

* Baremetal roles:

 * https://github.com/redhat-openstack/ansible-role-tripleo-baremetal-prep-virthost
 * https://github.com/redhat-openstack/ansible-role-tripleo-overcloud-prep-baremetal