..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
TripleO Ceph Ganesha Integration for Manila
===========================================

Starting in the Octopus release, Ceph introduced  its own day1 tool called
cephadm and its own day2 tool called orchestrator which replaced ceph-ansible.
During the Wallaby and Xena cycles TripleO moved away from ceph-ansible and
adopted cephadm [1]_ as described in [2]_.
However, the ganesha deamon deployment remained under the tripleo-ansible
control, with a set of tasks that are supposed to replicate the relevant part
of the ceph-nfs ceph-ansible role [3]_.
This choice ensured backward compatibility with the older releases.

Problem Description
===================

In TripleO we support deployment of Ganesha both when the Ceph cluster is
itself managed by TripleO and when the Ceph cluster is itself not managed
by TripleO.
When the cluster is managed by TripleO, an NFS daemon can be deployed as a
regular TripleO service via the tripleo-ansible module [4]_.
It is preferable to have cephadm manage the lifecycle of the NFS container
instead of deploying it with tripleo-ansible.
In order to do this we will require the following changes on both TripleO
and Manila:

- the orchestrator provides an interface that should be used by Manila to
  interact with the ganesha instances. The nfs orchestrator interface is
  described in [5]_ and can be used to manipulate the nfs daemon, as well
  as create and delete exports.
  In the past the ganesha configuration file was fully customized by
  ceph-ansible; the orchestrator is going to have a set of overrides to
  preserve backwards compatibility. This result is achieved by setting a
  userconfig object that lives within the Ceph cluster [5]_. It's going
  to be possible to check, change and reset the nfs daemon config using
  the same interface provided by the orchestrator [11]_.

- The deployed NFS daemon is based on the watch_url mechanism [6]_:
  adopting a cephadm deployed ganesha instance requires the Manila driver
  be updated to support this new approach. This work is described in [10]_.

- The ceph-nfs daemon deployed by cephadm has its own HA mechanism, called
  ingress, which is based on haproxy and keepalived [7]_ so we would no
  longer use pcmk as the VIP owner.
  Note this means we would run pcmk and keepalived in addition to haproxy
  (deployed by tripleo) and another haproxy (deployed by cephadm) on the
  same server (though with listeners on different ports).
  Because cephadm is controlling the ganesha life cycle, the pcs cli will
  no longer be used to interact with the ganesha daemon and we will change
  where the ingress daemon is used.

When the Ceph cluster is *not* managed by TripleO, the Ganesha service is
currently deployed standalone on the overcloud and it's configured to use
the external Ceph MON and MDS daemons.
However, if this spec is implemented, then the standalone ganesha service
will no longer be deployed by TripleO. Instead, we will require that the
admin of the external ceph cluster add the ceph-nfs service to that cluster.
Though TripleO will still configure Manila to use that service.

Thus in the external case, Ganesha won't be deployed and details about the
external Ganesha must be provided as input during overcloud deployment. We
will also provide tools to help someone who has deployed Ganesaha on the
overcloud transition the service to their external Ceph cluster. From a high
level the process will be the following:

1. Generate a cephadm spec so that after the external ceph cluster becomes
   managed by cephadm the spec can be used to add a the ceph-nfs service
   with the required properties.
2. Disable the VIP PCS uses and provide a documented method for it to be
   moved to the external ceph cluster.

Proposed Change
===============

Overview
--------

An ansible task will generate the Ceph NFS daemon spec and it will trigger
cephadm [2]_ to deploy the Ganesha container.

- the NFS spec should be rendered and applied against the existing Ceph
  cluster
- the ingress spec should be rendered (as part of the NFS deployment)
  and applied against the cluster

The container will be no longer controlled by pacemaker.


Security Impact
---------------

None, the same code which TripleO would already use for the generation of
the Ceph cluster config and keyrings will be consumed.

Upgrade Impact
--------------

- We will deprecate the ganesha managed by PCS so that it will still work
  up until Z.
- We will provide playbooks which migrate from the old NFS service to the
  new one.
- We will assume these playbooks will be available in Z and run prior to
  the upgrade to the next release.

Other End User Impact
---------------------

For fresh deployments, the existing input parameters will be reused to
drive the newer deployment tool.
For an existing environment, after the Ceph upgrade, the TripleO deployed
NFS instance will be stopped and removed by the migration playbook provided,
as well as the related pacemaker resources and constraints; cephadm will
be able to deploy and manage the new NFS instances, and the end user will
see a disruption in the NFS service.

Performance Impact
------------------

No changes.

Other Deployer Impact
---------------------

* "deployed ceph": For the first implementation of this spec we'll deploy
  during overcloud deployment but we will aim to deliver this so that it
  is compatible with "deployed ceph". VIPs are provisioned with 
  `openstack overcloud network vip provision` before
  `openstack overcloud network provision` and before
  `openstack overcloud node provision` so we would have an ingress VIP in
  advance so we could do this with "deployed ceph".

* directord/task-core: We will ultimately need this implemented for the
  directord/task-core tool but could start with ansible tasks added to
  the tripleo_ceph role. Depending on the state of the directord/task-core
  migration when we implement we might skip the ansible part, though we
  could POC with it to get started.

Developer Impact
----------------

Assuming the manila services are able to interact with Ganesha using the
watch_url mechanism, the NFS daemon can be generated as a regular Ceph
daemon using the spec approach provided by the tripleo-ansible module [4]_.

Implementation
==============

Deployment Flow
---------------

The deployment and configuration described in this spec will happen during
`openstack overcloud deploy`, as described in [8]_.
This is consistent with how tripleo-ansible used to run during step2 to
configure these services. The tripleo-ansible tasks should be moved from a
pure ansible templating approach that generates the systemd unit according
to the input provided to a cephadm based daemon that can be configured with
the usual Ceph mgr config-key mechanism.
As described in the overview section, an ingress object will be defined and
deployed and this is supposed to manage both the VIP and the HA for this
component.

Assignee(s)
-----------

- fmount
- fultonj
- gfidente

Work Items
----------

- Change the tripleo-ansible module to support the Ceph ingress daemon
  type
- Create a set of tasks to deploy both the nfs and the related ingress
  daemons
- Deprecate the pacemaker related configuration for ceph-nfs, including
  pacemaker constraints between the manila-share service and ceph-nfs
- Create upgrade playbooks to transition from TripleO/pcmk managed nfs
  ganesha to nfs daemons deployed by cephadm and managed by ceph orch

Dependencies
============

- This work depends on the manila spec [10]_ that moves from dbus to the
  watch_url approach

Testing
=======

The NFS daemon feature can be enabled at day1 and it will be tested against
the existing TripleO scenario004 [9]_.
As part of the implementation plan, the update of the existing heat templates
environment CI files, which contain the testing job parameters, is one of the
goals of this spec.
An important aspect of the job definition process is related to standalone vs
multinode.
As seen in the past, multinode can help catching issues that are not visible
in a standalone environment, but of course the job configuration can be improved
in the next cycles, and we can start with standalone testing, which is what is
present today in CI.

Documentation Impact
====================

No changes should be necessary to the TripleO documentation, as the described
interface remains the unchanged.
However, we should provide upgrade instructions for pre existing environments
that need to transition from TripleO/pcmk managed nfs ganesha to nfs daemons
deployed by cephadm and managed by ceph orch.

References
==========

.. [1] `cephadm <https://github.com/ceph/ceph/tree/master/src/cephadm>`_
.. [2] `tripleo-ceph <https://specs.openstack.org/openstack/tripleo-specs/specs/wallaby/tripleo-ceph.html>`_
.. [3] `tripleo-ceph-ganesha <https://github.com/openstack/tripleo-ansible/tree/master/tripleo_ansible/roles/tripleo_cephadm/tasks/ganesha>`_
.. [4] `tripleo-ceph-mkspec <https://github.com/openstack/tripleo-ansible/blob/master/tripleo_ansible/ansible_plugins/modules/ceph_mkspec.py>`_
.. [5] `tripleo-ceph-nfs <https://docs.ceph.com/en/latest/cephfs/fs-nfs-exports>`_
.. [6] `ganesha-watch_url <https://github.com/nfs-ganesha/nfs-ganesha/blob/next/src/config_samples/ceph.conf#L206>`_
.. [7] `cephadm-nfs-ingress <https://docs.ceph.com/en/pacific/cephadm/nfs/#high-availability-nfs>`_
.. [8] `tripleo-cephadm <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/cephadm.html>`_
.. [9] `tripleo-scenario004 <https://github.com/openstack/tripleo-heat-templates/blob/master/ci/environments/scenario004-standalone.yaml>`_
.. [10] `cephfs-nfs-drop-dbus <https://blueprints.launchpad.net/manila/+spec/cephfs-nfs-drop-dbus>`_
.. [11] `cephfs-get-config <https://github.com/ceph/ceph/pull/43504>`_

