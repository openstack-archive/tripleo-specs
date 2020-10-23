..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
TripleO Ceph Ganesha Integration for Manila
===========================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-ceph-ganesha

Starting in the Octopus release, Ceph has its own day1 tool called cephadm and
its own day2 tool called orchestrator which will replace ceph-ansible.

During the Wallaby cycle TripleO will no longer use ceph-ansible for Ceph
deployment and instead use cephadm [2]_ as described in [1]_. Ganesha deserves
special attention because for its deployment we will use special functionalities
in cephadm [2]_ meant to deploy the Ganesha service standalone when the Ceph
cluster is external.

Problem Description
===================

In TripleO we support deployment of Ganesha both when the Ceph cluster is itself
managed by TripleO and when the Ceph cluster is itself not managed by TripleO.

When the Ceph cluster is *not* managed by Tripleo, the Ganesha service must be
deployed standalone; that is, without any additional core Ceph daemon and it
should instead be configured to use the external Ceph MON and MDS daemons.

Proposed Change
===============

Overview
--------

An ansible task will trigger cephadm [2]_ with special arguments for it to stand
up a standalone Ganesha container and to it we will provide:

- the Ceph cluster config file, generated using tripleo-ceph-client [3]_ role
- the Ceph cluster keyring to interact with MDS
- the Ganesha config file with pointers to the Ceph config/keyring to use

The container will then be controlled by pacemaker, as it is today and reusing
the same code which today manages the ceph-nfs systemd service created by
ceph-ansible.

Alternatives
------------

Forking and reusing the existing ceph-ansible role for ceph-nfs has been
discussed but ultimately discarded as that would have moved ownership of the
Ganesha deployment tasks in TripleO, while our goal remaing to keep ownership
where subject expertise is, in the Ceph deployment tool.

Security Impact
---------------

None, the same code which TripleO would already use for the generation of the
Ceph cluster config and keyrings will be consumed.

Upgrade Impact
--------------

Some upgrade tasks which stop and remove the pre-existing ceph-nfs container
and systemd unit will be added to clean up the system from the ceph-ansible
managed resources.

Other End User Impact
---------------------

None, the existing input parameters will be reused to drive the newer deployment
tool.

Performance Impact
------------------

No changes.

Other Deployer Impact
---------------------

No impact on users.

Developer Impact
----------------

The Ganesha config file will be generated using a specific tripleo-ceph task
while previously, with ceph-ansible, this was created by ceph-ansible itself.

Implementation
==============

The existing implementation which depends on ceph-ansible will remain
in-tree for at least 1 deprecation cycle. By reusing the existing Heat
input parameters we should be able to transparently make the Ganesha
deployment happen with ceph-ansible or the new role just by switching
the environment file used at deployment time.

Deployment Flow
---------------

The deployment and configuration described in this spec will
happen before `openstack overcloud deploy`, as described in
[1]_. This is consistent with how ceph-ansible used to run during
step2 to configure these services. However, parts of the Manila
configuration which use Ganesha will still happen when `openstack
overcloud deploy` is run. This is because some of the configuration
for Ganesha and Manila needs to happen during step 5. Thus, files like
`environments/manila-cephfsganesha-config.yaml` will be updated to
trigger the new required actions.

Assignee(s)
-----------

- fmount
- fultonj
- gfidente

Work Items
----------

- Create a set of tasks to deploy on overcloud nodes the Ganesha config file
- Create a set of tasks to trigger cephadm with special arguments

Dependencies
============

- The tripleo-ceph spec [1]_

Testing
=======

Testing is currently impossible as we only have one network while for Ganesha
we require at least two, one which connects it to the Ceph public network and
another where the NFS proxy service is exposed to tenants.

This is a design decision, one of the values added by the use of an NFS proxy
for CephFS is to implement network isolation in between the tenant guests and
the actual Ceph cluster.

Such a limitation does not come from the migration to cephadm [2]_ but it has
always existed; the code which enforces the use of two isolated networks is in
fact in TripleO, not in the Ceph tool itself. We might revisit this in the
future but it is not a goal of this spec to change this.

Documentation Impact
====================

No changes should be necessary to the TripleO documentation.

References
==========

.. [1] `tripleo-ceph <https://specs.openstack.org/openstack/tripleo-specs/specs/wallaby/tripleo-ceph.html>`_
.. [2] `cephadm <https://github.com/ceph/ceph/tree/master/src/cephadm>`_
.. [3] `tripleo-ceph-client <https://specs.openstack.org/openstack/tripleo-specs/specs/wallaby/tripleo-ceph-client.html>`_
