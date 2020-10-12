..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
TripleO Ceph Client
===================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-ceph-client

Native Ansible roles for TripleO integration with Ceph clusters.


Problem Description
===================

Starting in the Octopus release, Ceph has its own day1 tool called
cephadm [1]_ and it's own day2 tool called orchestrator [2]_ which
will replace ceph-ansible [3]_. While ceph-ansible had the necessary
features to configure Ceph clients, distributing for example config file
and keyrings as necessary on nodes which aren't members of the Ceph cluster,
neither cephadm or the orchestrator will manage Ceph clients configuration.

Goal is to create some new ansible roles in TripleO to perform the
Ceph clients (Nova, Cinder, Glance, Manila) configuration, which is of special
importance in TripleO to support deployment scenarios where the Ceph cluster
is externally managed, not controlled by the undercloud, yet the OpenStack
services configuration remains a responsibility of TripleO.


.. _proposed-change:

Proposed Change
===============

Overview
--------

Introduce a new role into tripleo-ansible for Ceph client configuration.

The new role will:

- Configure OpenStack services as clients of an external Ceph cluster
  (in the case of collocation, the ceph cluster is still logically
  external)
- Provide Ceph configuration files and cephx keys for OpenStack
  clients of RBD and CephFS (Nova, Cinder, Glance, Manila)
- Full multiclient support, e.g. one OpenStack deployment may use
  multiple Ceph clusters, e.g. multibackend Glance
- Configure clients quickly, e.g. generate the key in one place
  and copy it efficiently
- This is a standalone role which is reusable to configure OpenStack
  against an externally managed Ceph cluster
- Not break existing support for CephExternalMultiConfig which is used
  for configuring OpenStack to work with more than one Ceph cluster
  when deploying Ceph in DCN environments (Deployment of dashboard on
  DCN sites is not in scope with this proposal).


Alternatives
------------

Support for clients configuration might be added in future versions
of cephadm, yet there are some reasons why we won't be able to use this
feature as-is even if it was available today:

- it assumes the for the cephadm tool to be configured with admin privileges
  for the external Ceph cluster, which we don't have when Ceph is not
  managed by TripleO;
- it also assumes that each and every client node has been provisioned into
  the external Ceph orchestrator inventory so that evey Ceph MON is able to
  log into the client node (overcloud nodes) via SSH;
- while offering the necessary functionalities to copy the config
  files and cephx keyrings over to remote client nodes, it won't be able to
  configure for example Nova with the libvirtd secret for qemu-kvm, which is
  a task only relevant when the client is OpenStack;

Security Impact
---------------

None derived directly from the decision to create new ansible roles. The
distribution of the cephx keyrings itself though should be implemented using
a TripleO service, like the existing CephClient service, so that keyrings
are only deployed on those nodes which actually need those.

Upgrade Impact
--------------

The goal is to preserve and reuse any existing Heat parameter which is
currently consumed to drive ceph-ansible; from operators' perspective the
problem of configuring a Ceph client isn't changed and there shouldn't be
a need to change the existing parameters, it's just the implementation
which will change.

Performance Impact
------------------

As described in the :ref:`proposed-change` section, the purpose of this
role is to proper configure clients and it allows OpenStack services to
connect to an internal or external Ceph cluster, as well as multiple Ceph
cluster in a DCN context.
Since both config files and keys are necessary for many OpenStack services
(Nova, Cinder, Glance, Manila) to make them able to properly interact with
the Ceph cluster, at least two actions should be performed:

- generate keys in one place
- copy the generated keys efficiently

The `ceph_client` role should be very small, and a first improvement
in terms of performances can be found on key generation since they are
created in one, centralized place.
The generated keys, then, just need to be distributed across the nodes
of the Ceph cluster, as well as the Ceph cluster config file.
Adding this role to tripleo-ansible avoid adding an extra calls from
a pure deployment perspective; in fact, no additional ansible playbooks
will be triggered and we expect to see performances improved since no
additional layers are involved here.

Developer Impact
----------------

How Ceph is deployed could change for anyone maintaining TripleO code
for OpenStack services which use Ceph. In theory there should be no
change as the CephClient service will still configure the Ceph
configuration and Ceph key files in the same locations. Those
developers will just need to switch to the new templates when they are
stable.


Implementation
==============

The new role should be enabled by a TripleO service, like it happens
today with the CephClient service.
Depending on the environment file chosen at deployment time, the
actual implementation of such a service could either be based on
ceph-ansible or on the new role.

When the Ceph cluster is not external, the role will also create
pools and the cephx keyrings into the Ceph cluster; these steps
will be skipped instead when Ceph is external precisely because we won't
have admin privileges to change the cluster configuration in that case.

TripleO Heat Templates
----------------------

The existing implementation which depends on ceph-ansible will remain
in-tree for at least 1 deprecation cycle. By reusing the existing Heat
input parameters we should be able to transparently make the clients
configuration happen with ceph-ansible or the new role just by
switching the environment file used at deployment time.
TripleO users who currently use
`environments/ceph-ansible/ceph-ansible-external.yaml` in order to
have their Overcloud use an existing Ceph cluster, should be able to
apply the same templates to the new template for configuring Ceph
clients, e.g. `environments/ceph-client.yaml`. This will result in
the new tripleo-ansible/roles/ceph_client role being executed.

Assignee(s)
-----------

- fmount
- fultonj
- gfidente
- jmolmo

Work Items
----------

Proposed Schedule
-----------------

- OpenStack W: start tripleo-ansible/roles/ceph_client as experimental
  and then set it as default in scenarios 001/004. We expect to to
  become stable during the W cycle.

Dependencies
============

The `ceph_client` role will be added in tripleo-ansible and allow
configuring the OpenStack services as clients of an external or TripleO
managed Ceph cluster; no new dependencies are added for tripleo-ansible
project. The `ceph_client` role will work with External Ceph, Internal
Ceph deployed by ceph-ansible, and the Ceph deployment described in
[4]_.

Testing
=======

It should be possible to reconfigure one of the existing CI scenarios
already deploying with Ceph to use the newer `ceph_client` role,
making it non-voting until the code is stable. Then switch the other
existing CI scenario to it.


Documentation Impact
====================

No doc changes should be needed.


References
==========

.. [1] `cephadm <https://github.com/ceph/ceph/tree/master/src/cephadm>`_
.. [2] `orchestrator <https://docs.ceph.com/docs/octopus/mgr/orchestrator/>`_
.. [3] `ceph-ansible <https://github.com/ceph/ceph-ansible>`_
.. [4] `tripleo-ceph <https://review.opendev.org/#/c/723108>`_
