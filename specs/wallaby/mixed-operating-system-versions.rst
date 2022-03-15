..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Mixed Operating System Versions
===============================

https://blueprints.launchpad.net/tripleo/+spec/mixed-operating-system-versions

This spec proposes that a single TripleO release supports multiple operating
system versions.

Problem Description
===================

Historically a single branch or release of TripleO has supported only a single
version of an operating system at a time. In the past, this has been specific
versions of Ubuntu or Fedora in the very early days, and now has standardized
on specific versions of CentOS Stream.

In order to upgrade to a later version of OpenStack, it involves first
upgrading the TripleO undercloud, and then upgrading the TripleO overcloud to
the later version of OpenStack. The problem with supporting only a single
operating system version at a time is that such an OpenStack upgrade typically
implies an upgrade of the operating system at the same time. Combining the
OpenStack upgrade with a simultaneous operating system upgrade is problematic
due to:

1. Upgrade complexity
2. Upgrade time resulting in extended maintenance windows
3. Operating system incompatibilities with running workloads (kernel, libvirt,
   KVM, qemu, OVS/OVN, etc).
4. User impact of operating system changes (docker vs. podman, network-scripts
   vs. NetworkManager, etc).

Proposed Change
===============

Overview
--------

This spec proposes that a release of TripleO support 2 major versions of an
operating system, particularly CentOS Stream. A single release of TripleO
supporting two major versions of CentOS Stream will allow for an OpenStack
upgrade while remaining on the same operating version.

There are multiple software versions in play during an OpenStack upgrade:

:TripleO:
  The TripleO version is the version of the TripleO related packages installed
  on the undercloud. While some other OpenStack software versions are used here
  (Ironic, Neutron, etc), for the purposes of this spec, all TripleO and
  OpenStack software on the undercloud will be referred to as the TripleO
  version. The TripleO version corresponds to an OpenStack version.
  Examples: Train, Wallaby, Zed.

:OpenStack:
  The OpenStack version is the version of OpenStack on the overcloud that is
  being managed by the TripleO undercloud.
  Examples: Train, Wallaby, Zed.

:Operating System:
  The operating system version is the version of CentOS Stream. Both the
  undercloud and overcloud have operating system versions. The undercloud and
  the overcloud may not have the same operating system version, and all nodes
  in the overcloud may not have the same operating system version.
  Examples: CentOS Stream 8, 9, 10

:Container Image:
  The container image version is the version of the base container image used
  by tcib. This is a version of the Red Hat universal base image (UBI).
  Examples: UBI 8, 9, 10

For the purposes of this spec, the operating system versions being discussed
will be CentOS Stream 8 and 9, while the OpenStack versions will be Train and
Wallaby. However, the expectation is that TripleO continues to support 2
operating system versions with each release going forward. Subsequently, the
Zed. release of TripleO would support CentOS Stream 9 and 10.

With the above version definitions and considerations in mind, a TripleO
managed upgrade from Train to Wallaby would be described as the following:

#. Upgrade the undercloud operating system version from CentOS Stream 8 to 9.
#. Upgrade the undercloud TripleO version from Train to Wallaby.

   #. The Wallaby version of the TripleO undercloud will only run on CentOS Stream
      9.
   #. Implies upgrading all TripleO and OpenStack software on the undercloud to
      Wallaby.

#. Upgrade the OpenStack version on the overcloud from Train to Wallaby

   #. Does not imply upgrading the operating system version from CentOS Stream 8
      to 9.
   #. Implies upgrading to new container image versions that are the images for
      OpenStack Wallaby. These container image versions will likely be service
      dependent. Some services may use UBI version 9, while some may remain on UBI
      version 8.

#. Upgrade the operating system version on the overcloud nodes from CentOS
   Stream 8 to 9.

   #. Can happen node by node, with given constraints that might include all
      control plane nodes need to be upgraded at the same time.
   #. Data plane nodes could be selectively upgraded.

The default behavior will be that users and operators can choose to upgrade to
CentOS Stream 9 separately from the OpenStack upgrade. For those operators who
want a combined OpenStack and operating system upgrade to match previous FFU
behavior, they can perform both upgrades back to back. The OpenStack and
operating system upgrades will be separate processes. There may be UX around
making the processes appear as one, but that is not prescribed by this spec.

New TripleO deployments can choose either CentOS Stream 8 or 9 for their
Overcloud operating system version.

The implication with such a change is that the TripleO software needs to know
how to manage OpenStack on different operating system versions. Ansible roles,
puppet modules, shell scripts, etc, all need to remove any assumptions about a
given operating system and be developed to manage both CentOS Stream 8 and 9.
This includes operating system utilities that may function quite differently
depending on the underlying version, such as podman and container-tools.

CentOS Stream 8 support could not be dropped until the Zed. release of TripleO,
at which time, support would be needed for CentOS Stream 9 and 10.

Alternatives
------------

:Alternative 1:
  The TripleO undercloud Wallaby version could support running on both CentOS
  Stream 8 and 9. There does not seem to be much benefit in supporting both.
  Some users may refuse to introduce 9 into their environments at all, but
  TripleO has not encountered similar resistance in the past.

:Alternative 2:
  When upgrading the overcloud to the OpenStack Wallaby version, it could be
  required that all control plane nodes go through an operating system upgrade
  as well. Superficially, this appears to reduce the complexity of the
  development and test matrix. However, given the nature of composable roles,
  this requirement would really need to be prescribed per-service, and not
  per-role. Enforcing such a requirement would be problematic given the
  flexibility of running any service on any role. It would instead be better
  that TripleO document what roles need to be upgraded to a newer operating
  system version at the same time, by documenting a set of already provided
  roles or services. E.g., all nodes running a pacemaker managed service need
  to be upgraded to the same operating system version at the same time.

:Alternative 3:
  A single container image version could be used for all of OpenStack Wallaby. In
  order to support running those containers on both CentOS Stream 8 and 9, the
  single UBI container image would likely need to be 8, as anticipated support
  statements may preclude support for running UBI 9 images on 8.

:Alternative 4:
  New deployments could be forced to use CentOS Stream 9 only for their
  overcloud operating system version. However, some users may have workloads
  that have technical or certification requirements that could require CentOS
  Stream 8.

Security Impact
---------------

None.

Upgrade Impact
--------------

This proposal is meant to improve the FFU process by separating the OpenStack
and operating system upgrades.

Most users and operators will welcome this change. Some may prefer the old
method which offered a more simultaneous and intertwined upgrade. While the new
process could be implemented in such a way to offer a similar simultaneous
experience, it will still be different and likely appear as 2 distinct steps.

Distinct steps should result in an overall simplification of the upgrade
process.

Other End User Impact
---------------------

None.

Performance Impact
------------------

The previous implementations of FFU had the OpenStack and operating system
upgrades intertwined in the way that they were performed.  With the separation
of the upgrade processes, the overall upgrade of both OpenStack and the
operating system may take a longer amount of time overall. Operators would need
to plan for longer maintenance windows in the cases where they still want to
upgrade both during the same windows.

Otherwise, operators can choose to upgrade just OpenStack first, and then the
operating system at a later date, resulting in multiple, but shorter,
maintenance windows.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

TripleO developers will need support managing OpenStack software across
multiple operating system versions.

Service developers responsible for TripleO integrations, will need to decide
upgrade requirements around their individual services when it comes to
container image versions and supporting different operating system versions.

Given that the roll out of CentOS Stream 9 support in TripleO has happened in a
way that overlaps with supporting 8, it is largely true today that TripleO
Wallaby already supports both 8 and 9. CI jobs exist that test Wallaby on both
8 and 9. Going forward, that needs to remain true.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Work Items
----------

1. tripleo-ansible - CentOS Stream 8 and 9 support
2. tripleo-heat-templates - CentOS Stream 8 and 9 support
3. puppet-tripleo - CentOS Stream 8 and 9 support
4. puppet-* - CentOS Stream 8 and 9 support
5. tcib - build right container image versions per service


Dependencies
============

* CentOS Stream 9 builds will be required to fully test and develop

Testing
=======

FFU is not typically tested in upstream CI. However, CI will be needed that
tests deploying OpenStack Wallaby on both CentOS Stream 8
and 9 in order to verify that TripleO Wallaby is compatible with both operating
system versions.


Documentation Impact
====================

The matrix of supported versions will need to be documented within
tripleo-docs.

References
==========

None.
