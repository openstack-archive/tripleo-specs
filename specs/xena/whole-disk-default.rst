..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Deploy whole disk images by default
===================================

https://blueprints.launchpad.net/tripleo/+spec/whole-disk-default

This blueprint tracks the tasks required to switch to whole-disk overcloud
images by default instead of the current overcloud-full partition image.

Whole disk images vs partition images
=====================================

The current overcloud-full partition image consists of the following:

* A compressed qcow2 image file which contains a single root partition with
  all the image contents

* A kernel image file for the kernel to boot

* A ramdisk file to boot with the kernel

Whereas the overcloud-hardened-uefi-full whole-disk image consists of a single
compressed qcow2 image containing the following:

* A partition layout containing UEFI boot, legacy boot, and a root partition

* The root partition contains a single lvm group with a number of logical
  volumes of different sizes which are mounted at /, /tmp, /var, /var/log, etc.

When a partition image is deployed, ironic-python-agent does the following on
the baremetal disk being deployed to:

* Creates the boot and root partitions on the disk

* Copies the partition image contents to the root partition

* Populates the empty boot partition with everything required to boot, including
  the kernel image, ramdisk file, a generated grub config, and an installed
  grub binary

When a whole-disk image is deployed, ironic-python-agent simply copies the whole
image to the disk.

When the partition image deploy boots for the first time, the root partition
grows to take up all of the available disk space. This mechanism is provided
by the base cloud image. There is no equivalent partition growing mechanism
for a multi-volume LVM whole-disk image.

Problem Description
===================

The capability to build and deploy a whole-disk overcloud image has been
available for many releases, but it is time to switch to this as the default.
Doing this will avoid the following issues and bring the following benefits:

* As of CentOS-8.4, grub will stop support for installing the bootloader on a
  UEFI system. ironic-python-agent depends on grub installs to set up EFI boot
  with partition images, so UEFI boot will stop working when CentOS 8.4 is
  used.

* Other than this new grub behaviour, keeping partition boot working in
  ironic-python-agent has been a development burden and involves code
  complexity which is avoided for whole-disk deployments.

* TripleO users are increasingly wanting to deploy with UEFI Secure Boot
  enabled, this is only possible with whole-disk images that use the signed
  shim bootloader.

* Partition images need to be distributed with kernel and ramdisk files, adding
  complexity to file management of deployed images compared to a single
  whole-disk image file.

* The `requirements for a hardened image`_ includes having separate volumes for
  root, data etc. All TripleO users get the security benefit of hardened images
  when a whole-disk image is used.

* We currently need dedicated CI jobs both in the upstream check/gate (when the
  relevant files changed) but also in periodic integration lines, to build and
  publish the latest 'current-tripleo' version of the hardened images. In the long
  term, only a single hardend UEFI whole-disk image needs to be built and
  published, reducing the CI footprint. (in the short term, CI footprint may go up
  so the whole-disk image can be published, and while hardened vs hardened-uefi
  jobs are refactored.

Proposed Change
===============

Overview
--------

Wherever the partition image overcloud-full.qcow2 is built, published, or used
needs to be updated to use overcloud-hardened-uefi-full.qcow2 by default.

This blueprint will be considered complete when it is possible to follow the
default path in the documentation and the result is an overcloud deployed
with whole-disk images.

Image upload tool
+++++++++++++++++

The default behaviour of ``openstack overcloud image upload`` needs to be
aware that overcloud-hardened-uefi-full.qcow2 should be uploaded by default
when it is detected in the local directory.

Reviewing image build YAML
++++++++++++++++++++++++++

Once the periodic jobs are updated, image YAML defining
overcloud-hardened-full can be deleted, leaving only
overcloud-hardened-uefi-full. Other refactoring can be done such as renaming
-python3.yaml back to -base.yaml.

Reviewing partition layout
++++++++++++++++++++++++++

Swift data is stored in ``/srv`` and according to the criteria of hardened
images this should be in its own partition. This will need to be added to the
existing partition layout for whole-disk UEFI images.

Partition growing
+++++++++++++++++

On node first boot, a replacement mechanism for growing the root partition is
required. This is a harder problem for the multiple LVM volumes which the
whole-disk image creates. Generally the ``/var`` volume should grow to take
available disk space because this is where TripleO and OpenStack services store
their state, but sometimes ``/srv`` will need to grow for Swift storage, and
sometimes there may need to be a proportional split of multiple volumes. This
suggests that there will be new tripleo-heat-templates variables which will
specify the volume/proportion growth behaviour on a per-role basis.

A new utility is required which automates this LVM volume growing
requirement. It could be implemented a number of ways:

1. A new project/package containing the utility, installed on the image and
   run by first-boot or early tripleo-ansible.

2. A utility script installed by a diskimage-builder/tripleo-image-elements
   element and run by first-boot or as a first-boot ansible task (post-provisioning
   or early deploy).

3. Implement entirely in an ansible role, either in its own repository, or as
   part of tripleo-ansible. It would be run by early tripleo-ansible.

This utility will also be useful to other cloud workloads which use LVM based
images, so some consideration is needed for making it a general purpose tool
which can be used outside an overcloud image. Because of this, option 2. is
proposed initially as the preferred way to install this utility, and it will
be proposed as a new element in diskimage-builder. Being coupled with
diskimage-builder means the utility can make assumptions about the partition
layout:

* a single Volume Group that defaults to name ``vg``

* volume partitions are formatted with XFS, which can be resized while mounted

Alternatives
------------

Because of the grub situation, the only real alternative is dropping support
for UEFI boot, which means only supporting legacy BIOS boot indefinitely.
This would likely have negative feedback from end-users.

Security Impact
---------------

* All deployments will use images that comply with the hardened-image
  requirements, so deployments will gain these security benefits

* Whole disk images are UEFI Secure Boot enabled, so this blueprint brings us
  closer to recommending that Secure Boot be switched on always. This will
  validate to users that they have deployed boot/kernel binaries signed by Red
  Hat.

Upgrade Impact
--------------

Nodes upgraded in-place will continue to be partition image based, and
new/replaced nodes will be deployed with whole-disk images. This doesn't have
a specific upgrade implication, unless we document an option for replacing
every node in order to ensure all nodes are deployed with whole-disk images.

Other End User Impact
---------------------

There is little end-user impact other than:

* The change of habit required to use overcloud-hardened-uefi-full.qcow2
  instead of overcloud-full.qcow2

* The need to set the heat variable if custom partition growing behaviour is
  required

Performance Impact
------------------

There is no known performance impact with this change.

Other Deployer Impact
---------------------

All deployer impacts have already been mentioned elsewhere.

Developer Impact
----------------

There are no developer impacts beyond the already mentioned deployer impacts.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Steve Baker <sbaker@redhat.com>

Work Items
----------

* python-tripleoclient: image upload command, handle
  overcloud-hardened-uefi-full.qcow2 as the default if it exists locally

* tripleo-ansible/cli-overcloud-node-provision.yaml: detect
  overcloud-hardened-uefi-full.(qcow2|raw) as the default if it exists in
  /var/lib/ironic/images

* RDO jobs:
  * add periodic job for overcloud-hardened-uefi-full
  * remove periodic job for overcloud-hardened-full
  * modify image publishing jobs to publish overcloud-hardened-uefi-full.qcow2

* tripleo-image-elements/overcloud-partition-uefi: add ``/srv`` logical volume
  for swift data

* tripleo-quickstart-extras: Use the whole_disk_images=True variable to switch to
  downloading/uploading/deploying overcloud-hardened-uefi-full.qcow2

* tripleo-ci/featureset001/002: Enable whole_disk_images=True

* diskimage-builder: Add new element which installs utility for growing LVM
  volumes based on specific volume/proportion mappings

* tripleo-common/image-yaml:
  * refactor to remove non-uefi hardened image
  * rename -python3.yaml back to -base.yaml
  * add the element which installs the grow partition utility

* tripleo-heat-templates: Define variables for driving partition growth
  volume/proportion mappings

* tripleo-ansible: Consume the volume/proportion mapping and run the volume
  growing utility on every node in early boot.

* tripleo-docs:
  * Update the documentation for deploying whole-disk images by default
  * Document variables for controlling partition growth

Dependencies
============

Unless diskimage-builder require separate tracking to add the partition
growth utility, all tasks can be tracked under this blueprint.

Testing
=======

Image building and publishing
-----------------------------

Periodic jobs which build images, and jobs which build and publish images to
downloadable locations need to be updated to build and publish
overcloud-hardened-uefi-full.qcow2. Initially this can be in parallel with
the existing overcloud-full.qcow2 publishing, but eventually that can be
switched off.

overcloud-hardened-full.qcow2 is the same as
overcloud-hardened-uefi-full.qcow2 except that it only supports legacy BIOS
booting. Since overcloud-hardened-uefi-full.qcow2 supports both legacy BIOS
and UEFI boot, the periodic jobs which build overcloud-hardened-full.qcow2
can be switched off from Wallaby onwards (assuming these changes are backported
as far back as Wallaby).

CI support
----------

CI jobs which consume published images need to be modified so they can
download overcloud-hardened-uefi-full.qcow2 and deploy it as a whole-disk
image.

Documentation Impact
====================

The TripleO Deployment Guide needs to be modified so that
overcloud-hardened-uefi-full.qcow2 is referred to throughout, and so that it
correctly documents deploying a whole-disk image based overcloud.

References
==========

.. _requirements for a hardened image: https://teknoarticles.blogspot.com/2017/07/build-and-use-security-hardened-images.html
