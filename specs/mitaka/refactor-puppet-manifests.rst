..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Refactor top level puppet manifests
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/refactor-puppet-manifests

The current overcloud controller puppet manifests duplicate a large amount
of code between the pacemaker (HA) and non-ha version. We can reduce the
effort required to add new features by refactoring this code, and since
there is already a puppet-tripleo module this is the logical destination.

Problem Description
===================

Large amounts of puppet/manifests/overcloud\_controller.pp are shared with
puppet/manifests/overcloud\_controller\_pacemaker.pp. When adding a feature
or fixing a mistake in the former, it is frequently also an issue in the
latter. It is a violation of the common programming principle of DRY, which
while not an inviolable rule, is usually considered good practice.

In addition, moving this code into separate classes in another module will
make it simpler to enable/disable components, as it will be a matter of
merely controlling which classes (profiles) are included.

Finally, it allows easier experimentation with modifying the 'ha strategy'.
Currently this is done using 'step', but could in theory be done using a
service registry. By refactoring into ha+non-ha classes this would be quite
simple to swap in/out.

Proposed Change
===============

Overview
--------

While there are significant differences in ha and non-ha deployments, in almost
all cases the ha code will be a superset of the non-ha. A simple example of
this is at the top of both files, where the load balancer is handled. The non
ha version simply includes the loadbalancing class, while the HA version
instantiates the exact same class but with some parameters changed. Across
the board the same classes are included for the openstack services, but with
manage service set to false in the HA case.

I propose first breaking up the non-ha version into profiles which can reside
in puppet-tripleo/manifests/profile/nonha, then adding ha versions which
use those classes under puppet-tripleo-manifests/profile/pacemaker. Pacemaker
could be described as an 'ha strategy' which in theory should be replaceable.
For this reason we use a pacemaker subfolder since one day perhaps we'll have
an alternative.

Alternatives
------------

We could leave things as they are, which works and isn't the end of the world,
but it's probably not optimal.

We could use kolla or something that removes the need for puppet entirely, but
this discussion is outside the scope of this spec.

Security Impact
---------------

None

Other End User Impact
---------------------

It will make downstreams happy since they can sub in/out classes more easily.

Performance Impact
------------------

Adding wrapper classes isn't going to impact puppet compile times very much.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Changes in t-h-t and puppet-tripleo will often be coupled, as t-h-t
defines the data on which puppet-tripleo depends on.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  michaeltchapman

Work Items
----------

Move overcloud controller to profile classes
Move overcloud controller pacemaker to profile classes
Move any other classes from the smaller manifests in t-h-t

Dependencies
============

None

Testing
=======

No new features so current tests apply in their entirety.
Additional testing can be added for each profile class

Documentation Impact
====================

None

References
==========

None
