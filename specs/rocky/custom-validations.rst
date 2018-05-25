..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add Support for Custom TripleO Validations
==========================================

https://blueprints.launchpad.net/tripleo/+spec/custom-validations

All validations are currently stored in a single directory. This makes
it inconvenient to try and write new validations, update from a remote
repository or to add an entirely new (perhaps private) source.


Problem Description
===================

* The deployer wants to develop and test their own validations in a
  personal checkout without risking changes to the default ones.

* The deployer wants to use a stable release of TripleO but consume
  the latest validations because they are non-disruptive and check for
  more stuff.

* A third party has developed validations specific to their product
  that they don't want to or can't include in the tripleo-validations
  repository.



Proposed Change
===============

Overview
--------

We will store a default set of TripleO validations in a Swift container called
``tripleo-validations``. These will be shared across all plans and are not
expected to be updated by the deployer. This container should be created on
initial undercloud deployment.

We will provide a mechanism for deployers to add a custom set of validations
per deployment plan. These plan-specific validations will be stored in a
``custom-validations`` subdirectory in the plan's Swift container. Storing them
together with the plan makes sense as these validations can be specific to
particular deployment plan configuration, as well as makes the import/export
easier.

Since custom validation will be stored as part of the plan, no additional
workflows/actions to perform CRUD operations for them will be necessary; we can
simply use the existing plan create/update for this purpose.

The validation Mistral actions (e.g. ``list`` and ``run_validation``)
will need to be updated to take into account this new structure of
validations. They will need to look for validations in the
``tripleo-validations`` Swift container (for default validations) and the
plan's ``custom-validations`` subdirectory (for custom validations), instead of
sourcing them from a directory on disk, as they are doing now.

If a validation with the same name is found both in default in custom
validations, we will always pick the one stored in custom validations.

.. note:: As a further iteration, we can implement validations as per-service
          tasks in standalone service Ansible roles. They can then be consumed
          by tripleo-heat-templates service templates.

Alternatives
------------

* Do nothing. The deployers can already bring in additional
  validations, it's just less convenient and potentially error-prone.

* We could provide a know directory structure conceptually similar to
  ``run-parts`` where the deployers could add their own validation
  directories.


Security Impact
---------------

None

Other End User Impact
---------------------

In order to add their own validations, the deployer will need to
update the deployment plan by adding a ``custom-validations`` directory to it,
and making sure this directory contains the desired custom validations. The
plan update operation is already supported in the CLI and the UI.

Performance Impact
------------------

Since the validation sources will now be Swift containers, downloading
validations will potentially be necessary on each run. We will have to keep an
eye on this an potentially introduce caching if this turns out to be a problem.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Testing and developing new validations in both development and
production environments will be easier with this change.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
  * akrivoka

Other contributors:
  * florianf

Work Items
----------
* Move to using Swift as default storage for tripleo-validations ([1]_).

* Update ``load_validations`` and ``find_validation`` functions to
  read validations from all the sources specified in this document.

Dependencies
============

In order to be able to implement this new functionality, we first need to have
the validations use Swift as the default storage. In other words, this spec
depends on the blueprint [1]_.

Testing
=======

The changes will be unit-tested in all the tripleo repos that related
changes land in (tripleo-common, instack-undercloud, tripleo-heat-templates,
etc).

We could also add a new CI scenario that would have a custom-validations
directory within a plan set up.


Documentation Impact
====================

We will need to document the format of the new custom-validations plan
subdirectory and the new behaviour this will introduce.


References
==========

.. [1] https://blueprints.launchpad.net/tripleo/+spec/store-validations-in-swift
