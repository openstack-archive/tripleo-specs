..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
TripleO Deploy Cloud Hypervisor Type
====================================

# TODO: file the actual blueprint...
https://blueprints.launchpad.net/tripleo/+spec/tripleo-deploy-cloud-hypervisor-type

The goal of this spec is to detail how the TripleO deploy cloud type could be
varied from just baremetal to baremetal plus other hypervisors to deploy
Overcloud services.

Linux kernel containers make this approach attractive due to the lightweight
nature that services and process can be virtualized and isolated, so it seems
likely that libvirt+lxc and Docker would be likely targets. However we should
aim to make this approach as agnostic as possible for those deployers who may
wish to use any Nova driver, such as libvirt+kvm.

Problem Description
===================

The overcloud control plane is generally lightly loaded and allocation of
entire baremetal machines to it is wasteful. Also, when the Overcloud services
are running entirely on baremetal they take longer to upgrade and rollback.

Proposed Change
===============

We should support any Nova virtualization type as a target for Overcloud
services, as opposed to using baremetal nodes to deploy overcloud images.
Containers are particularly attractive because they are lightweight, easy to
upgrade/rollback and offer similar isolation and security as full VM's. For the
purpose of this spec, the alternate Nova virtualization target for the
Overcloud will be referred to as alt-hypervisor. alt-hypervisor could be
substituted with libvirt+lxc, Docker, libvirt+kvm, etc.

At a minimum, we should support running each Overcloud service in isolation in
its own alt-hypervisor instance in order to be as flexible as possible to deployer
needs. We should also support combining services.

In order to make other alt-hypervisors available as deployment targets for the
Overcloud, we need additional Nova Compute nodes/services configured to use
alt-hypervisors registered with the undercloud Nova.

Additionally, the undercloud must still be running a Nova compute with the
ironic driver in order to allow for scaling itself out to add additional
undercloud compute nodes.

To accomplish this, we can run 2 Nova compute processes on each undercloud
node.  One configured with Nova+Ironic and one configured with
Nova+alt-hypervisor.  For the straight baremetal deployment, where an alternate
hypervisor is not desired, the additional Nova compute process would not be
included. This would be accomplished via the standard inclusion/exclusion of
elements during a diskimage-builder tripleo image build.

It will also be possible to build and deploy just an alt-hypervisor compute
node that is registered with the Undercloud as an additional compute node.

To minimize the changes needed to the elements, we will aim to run a full init
stack in each alt-hypervisor instance, such as systemd. This will allow all the
services that we need to also be running in the instance (cloud-init,
os-collect-config, etc). It will also make troubleshooting similar to the
baremetal process in that you'd be able to ssh to individual instances, read
logs, restart services, turn on debug mode, etc.

To handle Neutron network configuration for the Overcloud, the Overcloud
neutron L2 agent will have to be on a provider network that is shared between
the hypervisors. VLAN provider networks will have to be modeled in Neutron and
connected to alt-hypervisor instances.

Overcloud compute nodes themselves would be deployed to baremetal nodes. These
images would be made up of:
* libvirt+kvm (assuming this is the hypervisor choice for the Overcloud)
* nova-compute + libvirt+kvm driver (registered to overcloud control).
* neutron-l2-agent (registered to overcloud control)
An image with those contents is deployed to a baremetal node via nova+ironic
from the undercloud.

Alternatives
------------

Deployment from the seed
^^^^^^^^^^^^^^^^^^^^^^^^
An alternative to having the undercloud deploy additional alt-hypervisor
compute nodes would be to register additional baremetal nodes with the seed vm,
and then describe an undercloud stack in a template that is the undercloud
controller and its set of alt-hypervisor compute nodes.  When the undercloud
is deployed via the seed, all of the nodes are set up initially.

The drawback with that approach is that the seed is meant to be short-lived in
the long term. So, it then becomes difficult to scale out the undercloud if
needed. We could offer a hybrid of the 2 models: launch all nodes initially
from the seed, but still have the functionality in the undercloud to deploy
more alt-hypervisor compute nodes if needed.

The init process
^^^^^^^^^^^^^^^^
If running systemd in a container turns out to be problematic, it should be
possible to run a single process in the container that starts just the
OpenStack service that we care about. However that process would also need to
do things like read Heat metadata. It's possible this process could be
os-collect-config. This change would require more changes to the elements
themselves however since they are so dependent on an init process currently in
how they enable/restart services etc. It may be possible to replace os-svc-*
with other tools that don't use systemd or upstart when you're building images
for containers.

Security Impact
---------------
* We should aim for equivalent security when deploying to alt-hypervisor
  instances as we do when deploying to baremetal. To the best of our ability, it
  should not be possible to compromise the instance if an individual service is
  compromised.

* Since Overcloud services and Undercloud services would be co-located on the
  same baremetal machine, compromising the hypervisor and gaining access to the
  host is a risk to both the Undercloud and Overcloud. We should mitigate this
  risk to the best of our ability via things like SELinux, and removing all
  unecessary software/processes from the alt-hypervisor instances.

* Certain hypervisors are inherently more secure than others. libvirt+kvm uses
  virtualization and is much more secure then container based hypervisors such as
  libvirt+lxc and Docker which use namespacing.

Other End User Impact
---------------------
None. The impact of this change is limited to Deployers. End users should have
no visibility into the actual infrastructure of the Overcloud.

Performance Impact
------------------
Ideally, deploying an overcloud to containers should result in a faster
deployment than deploying to baremetal. Upgrading and downgrading the Overcloud
should also be faster.

More images will have to be built via diskimage-builder however, which will
take more time.

Other Deployer Impact
---------------------
The main impact to deployers will be the ability to use alt-hypervisors
instances, such as containers if they wish. They also must understand how to
use nova-baremetal/ironic on the undercloud to scale out the undercloud and add
additional alt-hypervisor compute nodes if needed.

Additional space in the configured glance backend would also likely be needed
to store additional images.

Developer Impact
----------------
* Developers working on TripleO will have the option of deploying to
  alt-hypervisor instances.  This should make testing and developing on some
  aspects of TripleO easier due to the need for less vm's.

* More images will have to be built due to the greater potential variety with
  alt-hypervisor instances housing Overcloud services.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  james-slagle

Work Items
----------

tripleo-incubator
^^^^^^^^^^^^^^^^^
* document how to use an alternate hypervisor for the overcloud deployment
  ** eventually, this could possibly be the default
* document how to troubleshoot this type of deployment
* need a user option or json property to describe if the devtest
  environment being set up should use an alternate hypervisor for the overcloud
  deployment or not. Consider using HEAT_ENV where appropriate.
* load-image should be updated to add an additional optional argument that sets
  the hypervisor_type property on the loaded images in glance. The argument is
  optional and wouldn't need to be specified for some images, such as regular
  dib images that can run under KVM.
* Document commands to setup-neutron for modeling provider VLAN networks.

tripleo-image-elements
^^^^^^^^^^^^^^^^^^^^^^
* add new element for nova docker driver
* add new element for docker registry (currently required by nova docker
  driver)
* more hypervisor specific configuration files for the different nova compute
  driver elements
  ** /etc/nova/compute/nova-kvm.conf
  ** /etc/nova/compute/nova-baremetal.conf
  ** /etc/nova/compute/nova-ironic.conf
  ** /etc/nova/compute/nova-docker.conf
* Separate configuration options per compute process for:
  ** host (undercloud-kvm, undercloud-baremetal, etc).
  ** state_path (/var/lib/nova-kvm, /var/lib/nova-baremetal, etc).
* Maintain backwards compatibility in the elements by consulting both old and
  new heat metadata key namespaces.

tripleo-heat-templates
^^^^^^^^^^^^^^^^^^^^^^
* Split out heat metadata into separate namespaces for each compute process
  configuration.
* For the vlan case, update templates for any network modeling for
  alt-hypervisor instances so that those instances have correct interfaces
  attached to the vlan network.

diskimage-builder
^^^^^^^^^^^^^^^^^
* add ability where needed to build new image types for alt-hypervisor
  ** Docker
  ** libvirt+lxc
* Document how to build images for the new types

Dependencies
============
For Docker support, this effort depends on continued development on the nova
Docker driver. We would need to drive any missing features or bug fixes that
were needed in that project.

For other drivers that may not be as well supported as libvirt+kvm, we will
also have to drive missing features there as well if we want to support them,
such as libvirt+lxc, openvz, etc.

This effort also depends on the provider resource templates spec (unwritten)
that will be done for the template backend for Tuskar. That work should be done
in such a way that the provider resource templates are reusable for this effort
as well in that you will be able to create templates to match the images that
you intend to create for your Overcloud deployment.

Testing
=======
We would need a separate set of CI jobs that were configured to deploy an
Overcloud to each alternate hypervisor that TripleO intended to support well.

For Docker support specifically, CI jobs could be considered non-voting since
they'd rely on a stackforge project which isn't officially part of OpenStack.
We could potentially make this job voting if TripleO CI was enabled on the
stackforge/nova-docker repo so that changes there are less likely to break
TripleO deployments.

Documentation Impact
====================
We should update the TripleO specific docs in tripleo-incubator to document how
to use an alternate hypervisor for an Overcloud deployment.

References
==========
Juno Design Summit etherpad: https://etherpad.openstack.org/p/juno-summit-tripleo-and-docker
nova-docker driver: https://git.openstack.org/cgit/stackforge/nova-docker
Docker: https://www.docker.io/
Docker github: https://github.com/dotcloud/docker
