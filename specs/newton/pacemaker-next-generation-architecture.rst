..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Pacemaker Next Generation Architecture
======================================

https://blueprints.launchpad.net/tripleo/+spec/ha-lightweight-architecture

Change the existing HA manifests and templates to deploy a minimal pacemaker
architecture, where all the openstack services are started and monitored by
systemd with the exception of: VIPs/Haproxy, rabbitmq, redis and galera.

Problem Description
===================

The pacemaker architecture deployed currently via
`puppet/manifests/overcloud_controller_pacemaker.pp` manages most
service on the controllers via pacemaker. This approach, while having the
advantage of having a single entity managing and monitoring all services, does
bring a certain complexity to it and assumes that the operaters are quite
familiar with pacemaker and its management of resources. The aim is to
propose a new architecture, replacing the existing one, where pacemaker
controls the following resources:

* Virtual IPs + HAProxy
* RabbitMQ
* Galera
* Redis
* openstack-cinder-volume (as the service is not A/A yet)
* Any future Active/Passive service

Basically every service that is managed today by a specific resource agent
and not systemd, will be still running under pacemaker. The same goes
for any service (like openstack-cinder-volume) that need to be active/passive.

Proposed Change
===============

Overview
--------

Initially the plan was to create a brand new template implementing this
new HA architecture. After a few rounds of discussions within the TripleO
community, it has been decided to actually have a single HA architecture.
The main reasons for moving to a single next generation HA architecture are due to
the amount work needed to maintain two separate architectures and to the
fact that the previous HA architecture does not bring substantial advantages
over this next generation one.

The new architecture will enable most services via systemd and will remove most
pacemaker resource definitions with their corresponding constraints.
In terms of ordering constraints we will go from a graph like this one:
http://acksyn.org/files/tripleo/wsgi-openstack-core.pdf (mitaka)

to a graph like this one:
http://acksyn.org/files/tripleo/light-cib-nomongo.pdf (next-generation-mitaka)

Once this new architecture is in place and we have tested it extensively, we
can work on the upgrade path from the previous fully-fledged pacemaker HA
architecture to this new one. Since the impact of pacemaker in the new
architecture is quite small, it is possible to consider dropping the non-ha
template in the future for every deployment and every CI job. The decision
on this can be taken in a later step, even post-newton.

Another side-benefit is that with this newer architecture the
whole upgrade/update topic is much easier to manage with TripleO,
because there is less coordination needed between pacemaker, the update
of openstack services, puppet and the update process itself.

Note that once composable service land, this next generation architecture will
merely consist of a single environment file setting some services to be
started via systemd, some via pacemaker and a bunch of environment variables
needed for the services to reconnect even when galera and rabbitmq are down.
All services that need to be started via systemd will be done via the default
state:
https://github.com/openstack/tripleo-heat-templates/blob/40ad2899106bc5e5c0cf34c40c9f391e19122a49/overcloud-resource-registry-puppet.yaml#L124

The services running via pacemaker will be explicitely listed in an
environment file, like here:
https://github.com/openstack/tripleo-heat-templates/blob/40ad2899106bc5e5c0cf34c40c9f391e19122a49/environments/puppet-pacemaker.yaml#L12

Alternatives
------------

There are many alternative designs for the HA architecture. The decision
to use pacemaker only for a certain set of "core" services and all the
Active/Passive services comes from a careful balance between complexity
of the architecture and its management and being able to recover resources
in a known broken state. There is a main assumption here about native
openstack services:

They *must* be able to start when the broker and the database are down and keep
retrying.

The reason for using only pacemaker for the core services and not, for
example keepalived for the Virtual IPs, is to keep the stack simple and
not introduce multiple distributed resource managers. Also, if we used
only keepalived, we'd have no way of recovering from a failure beyond
trying to relocate the VIP.

The reason for keeping haproxy under pacemaker's management is that
we can guarantee that a VIP will always run where haproxy is running,
should an haproxy service fail.


Security Impact
---------------

No changes regarding security aspects compared to the existing status quo.

Other End User Impact
---------------------

The operators working with a cloud are impacted in the following ways:

* The services (galera, redis, openstack-cinder-volume, VIPs,
  haproxy) will be managed as usual via `pcs`. Pacemaker will monitor these
  services and provide their status via `pcs status`.

* All other services will be managed via `systemctl` and systemd will be
  configured to automatically restart a failed service. Note, that this is
  already done in RDO with (Restart={always,on-failure}) in the service files.
  It is a noop when pacemaker manages the service as an override file is
  created by pacemaker:

    https://github.com/ClusterLabs/pacemaker/blob/master/lib/services/systemd.c#L547

  With the new architecture, restarting a native openstack service across
  all controllers will require restaring it via `systemctl` on each node (as opposed
  to a single `pcs` command as it is done today)

* All services will be configured to retry indefinitely to connect to
  the database or to the messaging broker. In case of a controller failure,
  the failover scenario will be the same as with the current HA architecture,
  with the difference that the services will just retry to re-connect indefinitely.

* Previously with the HA template every service would be monitored and managed by
  pacemaker. With the split between openstack services being managed by systemd and
  "core" services managed by pacemaker, the operator needs to know which service
  to monitor with which command.

Performance Impact
------------------

No changes compared to the existing architecture.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

In the future we might see if the removal of the non-HA template is feasible,
thereby simplifying our CI jobs and have single more-maintained template.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  michele

Other contributors:
  ...


Work Items
----------

* Prepare the roles that deploy the next generation architecture.  Initially,
  keep it as close as possible to the existing HA template and make it simpler
  in a second iteration (remove unnecesary steps, etc.) Template currently
  lives here and deploys successfully:

    https://review.openstack.org/#/c/314208/

* Test failure scenarios and recovery scenario, open bugs against services that
  misbehave in the face of database and/or broker being down.


Dependencies
============

None

Testing
=======

Initial smoke-testing has been completed successfully. Another set of tests
focusing on the behaviour of openstack services when galera and rabbitmq are
down is in the process of being run.

Particular focus will be on failover scenarios and recovery times and making
sure that there are no regressions compared to the current HA architecture.


Documentation Impact
====================

Currently we do not describe the architectures as deployed by TripleO itself,
so no changes needed. A short page in the docs describing the architecture
would be a nice thing to have in the future.

References
==========

This design came mostly out from a meeting in Brno with the following attendees:

* Andrew Beekhof
* Chris Feist
* Eoghan Glynn
* Fabio Di Nitto
* Graeme Gillies
* Hugh Brock
* Javier Peña
* Jiri Stransky
* Lars Kellogg-Steadman
* Mark Mcloughlin
* Michele Baldessari
* Raoul Scarazzini
* Rob Young
