..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
TripleO Ceph Ingress Daemon Integration
===========================================

Starting in the Octopus release, Ceph introduced  its own day1 tool called
cephadm and its own day2 tool called orchestrator which replaced ceph-ansible.
During the Wallaby and Xena cycles TripleO moved away from ceph-ansible and
adopted cephadm [1]_ as described in [2]_.
During Xena cycle a new approach of deploying Ceph in a TripleO context has
been established and now a Ceph cluster can be provisioned before the overcloud
is created, leaving to the overcloud deployment phase the final configuration
of the Ceph cluster which depends on the OpenStack enabled services defined by
the tripleo-heat-templates interface.
The next goal is to deploy as many Ceph services as possible using the deployed
ceph interface instead of during overcloud deployment.
As part of this effort, we should pay attention to the high-availability aspect,
how it's implemented in the current release and how it should be changed for
Ceph.
This spec represents a follow up of [3]_, it defines the requirements to rely
on the Ceph provided HA daemons and describes the changes required in TripleO
to meet this goal.

Problem Description
===================

In the following description we are referring to the Ganesha daemon and the
need of the related Ceph Ingress daemon deployment, but the same applies to
all the existing daemons that requires an high-availability configuration
(e.g., RGW and the Ceph dashboard for the next Ceph release).
In TripleO we support deployment of Ganesha both when the Ceph cluster is
itself managed by TripleO and when the Ceph cluster is itself not managed by
TripleO.
When the cluster is managed by TripleO, as per spec [3]_, it is preferable to
have cephadm manage the lifecycle of the NFS container instead of deploying it
with tripleo-ansible, and this is broadly covered and solved by allowing the
tripleo Ceph mkspec module to support the new Ceph daemon [4]_.
The ceph-nfs daemon deployed by cephadm has its own HA mechanism, called
ingress, which is based on haproxy and keepalived [5]_ so we would no longer
use pcmk as the VIP owner.
This means we would run pcmk and keepalived in addition to haproxy (deployed by
tripleo) and another haproxy (deployed by cephadm) on the same server (though
with listeners on different ports).
This approach only relies on Ceph components, and both external and internal
scenarios are covered.
However, adopting the ingress daemon for a TripleO deployed Ceph cluster means
that we need to make the overcloud aware about the new running services: for
this reason the proposed change is meant to introduce a new TripleO resource
that properly handles the interface with the Ceph services and is consistent
with the tripleo-heat-templates roles.

Proposed Change
===============

Overview
--------

The change proposed by this spec requires the introduction of a new TripleO
Ceph Ingress resource that describes the ingress service that provides load
balancing and HA.
The impact of adding a new `OS::TripleO::Services::CephIngress` resource can
be seen on the following projects.


tripleo-common
--------------

As described in Container Image Preparation [6]_ the undercloud may be used as
a container registry for all the ceph related containers and a new, supported
syntax, has been introduced to `deployed ceph` to download containers from
authenticated registries.
However, as per [7]_, the Ceph ingress daemons won’t be baked into the Ceph
daemon container, hence `tripleo container image prepare` should be executed to
pull the new container images/tags in the undercloud as made for the Ceph
Dashboard and the regular Ceph image.
Once the ingress containers are available, it's possible to deploy the daemon
on top of ceph-nfs or ceph-rgw.
In particular, if this spec is going to be implemented, `deployed ceph` will be
the only way of setting up this daemon through cephadm for ceph-nfs, resulting
in a simplified tripleo-heat-templates interface and a less number of tripleo
ansible tasks execution because part of the configuration is moved before the
overcloud is deployed.
As part of this effort, considering that the Ceph related container images have
grown over the time, a new condition will be added to the tripleo-container jinja
template [8]_ to avoid pulling additional ceph images if Ceph is not deployed by
TripleO [10]_.
This will result in a new optimization for all the Ceph external cluster use cases,
as well as the existing CI jobs without Ceph.

tripleo-heat-templates
----------------------
A Heat resource will be created within the cephadm space. The new resource will
be also added to the existing Controller roles and all the relevant environment
files will be updated with the new reference.
In addition, as described in the spec [3]_, pacemaker constraints for ceph-nfs
and the related vip will be removed.
The tripleo-common ceph_spec library is already able to generate the spec for
this kind of daemon and it will trigger cephadm [4]_ to deploy an ingress daemon
provided that the NFS Ceph spec is applied against an existing cluster and the
backend daemon is up and running.
As mentioned before, the ingress daemon can also be deployed on top of an RGW
instance, therefore the proposed change is valid for all the Ceph services that
require an HA configuration.


Security Impact
---------------

The ingress daemon applied to an existing ceph-nfs instance is managed by
cephadm, resulting in a simplified model in terms of lifecycle. A Ceph spec for
the ingress daemon is generated right after the ceph-nfs instance is applied,
and as per [5]_ it requires two additional options:

* frontend_port
* monitoring_port

The two ports are required by haproxy to accept incoming requests and for
monitoring purposes, hence we need to make TripleO aware about this new service
and properly setup the firewall rules. As long as the ports defined by the spec
are passed to the overcloud deployment process and defined in the
tripleo-heat-templates CephIngress daemon resource, the `firewall_rules`
tripleo ansible role is run and rules are applied for both the frontend and
monitoring port. The usual network used by this daemon (and affected by the new
applied rules) is the `StorageNFS`, but we might have cases where an operator
overrides it.
The lifecycle, builds and security aspects for the container images associated
to the CephIngress resource are not managed by TripleO, and the Ceph
organization takes care about maintanance and updates.



Upgrade Impact
--------------

The problem of an existing Ceph cluster is covered by the spec [8]_.


Performance Impact
------------------

Since two new images (and the equivalent tripleo-heat-templates services) have
been introduced, some time is required to pull these new additional containers
in the undercloud. However, the tripleo_containers jinja template has been
updated, splitting off the Ceph related container images. In particular, during
the containers image prepare phase, a new boolean option has been added and
pulling the Ceph images can be avoided by setting the `ceph_images` boolean to
false. By doing this we can improve performances when Ceph is not required.

Developer Impact
----------------
This effort can be easily extended to move the RGW service to deployed ceph,
which is out of scope of this spec.

Implementation
==============

Deployment Flow
---------------

The deployment and configuration described in this spec will happen during
`openstack overcloud ceph deploy`, as described in [8]_.
The current implementation of `openstack overcloud network vip provision`
allows to provision 1 vip per network, which means that using the new Ceph
Ingress daemon (that requires 1 vip per service) can break components that
are still using the VIP provisioned on the storage network (or any other
network depending on the tripleo-heat-templates override specified) and
are managed by pacemaker.
A new option `--ceph-vip` for `openstack overcloud ceph deploy` command
will be added [11]_. This option may be used to reserve VIP(s) for each
Ceph service specified by the 'service/network' mapping defined as input.
For instance, a generic ceph service mapping can be something like the
following::

  ---
  ceph_services:
    - service: ceph_nfs
      network: storage
    - service: ceph_rgw
      network: storage

For each service added to the list above, a virtual ip on the specified
network (that can be a composable network) will be created and used as
frontend_vip of the ingress daemon.
As described in the overview section, an ingress object will be defined
and deployed and this is supposed to manage both the VIP and the HA for
this component.

Assignee(s)
-----------

- fmount
- fultonj
- gfidente

Work Items
----------

- Create a new Ceph prefixed Heat resource that describes the Ingress daemon
  in the TripleO context.
- Add both haproxy and keepalived containers to the Ceph container list so that
  they can be pulled during the `Container Image preparation` phase.
- Create a set of tasks to deploy both the nfs and the related ingress
  daemon
- Deprecate the pacemaker related configuration for ceph-nfs, including
  pacemaker constraints between the manila-share service and ceph-nfs
- Create upgrade playbooks to transition from TripleO/pcmk managed nfs
  ganesha to nfs/ingress daemons deployed by cephadm and managed by ceph
  orch

Depending on the state of the directord/task-core migration we might skip the
ansible part, though we could POC with it to get started, extending the existing
tripleo-ansible cephadm role.

Dependencies
============

This work depends on the tripleo_ceph_nfs spec [3]_ that moves from tripleo
deployed ganesha to the cephadm approach.

Testing
=======

The NFS daemon feature can be enabled at day1 and it will be tested against
the existing TripleO scenario004 [9]_.
As part of the implementation plan, the update of the existing heat templates
environment CI files, which contain both the Heat resources and the testing
job parameters, is one of the goals of this spec.


Documentation Impact
====================

The documentation will describe the new parameters introduced to the `deployed
ceph` cli to give the ability to deploy additional daemons (ceph-nfs and the
related ingress daemon) as part of deployed ceph.
However, we should provide upgrade instructions for pre existing environments
that need to transition from TripleO/pcmk managed nfs ganesha to nfs daemons
deployed by cephadm and managed by ceph orch.


References
==========

.. [1] `cephadm <https://github.com/ceph/ceph/tree/master/src/cephadm>`_
.. [2] `tripleo-ceph <https://specs.openstack.org/openstack/tripleo-specs/specs/wallaby/tripleo-ceph.html>`_
.. [3] `tripleo-nfs-spec <https://specs.openstack.org/openstack/tripleo-specs/specs/yoga/tripleo_ceph_manila.html>`_
.. [4] `tripleo-ceph-mkspec <https://review.opendev.org/c/openstack/tripleo-ansible/+/818786>`_
.. [5] `cephadm-nfs-ingress <https://docs.ceph.com/en/pacific/cephadm/nfs/#high-availability-nfs>`_
.. [6] `container-image-preparation <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/container_image_prepare.html>`_
.. [7] `ceph-ingress-containers <https://github.com/ceph/ceph/blob/master/src/cephadm/cephadm#L55-L56>`_
.. [8] `tripleo-common-j2 <https://github.com/openstack/tripleo-common/blob/master/container-images/tripleo_containers.yaml.j2>`_
.. [9] `tripleo-scenario004 <https://github.com/openstack/tripleo-heat-templates/blob/master/ci/environments/scenario004-standalone.yaml>`_
.. [10] `tripleo-common-split-off <https://review.opendev.org/c/openstack/tripleo-common/+/824431>`_
.. [11] `tripleo-ceph-vip <https://review.opendev.org/q/topic:%22ceph_vip_provision%22+(status:open%20OR%20status:merged)>`_
