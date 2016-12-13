..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
GUI Deployment configuration improvements
==========================================

TripleO UI deployment configuration is based on enabling environments provided by
deployment plan (tripleo-heat-templates) and letting user set parameter values.

This spec proposes improvements to this approach.

Blueprint: https://blueprints.launchpad.net/tripleo/+spec/deployment-configuration-improvements

Problem Description
===================

The general goal of TripleO UI is to guide user through the deployment
process and provide relevant information along the way, so user does not
have to search for a context in documentation or by analyzing TripleO templates.

There is a set of problems identified with a current deployment configuration
solution. Resolving those problems should lead to improved user experience when
making deployment design decisions.

The important information about the usage of environment and relevant parameters
is usually included as a comment in environment file itself. This is not consumable by GUI.
We currently use capabilities-map.yaml to define environment meta data to work
around this.

* As the number of environments is growing it is hard to keep capabilities-map.yaml
  up to date. When certain environment is added, capabilities-map.yaml is usually
  not updated by the same developer, which leads to inaccuracy in environment
  description when added later.

* The environments depend on each other and potentially collide when used together

* There are no means to list and let user set parameters relevant to certain
  environment. These are currently listed as comments in environments - not
  consumable by GUI (example: [1])

* There are not enough means to organize parameters coming as a result of
  heat validate

* Not all parameters defined in tripleo-heat-templates have correct type set
  and don't include all relevant information that Hot Spec provides.
  (constraints...)

* Same parameters are defined in multiple templates in tripleo-heat-templates
  but their definition differs

* List of parameters which are supposed to get auto-generated when value is not
  provided by user are hard-coded in deployment workflow

Proposed Change
===============

Overview
--------

* Propose environment metadata to track additional information about environment
  directly as part of the file in Heat (partially in progress [2]). Similar concept is
  already present in heat resources [3].
  In the meantime update tripleo-common environment listing feature to read
  environments and include environment metadata.

  Each TripleO environment file should define:

  .. code::

    metadata:
      label: <human readable environment name>
      description: <description of the environment purpose>

    resource_registry:
      ...

    parameter_defaults:
      ...


* With the environment metadata in place, capabilities-map.yaml purpose would
  simplify to defining grouping and dependencies among environments.

* Implement environment parameter listing in TripleO UI

* To organize parameters we should use ParameterGroups.
  (related discussion: [4])

* Make sure that same parameters are defined the same way across tripleo-heat-templates
  There may be exceptions but in those cases it must be sure that two templates which
  define same parameter differently won't be used at the same time.

* Update parameter definitions in TripleO templates, so the type actually matches
  expected parameter value (e.g. 'string' vs 'boolean') This will result in correct
  input type being used in GUI

* Define a custom constraint for parameters which are supposed to be auto-generated.

Alternatives
------------

Potential alternatives to listing environment related parameters are:

* Use Parameter Groups to match template parameters to an environment. This
  solution ties the template with an environment and clutters the template.


* As the introduction of environment metadata depends on having this feature accepted
  and implemented in Heat, alternative solution is to keep title and description in
  capabilities map as we do now

Security Impact
---------------

No significant security impact

Other End User Impact
---------------------

Resolving mentioned problems greatly improves the TripleO UI workflow and
makes deployment configuration much more streamlined.

Performance Impact
------------------

Described approach allows to introduce caching of Heat validation which is
currently the most expensive operation. Cache gets invalid only in case
when a deployment plan is updated or switched.

Other Deployer Impact
---------------------

Same as End User Impact

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jtomasek

Other contributors:
  rbrady

Work Items
----------

* tripleo-heat-templates: update environments to include metadata (label,
  description), update parameter_defaults to include all parameters relevant
  to the environment

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/update-environment-files-with-related-parameters

* tripleo-heat-templates: update capabilities-map.yaml to map environment
  grouping and dependencies

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/update-capabilities-map-to-map-environment-dependencies

* tripleo-heat-templates: create parameter groups for deprecated and internal
  parameters

* tripleo-heat-templates: make sure that same parameters have the same definition

  bug: https://bugs.launchpad.net/tripleo/+bug/1640243

* tripleo-heat-templates: make sure type is properly set for all parameters

  bug: https://bugs.launchpad.net/tripleo/+bug/1640248

* tripleo-heat-templates: create custom constraint for autogenerated parameters

  bug: https://bugs.launchpad.net/tripleo/+bug/1636987

* tripleo-common: update environments listing to combine capabilities map with
  environment metadata

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/update-capabilities-map-to-map-environment-dependencies

* tripleo-ui: Environment parameters listing

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/get-environment-parameters

* tripleo-common: autogenerate values for parameters with custom constraint

  bug: https://bugs.launchpad.net/tripleo/+bug/1636987

* tripleo-ui: update environment configuration to reflect API changes, provide means to display and configure environment parameters

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/tripleo-ui-deployment-configuration

* tripleo-ui: add client-side parameter validations based on parameter type
  and constraints

  bugs: https://bugs.launchpad.net/tripleo/+bug/1638523, https://bugs.launchpad.net/tripleo/+bug/1640463

* tripleo-ui: don't show parameters included in deprecated and internal groups

Dependencies
============

* Heat Environment metadata discussion [2]

* Heat Parameter Groups discussion [3]

Testing
=======

The changes should be covered by unit tests in tripleo-common and GUI

Documentation Impact
====================

Part of this effort should be proper documentation of how TripleO environments
as well as capabilities-map.yaml should be defined

References
==========

[1] https://github.com/openstack/tripleo-heat-templates/blob/b6a4bdc3e4db97785b930065260c713f6e70a4da/environments/storage-environment.yaml

[2] http://lists.openstack.org/pipermail/openstack-dev/2016-June/097178.html

[3] http://docs.openstack.org/developer/heat/template_guide/hot_spec.html#resources-section.

[4] http://lists.openstack.org/pipermail/openstack-dev/2016-August/102297.html
