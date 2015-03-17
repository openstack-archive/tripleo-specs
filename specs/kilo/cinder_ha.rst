..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========
Cinder HA
=========

https://blueprints.launchpad.net/tripleo/+spec/tripleo-kilo-cinder-ha

Ensure Cinder volumes remain available if one or multiple nodes running
Cinder services or hosting volumes go down.


Problem Description
===================

TripleO currently deploys Cinder without a shared storage, balancing requests
amongst the nodes. Should one of the nodes running `cinder-volume` fail,
requests for volumes hosted by that node will fail as well. In addition to that,
without a shared storage, should a disk of any of the `cinder-volume` nodes
fail, volumes hosted by that node would be lost forever.


Proposed Change
===============

Overview
--------

We aim at introducing support for the configuration of Cinder's Ceph backend
driver and for the deployment of a Ceph storage for use with Cinder.

Such a scenario will install `ceph-osd` on an arbitrary number of Ceph storage
nodes and `cinder-api`, `cinder-scheduler`, `cinder-volume` and `ceph-mon` on
the controller nodes, allowing users to scale out the Ceph storage nodes
independently from the controller nodes.

To ensure HA of the volumes, these will be then hosted on the Ceph storage and
to achieve HA for the `cinder-volume` service, all Cinder nodes will use a
shared string as their `host` config setting so that will be able to operate
on the entire (and shared) set of volumes.

Support for configuration of more drivers could be added later.

Alternatives
------------

An alternative approach could be to deploy the `cinder-volume` services in an
active/standby configuration. This would allow us to support scenarios where the
storage is not shared amongst the Cinder nodes, one of which is for example
LVM over a shared Fiber Channel LUNs. Such a scenario would suffer from
downsides though, it won't permit to scale out and balance traffic over the
storage nodes as easily and may be prone to issues related to the iSCSI session
management on failover.

A different scenario, based instead on the usage of LVM and DRBD combined, could
be imagined too. Yet this would suffer from downsides as well. The deployment
program would be put in charge of managing the replicas and probably required to
have some understanding of the replicas status as well. These are easily covered
by Ceph itself which takes care of more related problems indeed, like data
rebalancing, or replicas recreation.

Security Impact
---------------

By introducing support for the deployment of the Ceph's tools, we will have to
secure the Ceph services.

We will allow access to the data hosted by Ceph only to authorized hosts via
usage of `cephx` for authentication, distributing the `cephx` keyrings on the
relevant nodes. Controller nodes will be provisioned with the `ceph.mon`
keyring, with the `client.admin` keyring and the `client.cinder` keyring,
Compute nodes will be provisioned with the `client.cinder` secret in libvirt and
lastly the Ceph storage nodes will be provisioned with the `client.admin`
keyring.

It is to be said that monitors should not be reachable from the public
network, despite being hosted on the Controllers. Also Cinder won't need
to get access to the monitors' keyring nor the `client.admin` keyring but
those will be hosted on same host as Controllers also run the Ceph monitor
service; Cinder config will not provide any knowledge about those though.

Other End User Impact
---------------------

Cinder volumes as well as Cinder services will remain available despite failure
of one (or more depending on scaling setting) of the Controller nodes or Ceph
storage nodes.

Performance Impact
------------------

The `cinder-api` services will remain balanced and the Controller nodes unloaded
of the LVM-file overhead and the iSCSI traffic so this topology should, as an
additional benefit, improve performances.

Other Deployer Impact
---------------------

* Automated setup of Cinder HA will require the deployment of Ceph.

* To take advantage of a pre-existing Ceph installation instead of deploying it
  via TripleO, deployers will have to provide the input data needed to configure
  Cinder's backend driver appropriately

* It will be possible to scale the number of Ceph storage nodes at any time, as
  well as the number of Controllers (running `cinder-volume`) but changing the
  backend driver won't be supported as there are no plans to support volumes
  migration.

* Not all Cinder drivers support the scenario where multiple instances of the
  `cinder-volume` service use a shared `host` string, notably the default LVM
  driver does not. We will use this setting only when appropriate config params
  are found in the Heat template, as it happens today with the param called
  `include_nfs_backend`.

* Ceph storage nodes, running the `ceph-osd` service, use the network to
  maintain replicas' consistency and as such may transfer some large amount of
  data over the network. Ceph allows for the OSD service to differentiate
  between a public network and a cluster network for this purpose. This spec
  is not going to introduce support for usage of a dedicated cluster network
  but we want to have a follow-up spec to implement support for that later.

Developer Impact
----------------

Cinder will continue to be configured with the LVM backend driver by default.

Developers interested in testing Cinder with the Ceph shared storage will have
to use an appropriate scaling setting for the Ceph storage nodes.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gfidente

Other contributors:
  jprovazn

Work Items
----------

* add support for deployment of Cinder's Ceph backend driver

* add support for deployment of the Ceph services

* add support for external configuration of Cinder's Ceph backend driver


Dependencies
============

None.


Testing
=======

Will be testable in CI when support for the deployment of the shared Ceph
storage nodes becomes available in TripleO itself.


Documentation Impact
====================

We will need to provide documentation on how users can deploy Cinder together
with the Ceph storage nodes and also on how users can use instead some
pre-existing Ceph deployment.


References
==========

juno mid-cycle meetup
kilo design session, https://etherpad.openstack.org/p/tripleo-kilo-l3-and-cinder-ha
