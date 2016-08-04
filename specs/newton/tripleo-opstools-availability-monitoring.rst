..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Enable deployment of availability monitoring
============================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-opstools-availability-monitoring

TripleO should be deploying out-of-the-box availability monitoring solution
to serve the overcloud.

Problem Description
===================

Currently there is no such feature implemented except for possibility to deploy
sensu-server, sensu-api and uchiwa (Sensu dashboard) services in the undercloud
stack. Without sensu-client services deployed on overcloud nodes this piece
of code is useless. Due to potential of high resource consumption it is also
reasonable to remove current undercloud code to avoid possible problems
when high number of overcloud nodes is being deployed.

Instead sensu-server, sensu-api and uchiwa should be deployed on the separate
node(s) whether it is on the undercloud level or on the overcloud level.
And so sensu-client deployment support should be flexible enough to enable
connection to external monitoring infrastructure or with Sensu stack deployed
on the dedicated overcloud node.

Summary of use cases:

1. sensu-server, sensu-api and uchiwa deployed in external infrastructure;
sensu-client deployed on each overcloud node
2. sensu-server, sensu-api and uchiwa deployed as a separate Heat stack in
the overcloud stack; sensu-client deployed on each overcloud node

Proposed Change
===============

Overview
--------

The sensu-client service will be deployed as a composable service on
the overcloud stack when it is explicitly stated via environment file.
Sensu checks will have to be configured as subscription checks (see [0]
for details). Each composable service will have it's own subscription string,
which will ensure that checks defined on Sensu server node (wherever it lives)
are run on the correct overcloud nodes.

There will be implemented a possibility to deploy sensu-server, sensu-api
and uchiwa services on a stand alone node deployed by the undercloud.
This standalone node will have a dedicated purpose for monitoring
(not only for availability monitoring services, but in future also for
centralized logging services or performance monitoring services)

The monitoring node will be deployed as a separate Heat stack to the overcloud
stack using Puppet and composable roles for required services.

Alternatives
------------

None

Security Impact
---------------

Additional service (sensu-client) will be installed on all overcloud nodes.
These services will have open connection to RabbitMQ instance running
on monitoring node and are used to execute commands (checks) on the overcloud
nodes. Check definition will live on the monitoring node.

Other End User Impact
---------------------

None

Performance Impact
------------------

We might consider deploying separate RabbitMQ and Redis for monitoring purposes
if we want to avoid influencing OpenStack deployment in the overcloud.

Other Deployer Impact
---------------------

* Sensu clients will be deployed by default on all overcloud nodes except the monitoring node.
* New Sensu common parameters:

    * MonitoringRabbitHost

        * RabbitMQ host Sensu has to connect to

    * MonitoringRabbitPort

        * RabbitMQ port Sensu has to connect to

    * MonitoringRabbitUseSSL

        * Whether Sensu should connect to RabbitMQ using SSL

    * MonitoringRabbitPassword

        * RabbitMQ password used for Sensu to connect

    * MonitoringRabbitUserName

        * RabbitMQ username used for Sensu to connect

    * MonitoringRabbitVhost

        * RabbitMQ vhost used for monitoring purposes.

* New Sensu server/API parameters

    * MonitoringRedisHost

        * Redis host Sensu has to connect to

    * MonitoringRedisPassword

        * Redis password used for Sensu to connect

    * MonitoringChecks:

        * Full definition (for all subscriptions) of checks performed by Sensu

* New parameters for subscription strings for each composable service:

    * For example for service nova-compute MonitoringSubscriptionNovaCompute, which will default to 'overcloud-nova-compute'


Developer Impact
----------------

Support for new node type should be implemented for tripleo-quickstart.

Implementation
==============

Assignee(s)
-----------

Martin Mágr <mmagr@redhat.com>

Work Items
----------

* puppet-tripleo profile for Sensu services
* puppet-tripleo profile for Uchiwa service
* tripleo-heat-templates composable service for sensu-client deployment
* tripleo-heat-templates composable service for sensu-server deployment
* tripleo-heat-templates composable service for sensu-api deployment
* tripleo-heat-templates composable service for uchiwa deployment
* Support for monitoring node in tripleo-quickstart
* Revert patch(es) implementing Sensu support in instack-undercloud

Dependencies
============

* Puppet module for Sensu services: sensu-puppet [1]
* Puppet module for Uchiwa: puppet-uchiwa [2]
* CentOS Opstools SIG repo [3]

Testing
=======

Sensu client deployment will be tested by current TripleO CI as soon as
the patch is merged, as it will be deployed by default.

We should consider creating CI job for deploying overcloud with monitoring
node to test the rest of the monitoring components.

Documentation Impact
====================

Process of creating new node type and new options will have to be documented.

References
==========

[0] https://sensuapp.org/docs/latest/reference/checks.html#subscription-checks
[1] https://github.com/sensu/sensu-puppet
[2] https://github.com/Yelp/puppet-uchiwa
[3] https://wiki.centos.org/SpecialInterestGroup/OpsTools
