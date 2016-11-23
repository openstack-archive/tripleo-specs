..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Composable HA architecture
==========================

https://blueprints.launchpad.net/tripleo/+spec/composable-ha

Since Newton, we have the following services managed by pacemaker:

* Cloned and master/slave resources:
  galera, redis, haproxy, rabbitmq

* Active/Passive resources:
  VIPs, cinder-volume, cinder-backup, manila-share

It is currently not possible to compose the above service in the same
way like we do today via composable roles for the non-pacemaker services
This spec aims to address this limitation and let the operator be more flexible
in the composition of the control plane.

Problem Description
===================

Currently tripleo has implemented no logic whatsoever to assign specific pacemaker
managed services to roles/nodes.

* Since we do not have a lot in terms of hard performance data, we typically support
  three controller nodes. This is perceived as a scalability limiting factor and there is
  a general desire to be able to assign specific nodes to specific pacemaker-managed
  services (e.g. three nodes only for galera, five nodes only for rabbitmq)

* Right now if the operator deploys on N controllers he will get N cloned instances
  of the non-A/P pacemaker services on the same N nodes. We want to be able to
  be much more flexible. E.g. deploy galera on the first 3 nodes, rabbitmq on the
  remaining 5 nodes, etc.

* It is also desirable for the operator to be able to choose on which nodes the A/P
  resources will run.

* We also currently have a scalability limit of 16 nodes for the pacemaker cluster.

Proposed Change
===============

Overview
--------

The proposal here is to keep the existing cluster in its current form, but to extend
it in two ways:
A) Allow the operator to include a specific service in a custom node and have pacemaker
run that resource only on that node. E.g. the operator can define the following custom nodes:

* Node A
  pacemaker
  galera

* Node B
  pacemaker
  rabbitmq

* Node C
  pacemaker
  VIPs, cinder-volume, cinder-backup, manila-share, redis, haproxy

With the above definition the operator can instantiate any number of A, B or C nodes
and scale up to a total of 16 nodes. Pacemaker will place the resources only on
the appropriate nodes.

B) Allow the operator to extend the cluster beyond 16 nodes via pacemaker remote.
For example an operator could define the following:

* Node A
  pacemaker
  galera
  rabbitmq

* Node B
  pacemaker-remote
  redis

* Node C
  pacemaker-remote
  VIPs, cinder-volume, cinder-backup, manila-share, redis, haproxy

This second scenario would allow an operator to extend beyond the 16 nodes limit.
The only difference to scenario 1) is the fact that the quorum of the cluster is
obtained only by the nodes from Node A.

The way this would work is that the placement on nodes would be controllerd by location
rules that would work based on node properties matching.

Alternatives
------------

A bunch of alternative designs was discussed and evaluated:
A) A cluster per service:

One possible architecture would be to create a separate pacemaker cluster for
each HA service. This has been ruled out mainly for the following reasons:

* It cannot be done outside of containers
* It would create a lot of network traffic

* It would increase the management/monitoring of the pacemaker resources and clusters
  exponentially

* Each service would still be limited to 16 nodes
* A new container fencing agent would have to be written

B) A single cluster where only the clone-max property is set for the non A/P services

This would be still a single cluster, but unlike today where the cloned and
master/slave resources run on every controller we would introduce variables to
control the maximum number of nodes a resource could run on. E.g.
GaleraResourceCount would set clone-max to a value different than the number of
controllers. Example: 10 controllers, galera has clone-max set to 3, rabbit to
5 and redis to 3.
While this would be rather simple to implement and would change very little in the
current semantics, this design was ruled out:

* We'd still have the 16 nodes limit
* It would not provide fine grained control over which services live on which nodes

Security Impact
---------------

No changes regarding security aspects compared to the existing status quo.

Other End User Impact
---------------------

No particular impact except added flexibility in placing pacemaker-managed resources.

Performance Impact
------------------

The performance impact here is that with the added scalability it will be possible for
an operator to dedicate specific nodes for certain pacemaker-managed services.
There are no changes in terms of code, only a more flexible and scalable way to deploy
services on the control plane.

Other Deployer Impact
---------------------

This proposal aims to use the same method that the custom roles introduced in Newton
use to tailor the services running on a node. With the very same method it will be possible
to do that for the HA services managed by pacemaker today.

Developer Impact
----------------

No impact

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

We need to work on the following:

1. Add location rule constraints support in puppet
2. Make puppet-tripleo set node properties on the nodes where a service profile
3. Create corresponding location rules
4. Add a puppet-tripleo pacemaker-remote profile

Dependencies
============

No additional dependencies are required.

Testing
=======

We will need to test the flexible placement of the pacemaker-managed services
within the CI. This can be done within today's CI limitations (i.e. in the three
controller HA job we can make sure that the placement is customized and working)

Documentation Impact
====================

No impact

References
==========

Mostly internal discussions within the HA team at Red Hat
