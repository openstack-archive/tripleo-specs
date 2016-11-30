..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Enable deployment of alternative backends for oslo.messaging
============================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/om-dual-backends

This spec describes adding two functional capabilities to the messaging
services of an overcloud deployment. The first capability is to enable
the selection and configuration of separate messaging backends for
oslo.messaging RPC and Notification communications. The second
capability is to introduce support for a brokerless messaging backend
for oslo.messaging RPC communications via the AMQP 1.0 Apache
qpid-dispatch-router.


Problem Description
===================

The oslo.messaging library supports the deployment of dual messaging system
backends. This enables alternative backends to be deployed for RPC and
Notification messaging communications. Users have identified the
constraints of using a store and forward (broker based) messaging system for RPC
communications and are seeking direct messaging (brokerless)
approaches to optimize the RPC messaging pattern. In addition to
operational challenges, emerging distributed cloud architectures
define requirements around peer-to-peer relationships and geo-locality
that can be addressed through intelligent messaging transport routing
capabilities such as is provided by the AMQP 1.0 qpid-dispatch-router.


Proposed Change
===============

Overview
--------

Provide the capability to select and configure alternative
transport_url's for oslo.messaging RPCs and Notifications across
overcloud OpenStack services.

Retain the current default behavior to deploy the rabbitMQ server as
the messaging backend for both RPC and Notification communications.

Introduce an alternative deployment of the qpid-dispatch-router as the
messaging backend for RPC communications.

Utilize the oslo.messaging AMQP 1.0 driver for delivering RPC services
via the dispatch-router messaging backend.

Alternatives
------------

The configuration of dual backends for oslo.messaging could be
performed post overcloud deployment.

Security Impact
---------------

The end result of using the AMQP 1.0 dispatch-router as an alternative
messaging backend for oslo.messaging RPC communications should be the
same from a security standpoint. The driver/router solution provides
SSL and SASL support in parity to the current rabbitMQ server deployment.

Other End User Impact
---------------------

The configuration of the dual backends for RPC and Notification
messaging communications should be transparent to the operation of the OpenStack
services.

Performance Impact
------------------

Using a dispatch-router mesh topology rather than broker clustering
for messaging communications will have a positive impact on
performance and scalability by:

* Directly expanding connection capacity

* Providing parallel communication flows across the mesh

* Increasing aggregate message transfer capacity

* Improving resource utilization of messaging infrastructure

Other Deployer Impact
---------------------

The deployment of the dispatch-router, however, will be new to
OpenStack operators. Operators will need to learn the
architectural differences as compared to a broker cluster
deployment. This will include capacity planning, monitoring,
troubleshooting and maintenance best practices.

Developer Impact
----------------

Support for alternative oslo.messaging backends and deployment of
qpid-dispatch-router in addition to rabbitMQ should be implemented for
tripleo-quickstart.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* John Eckersberg <jeckersb@redhat.com>

* Andy Smith <ansmith@redhat.com>


Work Items
----------

* Update overcloud templates for dual backends and dispatch-router service

* Add dispatch-router packages to overcloud image elements

* Add services template for dispatch-router

* Update OpenStack services base templates to select and configure
  transport_urls for RPC and Notification

* Deploy dispatch-router for controller and compute for topology

* Test failure and recovery scenarios for dispatch-router

Transport Configuration
-----------------------

The oslo.messaging configuration options define a default and
additional notification transport_url. If the notification
transport_url is not specified, oslo.messaging will use the default
transport_url for both RPC and Notification messaging communications.

The transport_url parameter is of the form::

  transport://user:pass@host1:port[,hostN:porN]/virtual_host

Where the transport scheme specifies the RPC or Notification backend as
one of rabbit or amqp, etc. Oslo.messaging is deprecating the host,
port and auth configuration options. All drivers will get these
options via the transport_url.


Dependencies
============

Support for dual backends in and AMQP 1.0 driver integration
with the dispatch-router depends on oslo.messaging V5.10 or later.


Testing
=======

In order to test this in CI, an environment will be needed where dual
messaging system backends (e.g. rabbitMQ server and dispatch-router
server) are deployed. Any existing hardware configuration should be
appropriate for the dual backend deployment.


Documentation Impact
====================

The deployment documentation will need to be updated to cover the
configuration of dual messaging system backends and the use of the
dispatch-router. TripleO Heat template examples should also help with
deployments using dual backends.


References
==========

* [1] https://blueprints.launchpad.net/oslo.messaging/+spec/amqp-dispatch-router
* [2] http://qpid.apache.org/components/dispatch-router/
* [3] http://docs.openstack.org/developer/oslo.messaging/AMQP1.0.html
* [4] https://etherpad.openstack.org/p/ocata-oslo-consistent-mq-backends
* [5] https://github.com/openstack/puppet-qdr
