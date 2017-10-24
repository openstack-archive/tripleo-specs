..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Tripleo RPC and Notification Messaging Support
==============================================

https://blueprints.launchpad.net/tripleo

This specification proposes changes to tripleo to enable the selection
and configuration of separate messaging backends for oslo.messaging
RPC and Notification communications. This proposal is a derivative of
the work associated with the original blueprint [1]_ and specification
[2]_ to enable dual backends for oslo.messaging in tripleo.

Most of the groundwork to enable dual backends was implemented during
the pike release and the introduction of an alternative messaging
backend (qdrouterd) service was made. Presently, the deployment of this
alternative messaging backend is accomplished by aliasing the rabbitmq
service as the tripleo implementation does not model separate
messaging backends.

Problem Description
===================

The oslo.messaging library supports the deployment of dual messaging
system backends for RPC and Notification communications. However, tripleo
currently deploys a single rabbitmq server (cluster) that serves as a
single messaging backend for both RPC and Notifications.

::

     +------------+         +----------+
     | RPC Caller |         | Notifier |
     +-----+------+         +----+-----+
           |                     |
           +--+               +--+
              |               |
              v               v
            +-+---------------+-+
            |  RabbitMQ Service |
            | Messaging Backend |
            |                   |
            +-+---------------+-+
              ^               ^
              |               |
           +--+               +--+
           |                     |
           v                     v
    +------+-----+        +------+-------+
    |    RPC     |        | Notification |
    |   Server   |        |    Server    |
    +------------+        +--------------+


To support two separate and distinct messaging backends, tripleo needs
to "duplicate" the set of parameters needed to specify each messaging
system. The oslo.messaging library in OpenStack provides the API to the
messaging services. It is proposed that the implementation model the
RPC and Notification messaging services in place of the backend
messaging server (e.g. rabbitmq).

::

     +------------+          +----------+
     | RPC Caller |          | Notifier |
     +-----+------+          +----+-----+
           |                      |
           |                      |
           v                      v
  +-------------------+  +-------------------+
  |       RPC         |  |    Notification   |
  | Messaging Service |  | Messaging Service |
  |                   |  |                   |
  +--------+----------+  +--------+----------+
           |                      |
           |                      |
           v                      v
     +------------+        +------+-------+
     |    RPC     |        | Notification |
     |   Server   |        |    Server    |
     +------------+        +--------------+


Introducing the separate messaging services and associated parameters in place
of the rabbitmq server is not a major rework but special consideration
must be made to upgrade paths and capabilities to ensure that existing
configurations are not impacted.

Having separate messaging backends for RPC and Notification
communications provides a number of benefits. These benefits include:

* tuning the backend to the messaging patterns
* increased aggregate message capacity
* reduced applied load to messaging servers
* increased message throughput
* reduced message latency
* etc.


Proposed Change
===============

A number of issues need to be resolved in order to express RPC
and Notification messaging services on top of the backend messaging systems.

Overview
--------

The proposed change is similar to the concept of a service "backend"
that is configured by tripleo. A number of existing services support
such a backend (or plugin) model. The implementation of a messaging
service backend model should account for the following requirements:

* deploy a single messaging backend for both RPC and Notifications
* deploy a messaging backend twice, once for RPC and once for
  Notifications
* deploy a messaging backend for RPC and a different messaging backend
  for Notifications
* deploy an external messaging backend for RPC
* deploy an external messaging backend for Notifications

Generally, the parameters that were required for deployment of the
rabbitmq service should be duplicated and renamed to "RPC Messaging"
and "Notify Messaging" backend service definitions. Individual backend
files would exist for each possible backend type (e.g. rabbitmq,
qdrouterd, zeromq, kafka or external). The backend selected will
correspondingly define the messaging transport for the messaging
system.

* transport specifier
* username
* password (and generation)
* host
* port
* virtual host(s)
* ssl (enabled)
* ssl configuration
* health checks

Tripleo should continue to have a default configuration that deploys
RPC and Notifications messaging services on top of a single rabbitmq
backend server (cluster). Tripleo upgrades should map the legacy
rabbitmq service deployment onto the RPC and Notification messaging
services model.


Alternatives
------------

The configuration of separate messaging backends could be post
overcloud deployment (e.g. external to tripleo framework). This would
be problematic over the lifecycle of deployments e.g. during upgrades etc.

Security Impact
---------------

The deployment of dual messaging backends for RPC and Notification
communications should be the same from a security standpoint. This
assumes the backends have parity from a security feature
perspective, e.g authentication and encryption.

Other End User Impact
---------------------

Depending on the configuration of the messaging backend deployment,
there could be a number of end user impacts including the following:

* monitoring of separated messaging backend services
* understanding differences in functionality/behaviors between different
  messaging backends (e.g. broker versus router, etc.)
* handling exceptions (e.g. different places for logs, etc.)

Performance Impact
------------------

Using separate messaging systems for RPC and Notifications  should
have a positive impact on performance and scalability by:

* separating RPC and Notification messaging loads
* increased parallelism in message processing
* increased aggregate message transfer capacity
* tuned backend configuration aligned to messaging patterns

Other Deployer Impact
---------------------

The deployment of hybrid messaging will be new to OpenStack
operators. Operators will need to learn the architectural differences
as compared to a single backend deployment. This will include capacity
planning, monitoring, troubleshooting and maintenance best practices.

Developer Impact
----------------

Discuss things that will affect other developers working on OpenStack.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Andy Smith <ansmith@redhat.com>

* John Eckersberg <jeckersb@redhat.com>

Work Items
----------

tripleo-heat-templates:

* Modify *puppet/services/<service>base.yaml* to introduce separate RPC and
  Notification Messaging parameters (e.g. replace 'rabbit' parameters)
* Support two ssl environments (e.g. one for RPC and one for
  Notification when separate backends are deployed)
* Consider example backend model such as the following:

::

    tripleo-heat-templates
    |
    +--+ /environments
    |  |
    |  +--+ /messaging
    |     |
    |     +--+ messaging-(rpc/notify)-rabbitmq.yaml
    |     +--+ messaging-(rpc/notify)-qdrouterd.yaml
    |     +--+ messaging-(rpc/notify)-zmq.yaml
    |     +--+ messaging-(rpc/notify)-kafka.yaml
    +--+ /puppet
    |  |
    |  +--+ /services
    |     |
    |     +--+ messaging-(rpc/notify)-backend-rabbitmq.yaml
    |     +--+ messaging-(rpc/notify)-backend-qdrouterd.yaml
    |     +--+ messaging-(rpc/notify)-backend-zmq.yaml
    |     +--+ messaging-(rpc/notify)-backend-kafka.yaml
    |
    +--+ /roles


puppet_tripleo:

* Replace rabbitmq_node_names with messaging_rpc_node_names and
  messaging_notify_node_names or similar
* Add vhost support
* Consider example backend model such as the following:

::

    puppet-tripleo
    |
    +--+ /manifests
       |
       +--+ /profile
          |
          +--+ /base
             |
             +--+ /messaging
                |
                +--+ backend.pp
                +--+ rpc.pp
                +--+ notify.pp
                   |
                   +--+ /backend
                      |
                      +--+ rabbitmq.pp
                      +--+ qdrouterd.pp
                      +--+ zmq.pp
                      +--+ kafka.pp


tripleo_common:

* Add user and password management for RPC and Messaging services
* Support distinct health checks for separated messaging backends

packemaker:

* Determine what should happen when two separate rabbitmq clusters
  are deployed. Does this result in two pacemaker services or one?
  Some experimentation may be required.

Dependencies
============

None.

Testing
=======

In order to test this in CI, an environment will be needed where separate
messaging system backends (e.g. rabbitMQ server and dispatch-router
server) are deployed. Any existing hardware configuration should be
appropriate for the dual backend deployment.


Documentation Impact
====================

The deployment documentation will need to be updated to cover the
configuration of the separate messaging (RPC and Notify) services.


References
==========

.. [1] https://blueprints.launchpad.net/tripleo/+spec/om-dual-backends
.. [2] https://review.openstack.org/#/c/396740/
