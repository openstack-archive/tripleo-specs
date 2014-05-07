..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
QuintupleO - TripleO on OpenStack
==========================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-on-openstack

This is intended as a new way to do a TripleO deployment in a virtualized
environment.  Rather than provisioning the target virtual machines directly
via virsh, we would be able to use the standard OpenStack apis to create and
manage the instances.  This should make virtual TripleO environments more
scalable and easier to manage.

Ultimately the goal would be to make it possible to do virtual TripleO
deployments on any OpenStack cloud, except where necessary features have
explicitly been disabled.  We would like to have the needed features
available on the public clouds used for OpenStack CI, so existing providers
are invited to review this specification.

Problem Description
===================

TripleO development and testing requires a lot of hardware resources, and
this is only going to increase as things like HA are enabled by default.
In addition, we are going to want to be able to test larger deployments than
will fit on a single physical machine.  While it would be possible to set
this up manually, OpenStack already provides services capable of managing
a large number of physical hosts and virtual machines, so it doesn't make
sense to reinvent the wheel.

Proposed Change
===============

* Write a virtual power driver for OpenStack instances.  I already have a
  rough version for nova-baremetal, but it needs a fair amount of cleaning up
  before it could be merged into the main codebase.  We will also need to
  work with the Ironic team to enable this functionality there.

* Determine whether changes are needed in Neutron to allow us to run our own
  DHCP server, and if so work with the Neutron team to make those changes.
  This will probably require allowing an instance to be booted without any
  ip assigned.  If not, booting an instance without an IP would be a good
  future enhancement to avoid wasting IP quota.

* Likewise, determine how to use virtual ips with keepalived/corosync+pacemaker
  in Neutron, and if changes to Neutron are needed work with their team to
  enable that functionality.

* Enable PXE booting in Nova.  There is already a bug open to track this
  feature request, but it seems to have been abandoned.  See the link in the
  References section of this document.  Ideally this should be enabled on a
  per-instance basis so it doesn't require a specialized compute node, which
  would not allow us to run on a standard public cloud.

* For performance and feature parity with the current virtual devtest
  environment, we will want to be allow the use of unsafe caching for the
  virtual baremetal instances.

* Once all of the OpenStack services support this use case we will want to
  convert our CI environment to a standard OpenStack KVM cloud, as well as
  deprecate the existing method of running TripleO virtually and enable
  devtest to install and configure a local OpenStack installation (possibly
  using devstack) on which to run.

* Depending on the state of our container support at that time, we may want
  to run the devtest OpenStack using containers to avoid taking over the host
  system the way devstack normally does.  This may call for its own spec when
  we reach that point.

Alternatives
------------

* There's no real alternative to writing a virtual power driver.  We have to
  be able to manage OpenStack instances as baremetal nodes for this to work.

* Creating a flat Neutron network connected to a local bridge can address the
  issues with Neutron not allowing DHCP traffic, but that only works if you
  have access to create the local bridge and configure the new network.  This
  may not be true in many (all?) public cloud providers.

* I have not done any work with virtual IP addresses in Neutron yet, so it's
  unclear to me whether any alternatives exist for that.

* As noted earlier, using an iPXE image can allow PXE booting of Nova
  instances.  However, because that image is overwritten during the deploy,
  it is not possible to PXE boot the instance afterward.  Making the TripleO
  images bootable on their own might be an option, but it would diverge from
  how a real baremetal environment would work and thus is probably not
  desirable.

Deploy overcloud without PXE boot
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Since a number of the complications around doing TripleO development on an
OpenStack cloud relate to PXE booting the instances, one option that could
be useful in some situations is the ability to deploy images directly.  Since
we're using Heat for deployments, it should be possible to build the TripleO
images with the ``vm`` element and deploy them as regular instances instead of
fake baremetal ones.

This has the drawback of not exercising as much of the TripleO baremetal
functionality as a full virtual PXE boot process, but it should be easier to
implement, and for some development work not related to the deploy process
would be sufficient for verifying that a feature works as intended.  It might
serve as a good intermediate step while we work to enable full PXE boot
functionality in OpenStack clouds.

It would also prevent exercising HA functionality because we would likely not
be able to use virtual IP addresses if we can't use DHCP/PXE to manage our
own networking environment.

Security Impact
---------------

* The virtual power driver is going to need access to OpenStack
  credentials so it can control the instances.

* The Neutron changes to allow private networks to behave as flat networks
  may have security impacts, though I'm not exactly sure what they would be.
  The same applies to virtual IP support.

* PXE booting instances could in theory allow an attacker to override the
  DHCP server and boot arbitrary images, but in order to do that they would
  already need to have access to the private network being used, so I don't
  consider this a significant new threat.

Other End User Impact
---------------------

End users doing proof of concepts using a virtual deployment environment
would need to be switched to this new method, but that should be largely
taken care of by the necessary changes to devtest since that's what would
be used for such a deployment.

Performance Impact
------------------

In my testing, my OpenStack virtual power driver was significantly slower
than the existing virsh-based one, but I believe with a better implementation
that could be easily solved.

When running TripleO on a public cloud, a developer would be subject to the
usual limitations of shared hardware - a given resource may be oversubscribed
and cause performance issues for the processing or disk-heavy operations done
by a TripleO deployment.

Other Deployer Impact
---------------------

This is not intended to be visible to regular deployers, but it should
make our CI environment more flexible by allowing more dynamic allocation
of resources.

Developer Impact
----------------

If this becomes the primary method of doing TripleO development, devtest would
need to be altered to either point at an existing OpenStack environment or
to configure a local one itself.  This will have an impact on how developers
debug problems with their environment, but since they would be debugging
OpenStack in that case it should be beneficial in the long run.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bnemec

Other contributors:
  jang

Work Items
----------
* Implement an Ironic OpenStack virtual power driver.

* Implement a nova-baremetal OpenStack virtual power driver, probably out
  of tree based on the feedback we're getting from Nova and Ironic.

* Enable PXE booting of Nova instances.

* Enable unsafe caching to be enabled on Nova instances.

* Allow DHCP/PXE traffic on private networks in Neutron.

* If not already covered by the previous point, allow booting of instances
  without IP addresses.

* Migrate CI to use an OpenStack cloud for its virtual baremetal instances.

* Migrate devtest to install and configure an OpenStack cloud instead of
  managing instances and networking manually.

* To simplify the VM provisioning process, we should make it possible to
  provision but not boot a Nova VM.


Dependencies
============

The Ironic, Neutron, and Nova changes in the Work Items section will all have
to be done before TripleO can fully adopt this feature.


Testing
=======

* All changes in the other projects will be unit and functional tested as
  would any other new feature.

* We cannot test this functionality by running devstack to provision an
  OpenStack cloud in a gate VM, such as would be done for Tempest, because
  the performance of the nested qemu virtual machines would make the process
  prohibitively slow.  We will need to have a baremetal OpenStack deployment
  that can be targeted by the tests.  A similar problem exists today with
  virsh instances, however, and it can probably be solved in a similar
  fashion with dedicated CI environments.

* We will need to have Tempest tests gating on all the projects we use to
  exercise the functionality we depend on.  This should be largely covered
  by the functional tests for the first point, but it's possible we will find
  TripleO-specific scenarios that need to be added as well.


Documentation Impact
====================

devtest will need to be updated to reflect the new setup steps needed to run
it against an OpenStack-based environment.


References
==========

This is largely based on the discussion Devtest on OpenStack in
https://etherpad.openstack.org/p/devtest-env-reqs

Nova bug requesting PXE booting support:
https://bugs.launchpad.net/nova/+bug/1183885
