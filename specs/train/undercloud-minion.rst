..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Scale Undercloud with a Minion
==============================

https://blueprints.launchpad.net/tripleo/undercloud-minion

In order to improve our scale, we have identified heat-engine and possibly
ironic-conductor as services that we can add on to an existing undercloud
deployment.  Adding heat-engine allows for additional processing capacity
when creating and updating stacks for deployment.  By adding a new light
weight minion node, we can scale the Heat capacity horizontally.

Additionally since these nodes could be more remote, we could add an
ironic-conductor instance to be able to manage hosts in a remote region
while still having a central undercloud for the main management.


Problem Description
===================

Currently we use a single heat-engine on the undercloud for the deployment.
According to the Heat folks, it can be beneficial for processing to have
additional heat-engine instances for scale. The recommended scaling is out
rather than up.  Additionally by being able to deploy a secondary host, we
can increase our capacity for the undercloud when additional scale capacity
is required.


Proposed Change
===============

Overview
--------

We are proposing to add a new undercloud "minion" configuration that can be
used by operators to configure additional instances of heat-engine and
ironic-conductor when they need more processing capacity.  We would also
allow the operator to disable heat-engine from the main undercloud to reduce
the resource usage of the undercloud.  By removing the heat-engine from the
regular undercloud, the operator could possibly avoid timeouts on other services
like keystone and neutron that can occur when the system is under load.

Alternatives
------------

An alternative would be to make the undercloud deployable in a traditional
HA capacity where we share the services across multiple nodes. This would
increase the overall capacity but adds additional complexity to the undercloud.
Additionally this does not let us target specific services that are resource
heavy.

Security Impact
---------------

The new node would need to have access to the the main undercloud's keystone,
database and messaging services.

Upgrade Impact
--------------

The new minion role would need to be able to be upgraded by the user.

Other End User Impact
---------------------

None.

Performance Impact
------------------

* This additional minion role may improve heat processing due to the additional
  resource capacity being provided.

* Locating an ironic-conductor closer to the nodes being managed can improve
  performance by being closer to the systems (less latency, etc).


Other Deployer Impact
---------------------

Additional undercloud role and a new undercloud-minion.conf configuration file
will be created. Additionally a new option may be added to the undercloud.conf
to manage heat-engine instalation.

Developer Impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mwhahaha

Other contributors:
  slagle
  EmilienM

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.

python-tripleoclient
~~~~~~~~~~~~~~~~~~~~

* New 'openstack undercloud minion deploy' command for installation

* New 'openstack undercloud minion upgrade' command for upgrades

* New configuration file 'undercloud-minion.conf' to drive the installation
  and upgrades.

* New configuration option in 'undercloud.conf' to provide ability to disable
  the heat-engine on the undercloud.

tripleo-heat-templates
~~~~~~~~~~~~~~~~~~~~~~

* New 'UndercloudMinion' role file

* New environment file for the undercloud minion deployment

* Additional environment files to enable or disable heat-engine and
  ironic-conductor.

Dependencies
============

None.

Testing
=======

We would add a new CI job to test the deployment of the minion node. This job
will likely be a new multinode job.



Documentation Impact
====================

We will need to document the usage of the undercloud minion installation and
the specific use cases where this can be beneficial.


References
==========

See the notes from the Train PTG around Scaling.

* https://etherpad.openstack.org/p/tripleo-ptg-train

* https://etherpad.openstack.org/p/DEN-tripleo-forum-scale
