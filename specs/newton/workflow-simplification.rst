..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Workflow Simplification
==========================================

https://blueprints.launchpad.net/tripleo/+spec/workflow-simplification

The TripleO workflow is still too complex for many (most?) users to follow
successfully.  There are some fairly simple steps we can take to improve
that situation.

Problem Description
===================

The current TripleO workflow grew somewhat haphazardly out of a collection
of bash scripts that originally made up instack-undercloud.  These scripts
started out life as primarily a proof of concept exercise to demonstrate
that the idea was viable, and while the steps still work fine when followed
correctly, it seems "when followed correctly" is too difficult today, at least
based on the feedback I'm hearing from users.

Proposed Change
===============

Overview
--------

There seem to be a number of low-hanging fruit candidates for cleanup.  In the
order in which they appear in the docs, these would be:

#. **Node registration** Why is this two steps?  Is there ever a case where we
   would want to register a node but not configure it to be able to boot?
   If there is, is it a significant enough use case to justify the added
   step every time a user registers nodes?

   I propose that we configure boot on newly registered nodes automatically.
   Note that this will probably require us to also update the boot
   configuration when updating images, but again this is a good workflow
   improvement.  Users are likely to forget to reconfigure their nodes' boot
   images after updating them in Glance.

   .. note:: This would not remove the ``openstack baremetal configure boot``
             command for independently updating the boot configuration of
             Ironic nodes.  In essence, it would just always call the
             configure boot command immediately after registering nodes so
             it wouldn't be a mandatory step.

             This also means that the deploy ramdisk would have to be built
             and loaded into Glance before registering nodes, but our
             documented process already satisfies that requirement, and we
             could provide a --no-configure-boot param to import for cases
             where someone wanted to register nodes without configuring them.

#. **Flavor creation** Nowhere in our documentation do we recommend or
   provide guidance on customizing the flavors that will be used for
   deployment.  While it is possible to deploy solely based on flavor
   hardware values (ram, disk, cpu), in practice it is often simpler
   to just assign profiles to Ironic nodes and have scheduling done solely
   on that basis.  This is also the method we document at this time.

   I propose that we simply create all of the recommended flavors at
   undercloud install time and assign them the appropriate localboot and
   profile properties at that time.  These flavors would be created with the
   minimum supported cpu, ram, and disk values so they would work for any
   valid hardware configuration.  This would also reduce the possibility of
   typos in the flavor creation commands causing avoidable deployment
   failures.

   These default flavors can always be customized if a user desires, so there
   is no loss of functionality from making this change.

#. **Node profile assignment** This is not currently part of the standard
   workflow, but in practice it is something we need to be doing for most
   real-world deployments with heterogeneous hardware for controllers,
   computes, cephs, etc.  Right now the documentation requires running an
   ironic node-update command specifying all of the necessary capabilities
   (in the manual case anyway, this section does not apply to the AHC
   workflow).

   os-cloud-config does have support for specifying the node profile in
   the imported JSON file, but to my knowledge we don't mention that anywhere
   in the documentation.  This would be the lowest of low-hanging
   fruit since it's simply a question of documenting something we already
   have.

   We could even give the generic baremetal flavor a profile and have our
   default instackenv.json template include that[1], with a note that it can
   be overridden to a more specific profile if desired.  If users want to
   change a profile assignment after registration, the node update command
   for ironic will still be available.

   1. For backwards compatibility, we might want to instead create a new flavor
   named something like 'default' and use that, leaving the old baremetal
   flavor as an unprofiled thing for users with existing unprofiled nodes.

Alternatives
------------

tripleo.sh
~~~~~~~~~~
tripleo.sh addresses the problem to some extent for developers, but it is
not a viable option for real world deployments (nor should it be IMHO).
However, it may be valuable to look at tripleo.sh for guidance on a simpler
flow that can be more easily followed, as that is largely the purpose of the
script.  A similar flow codified into the client/API would be a good result
of these proposed changes.

Node Registration
~~~~~~~~~~~~~~~~~
One option Dmitry has suggested is to make the node registration operation
idempotent, so that it can be re-run any number of times and will simply
update the details of any already registered nodes.  He also suggested
moving the bulk import functionality out of os-cloud-config and (hopefully)
into Ironic itself.

I'm totally in favor of both these options, but I suspect that they will
represent a significantly larger amount of work than the other items in this
spec, so I think I'd like that to be addressed as an independent spec since
this one is already quite large.

Security Impact
---------------

Minimal, if any.  This is simply combining existing deployment steps.  If we
were to add a new API for node profile assignment that would have some slight
security impact as it would increase our attack surface, but I feel even that
would be negligible.

Other End User Impact
---------------------

Simpler deployments.  This is all about the end user.

Performance Impact
------------------

Some individual steps may take longer, but only because they will be
performing actions that were previously in separate steps.  In aggregate
the process should take about the same time.

Other Deployer Impact
---------------------

If all of these suggested improvements are implemented, it will make the
standard deployment process somewhat less flexible.  However, in the
Proposed Change section I attempted to address any such new limitations,
and I feel they are limited to the edgiest of edge cases that in most cases
can still be implemented through some extra manual steps (which likely would
have been necessary anyway - they are edge cases after all).

Developer Impact
----------------

There will be some changes in the basic workflow, but as noted above the same
basic steps will be getting run.  Developers will see some impact from the
proposed changes, but as they will still likely be using tripleo.sh for an
already simplified workflow it should be minimal.

Implementation
==============

Assignee(s)
-----------

bnemec

Work Items
----------

* Configure boot on newly registered nodes automatically.
* Reconfigure boot on nodes after deploy images are updated.
* Remove explicit step for configure boot from the docs, but leave the actual
  function itself in the client so it can still be used when needed.
* Create flavors at undercloud install time and move documentation on creating
  them manually to the advanced section of the docs.
* Add a 'default' flavor to the undercloud.
* Update the sample instackenv.json to include setting a profile (by default,
  the 'default' profile associated with the flavor from the previous step).



Dependencies
============

Nothing that I'm aware of.


Testing
=======

As these changes are implemented, we would need to update tripleo.sh to match
the new flow, which will result in the changes being covered in CI.


Documentation Impact
====================

This should reduce the number of steps in the basic deployment flow in the
documentation.  It is intended to simplify the documentation.


References
==========

Proposed change to create flavors at undercloud install time:
https://review.openstack.org/250059
https://review.openstack.org/251555
