..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Instance High Availability
==========================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/instance-ha

A very often requested feature by operators and customers is to be able to
automatically resurrect VMs that were running on a compute node that failed (either
due to hardware failures, networking issues or general server problems).
Currently we have a downstream-only procedure which consists of many manual
steps to configure Instance HA:
https://access.redhat.com/documentation/en/red-hat-openstack-platform/9/paged/high-availability-for-compute-instances/chapter-1-overview

What we would like to implement here is basically an optional opt-in automatic
deployment of a cloud that has Instance HA support.

Problem Description
===================

Currently if a compute node has a hardware failure or a kernel panic all the
instances that were running on the node, will be gone and manual intervention 
is needed to resurrect these instances on another compute node.

Proposed Change
===============

Overview
--------

The proposed change would be to add a few additional puppet-tripleo profiles that would help
us configure the pacemaker resources needed for instance HA. Unlike in previous iterations
we won't need to move nova-compute resources under pacemaker's management. We managed to
achieve the same result without touching the compute nodes (except by setting
up pacemaker_remote on the computes, but that support exists already) 

Alternatives
------------

There are a few specs that are modeling host recovery:

Host Recovery - https://review.openstack.org/#/c/386554/
Instances auto evacuation - https://review.openstack.org/#/c/257809

The first spec uses pacemaker in a very similar way but is too new
and too high level to really be able to comment at this point in time.
The second one has been stalled for a long time and it looks like there
is no consensus yet on the approaches needed. The longterm goal is
to morph the Instance HA deployment into the spec that gets accepted.
We are actively working on both specs as well. In any case we have
discussed the long-term plan with SuSe and NTT and we agreed
on a long-term plan of which this spec is the first step for TripleO.

Security Impact
---------------

No additional security impact.

Other End User Impact
---------------------

End users are not impacted except for the fact that VMs can be resurrected
automatically on a non-failed compute node.

Performance Impact
------------------

There are no performance related impacts as compared to a current deployment.

Other Deployer Impact
---------------------

So this change does not affect the default deployments. What it does it adds a boolean
and some additional profiles so that a deployer can have a cloud configured with Instance
HA support out of the box.

* One top-level parameter to enable the Instance HA deployment

* Although fencing configuration is already currently supported by tripleo, we will need
  to improve bits and pieces so that we won't need an extra command to generate the
  fencing parameters.

* Upgrades will be impacted by this change in the sense that we will need to make sure to test
  them when Instance HA is enabled.

Developer Impact
----------------

No developer impact is planned.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  michele

Other contributors:
  cmsj, abeekhof

Work Items
----------

* Make the fencing configuration fully automated (this is mostly done already, we need oooq integration
  and some optimization)

* Add the logic and needed resources on the control-plane

* Test the upgrade path when Instance HA is configured


Testing
=======

Testing this manually is fairly simple:

* Deploy with Instance HA configured and two compute nodes

* Spawn a test VM

* Crash the compute node where the VM is running

* Observe the VM being resurrected on the other compute node

Testing this in CI is doable but might be a bit more challenging due to resource constraints.

Documentation Impact
====================

A section under advanced configuration is needed explaining the deployment of
a cloud that supports Instance HA.

References
==========

* https://access.redhat.com/documentation/en/red-hat-openstack-platform/9/paged/high-availability-for-compute-instances/
