..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Enable deployment of centralized logging
========================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-opstools-centralized-logging

TripleO should be deploying with an out-of-the-box centralized logging
solution to serve the overcloud.

Problem Description
===================

With a complex distributed system like OpenStack, identifying and
diagnosing a problem may require tracking a transaction across many
different systems and many different logfiles.  In the absence of a
centralized logging solution, this process is frustrating to both new
and experienced operators and can make even simple problems hard to
diagnose.

Proposed Change
===============

We will deploy the Fluentd_ service in log collecting mode as a
composable service on all nodes in the overcloud stack when configured
to do so by the environment.  Each composable service will have its
own fluentd source configuration.

.. _fluentd: http://www.fluentd.org/

To receive these messages, we will deploy a centralized logging system
running Kibana_, Elasticsearch_ and Fluentd on dedicated nodes to
provide log aggregation and analysis.  This will be deployed in a
dedicated Heat stack that is separate from the overcloud stack using
composable roles.

.. _kibana: https://www.elastic.co/products/kibana
.. _elasticsearch: https://www.elastic.co/

We will also support sending messages to an external Fluentd
instance not deployed by tripleo.

Summary of use cases
--------------------

1. Elasticsearch, Kibana and Fluentd log relay/transformer deployed as
   a separate Heat stack in the overcloud stack; Fluentd log
   collector deployed on each overcloud node

2. ElasticSearch, Kibana and Fluentd log relay/transformer deployed in
   external infrastructure; Fluentd log collector deployed on each
   overcloud node

Alternatives
------------

None

Security Impact
---------------

Data collected from the logs of OpenStack services can contain
sensitive information:

- Communication between the
  fluentd agent and the log aggregator should be protected with SSL.

- Access to the Kibana UI must have at least basic HTTP
  authentication, and client access should be via SSL.

- ElasticSearch should only allow collections over ``localhost``.

Other End User Impact
---------------------

None

Performance Impact
------------------

Additional resources will be required for running Fluentd on overcloud
nodes.  Log traffic from the overcloud nodes to the log aggregator
will consume some bandwidth.

Other Deployer Impact
---------------------

- Fluentd will be deployed on all overcloud nodes.
- New parameters for configuring Fluentd collector.
- New parameters for configuring log collector (Fluentd,
  ElasticSearch, and Kibana)

Developer Impact
----------------

Support for the new node type should be implemented for tripleo-quickstart.

Implementation
==============

Assignee(s)
-----------

Martin Mágr <mmagr@redhat.com>
Lars Kellogg-Stedman <lars@redhat.com>

Work Items
----------

- puppet-tripleo profile for fluentd service
- tripleo-heat-templates composable role for FluentD collector deployment
- tripleo-heat-templates composable role for FluentD aggregator deployment
- tripleo-heat-templates composable role for ElasticSearch deployment
- tripleo-heat-templates composable role for Kibana deployment
- Support for logging node in tripleo-quickstart

Dependencies
============

- Puppet module for Fluentd: `konstantin-fluentd` [1]
- Puppet module for ElasticSearch `elasticsearch-elasticsearch` [2]
- Puppet module for Kibana (tbd)
- CentOS Opstools SIG package repository

Testing
=======

Fluentd client deployment will be tested by current TripleO CI as soon as
the patch is merged. Because the centralized logging features will not
be enabled by default we may need to introduce specific tests for
these features.

Documentation Impact
====================

Process of creating new node type and new options will have to be documented.

References
==========

[1] https://forge.puppet.com/srf/fluentd
[2] https://forge.puppet.com/elasticsearch/elasticsearch
