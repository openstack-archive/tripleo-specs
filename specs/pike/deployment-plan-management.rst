..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Deployment Plan Management changes
==================================

https://blueprints.launchpad.net/tripleo/+spec/deployment-plan-management-refactor

The goal of this work is to improve GUI and CLI interoperability by changing the way
deployment configuration is stored, making it more compact and simplify plan import
and export.

Problem Description
===================

The problem is broadly described in mailing list discussion [1]. This spec is a result
of agreement achieved in that discussion.

TripleO-Common library currently operates on Mistral environment for storing plan
configuration although not all data are stored there since there are additional files
which define plan configuration (roles_data.yaml, network_data.yaml, capabilities-map.yaml)
which are currently used by CLI to drive certain parts of deployment configuration.
This imposes a problem of synchronization of content of those files with Mistral
environment when plan is imported or exported.

TripleO-Common needs to be able to provide means for roles and networks management.

Proposed Change
===============

Overview
--------

TripleO plan configuration data should be stored in single place rather than in multiple
(mistral environment + plan meta files stored in Swift container).

TripleO-Common should move from using mistral environment to storing the information
in file (plan-environment.yaml) in Swift container so all plan configuration data
are stored in 'meta' files in Swift and tripleo-common provides API to perform operations
on this data.

Plan meta files: capabilities-map.yaml, roles_data.yaml, network_data.yaml [3],
plan-environment.yaml

Proposed plan-environment.yaml file structure::

  version: 1.0

  name: A name of a plan which this file describes
  description: >
    A description of a plan, it's usage and potential summary of features it provides
  template: overcloud.yaml
  environments:
    - path: overcloud-resource-registry-puppet.yaml
  parameter_defaults:
    ControllerCount: 1
  passwords:
    TrovePassword: "vEPKFbdpTeesCWRmtjgH4s7M8"
    PankoPassword: "qJJj3gTg8bTCkbtYtYVPtzcyz"
    KeystoneCredential0: "Yeh1wPLUWz0kiugxifYU19qaf5FADDZU31dnno4gJns="


This solution makes whole plan configuration stored in Swift container together with
rest of plan files, simplifies plan import/export functionality as no synchronization
is necessary between the Swift files and mistral environment. Plan configuration is
more straightforward and CLI/GUI interoperability is improved.

Initially the plan configuration is going to be split into multiple 'meta' files
(plan-environment.yaml, capabilities-map.yaml, roles_data.yaml, network_data.yaml)
all stored in Swift container.
As a next step we can evaluate a solution which merges them all into plan-environment.yaml

Using CLI workflow user works with local files. Plan, Networks and Roles are configured by
making changes directly in relevant files (plan-management.yaml, roles_data.yaml, ...).
Plan is created and templates are generated on deploy command.

TripleO Common library will implement CRUD actions for Roles and Networks
management. This will allow clients to manage Roles and Networks and generate relevant
templates (see work items).

TripleO UI and other clients use tripleo-common library which operates on plan stored in
Swift container.


Alternatives
------------

Alternative approach is treating Swift 'meta' files as an input during plan creation
and synchronize them to Mistral environment when plan is imported which is described
initially in [1] and is used in current plan import/export implementation [2]

This solution needs to deal with multiple race conditions, makes plan import/export
much more complicated and overall solution is not simple to understand. Using this
solution should be considered if using mistral environment as a plan configuration
storage has some marginal benefits over using file in Swift. Which is not the case
according to the discussion [1]

As a subsequent step to proposed solution, it is possible to join all existing
'meta' files into a single one.

Security Impact
---------------

None.

Other End User Impact
---------------------

CLI/GUI interoperability is improved

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

This change makes Deployment Plan import/export functionality much simpler as well as
makes the tripleo-common operate on the same set of files as CLI does. It is much
easier to understand the CLI users how tripleo-common works as it does not do any
swift files -> mistral environment synchronization on the background.

TripleO-Common can introduce functionality manage Roles and Networks which perfectly
matches to how CLI workflow does it.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  akrivoka

Other contributors:
  * d0ugal
  * rbrady
  * jtomasek

Work Items
----------

* [tripleo-heat-templates] Update plan-environment.yaml to match new specification.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/update-plan-environment-yaml

* [tripleo-common] Update relevant actions to store data in plan-environment.yaml in
  Swift instead of using mistral-environment. Migrate any existing data away from Mistral.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/stop-using-mistral-env

* [tripleo-common] On plan creation/update tripleo-common validates the plan and checks
  that roles_data.yaml and network_data.yaml exist as well as validates it's format.
  On success, plan creation/update templates are generated/regenerated.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/validate-roles-networks

* [tripleo-common] Provide a GetRoles action to list current roles in json format by reading
  roles_data.yaml.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/get-roles-action

* [tripleo-common] Provide a GetNetworks action to list current networks in json format
  by reading network_data.yaml.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/get-networks-action

* [tripleo-common] Provide an UpdateRoles action to update Roles. It takes data in
  json format validates it's contents and persists them in roles_data.yaml, after
  successful update, templates are regenerated.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/update-roles-action

* [tripleo-common] Provide an UpdateNetworks action to update Networks. It takes data in
  json format validates it's contents and persists them in network_data.yaml.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/update-networks-action

* [tripleo-ui] Provide a way to create/list/update/delete Roles by calling tripleo-common
  actions.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/roles-crud-ui

* [tripleo-ui] Provide a way to create/list/update/delete Networks by calling tripleo-common
  actions.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/networks-crud-ui

* [tripleo-ui] Provide a way to assign Networks to Roles.

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/networks-roles-assignment-ui

* [python-tripleoclient] Update CLI to use tripleo-common actions for operations
  that currently modify mistral environment

  related bug: https://bugs.launchpad.net/tripleo/+bug/1635409

Dependencies
============

None.

Testing
=======

Feature will be tested as part of TripleO CI

Documentation Impact
====================

Documentation should be updated to reflect the new capabilities of GUI (Roles/Networks management),
a way to use plan-environment.yaml via CLI workflow and CLI/GUI interoperability using plan import
and export features.

References
==========

[1] http://lists.openstack.org/pipermail/openstack-dev/2017-February/111433.html
[2] https://specs.openstack.org/openstack/tripleo-specs/specs/ocata/gui-plan-import-export.html
[3] https://review.openstack.org/#/c/409921/
