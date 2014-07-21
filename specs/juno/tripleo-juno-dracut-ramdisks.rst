..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Dracut Deploy Ramdisks
======================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-dracut-ramdisks

Our current deploy ramdisks include functionality that is duplicated from
existing tools such as Dracut, and do not include some features that those
tools do.  Reimplementing our deploy ramdisks to use Dracut would shrink
our maintenance burden for that code and allow us to take advantage of those
additional features.

Problem Description
===================

Currently our deploy ramdisks are implemented as a bash script that runs
as init during the deploy process.  This means that we are responsible for
correctly configuring things such as udev and networking which would normally
be handled by distribution tools.  While this isn't an immediate problem
because the implementation has already been done, it is an unnecessary
duplication and additional maintenance debt for the future as we need to add
or change such low-level functionality.

In addition, because our ramdisk is a one-off, users will not be able to make
use of any ramdisk troubleshooting methods that they might currently know.
This is an unnecessary burden when there are tools to build ramdisks that are
standardized and well-understood by the people using our software.

Proposed Change
===============

The issues discussed above can be dealt with by using a standard tool such as
Dracut to build our deploy ramdisks.  This will actually result in a reduction
in code that we have to maintain and should be compatible with all of our
current ramdisks because we can continue to use the same method of building
the init script - it will just run as a user script instead of process 0,
allowing Dracut to do low-level configuration for us.

Initially this will be implemented alongside the existing ramdisk element to
provide a fallback option if there are any use cases not covered by the
initial version of the Dracut ramdisk.

Alternatives
------------

For consistency with the rest of Red Hat/Fedora's ramdisks I would prefer to
implement this using Dracut, but if there is a desire to also make use of
another method of building ramdisks, that could probably be implemented
alongside Dracut.  The current purely script-based implementation could even
be kept in parallel with a Dracut version.  However, I believe Dracut is
available on all of our supported platforms so I don't see an immediate need
for alternatives.

Additionally, there is the option to replace our dynamically built init
script with Dracut modules for each deploy element.  This is probably
unnecessary as it is perfectly fine to use the current method with Dracut,
and using modules would tightly couple our deploy ramdisks to Dracut, making
it difficult to use any alternatives in the future.

Security Impact
---------------

The same security considerations that apply to the current deploy ramdisk
would continue to apply to Dracut-built ones.

Other End User Impact
---------------------

This change would enable end users to make use of any Dracut knowledge they
might already have, including the ability to dynamically enable tracing
of the commands used to do the deployment (essentially set -x in bash).

Performance Impact
------------------

Because Dracut supports more hardware and software configurations, it is
possible there will be some additional overhead during the boot process.
However, I would expect this to be negligible in comparison to the time it
takes to copy the image to the target system, so I see it as a reasonable
tradeoff.

Other Deployer Impact
---------------------

As noted before, Dracut supports a wide range of hardware configurations,
so deployment methods that currently wouldn't work with our script-based
ramdisk would become available.  For example, Dracut supports using network
disks as the root partition, so running a diskless node with separate
storage should be possible.

Developer Impact
----------------

There would be some small changes to how developers would add a new dependency
to the ramdisk images.  Instead of executables and their required libraries
being copied to the ramdisk manually, the executable can simply be added to
the list of things Dracut will include in the ramdisk.

Developers would also gain the dynamic tracing ability mentioned above in
the end user impact.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bnemec

Work Items
----------

* Convert the ramdisk element to use Dracut (see WIP change in References).

* Verify that DHCP booting of ramdisks still works.

* Verify that nova-baremetal ramdisks can be built successfully with Dracut.

* Verify that Ironic ramdisks can be built successfully with Dracut.

* Verify that Dracut can build Ironic-IPA ramdisks.

* Verify the Dracut debug shell provides equivalent functionality to the
  existing one.

* Provide ability for other elements to install additional files to the
  ramdisk.

* Provide ability for other elements to include additional drivers.

* Find a way to address potential 32-bit binaries being downloaded and run in
  the ramdisk for firmware deployments.

Dependencies
============

This would add a dependency on Dracut for building ramdisks.

Testing
=======

Since building deploy ramdisks is already part of CI, this should be covered
automatically.  If it is implemented in parallel with another method, then
the CI jobs would need to be configured to exercise the different methods
available.

Documentation Impact
====================

We would want to document the additional features available in Dracut.
Otherwise this should function in essentially the same way as the current
ramdisks, so any existing documentation will still be valid.

Some minor developer documentation changes may be needed to address the
different ways Dracut handles adding extra kernel modules and files.

References
==========

* Dracut: https://dracut.wiki.kernel.org/index.php/Main_Page

* PoC of building ramdisks with Dracut:
  https://review.openstack.org/#/c/105275/

* openstack-dev discussion:
  http://lists.openstack.org/pipermail/openstack-dev/2014-July/039356.html
