..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
GUI: Import/Export Deployment Plan
==================================

Add two features to TripleO UI:

* Import a deployment plan with a Mistral environment
* Export a deployment plan with a Mistral environment

Blueprint: https://blueprints.launchpad.net/tripleo/+spec/gui-plan-import-export


Problem Description
===================

Right now, the UI only supports simple plan creation. The user needs to upload
the plan files, make the environment selection and set the parameters. We want
to add a plan import feature which would allow the user to import the plan
together with a complete Mistral environment. This way the selection of the
environment and parameters would be stored and automatically imported, without
any need for manual configuration.

Conversely, we want to allow the user to export a plan together with a Mistral
environment, using the UI.


Proposed Change
===============

Overview
--------

In order to identify the Mistral environment when importing a plan, I propose
we use a JSON formatted file and name it 'plan-environment.json'. This file
should be uploaded to the Swift container together with the rest of the
deployment plan files. The convention of calling the file with a fixed name is
enough for it to be detected. Once this file is detected by the tripleo-common
workflow handling the plan import, the workflow then creates (or updates) the
Mistral environment using the file's contents. In order to avoid possible future
unintentional overwriting of environment, the workflow should delete this file
once it has created (or updated) the Mistral environment with its contents.

Exporting the plan should consist of downloading all the plan files from the
swift container, adding the plan-environment.json, and packing it all up in
a tarball.

Alternatives
------------

One alternative is what we have now, i.e. making the user perform all the
environment configuration settings and parameter settings manually each time.
This is obviously very tedious and the user experience suffers greatly as a
result.

The alternative to deleting the plan-environment.json file upon its
processing is to leave in the swift container and keep it in sync with all
the updates that might happen thereafter. This can get very complicated and is
entirely unnecessary, so deleting the file instead is a better choice.

Security Impact
---------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

The import and export features will only be triggered on demand (user clicks
on button, or similar), so they will have no performance impact on the rest
of the application.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  akrivoka

Other contributors:
  jtomasek
  d0ugal

Work Items
----------

* tripleo-common: Enhance plan creation/update to consume plan-environment.json

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/enhance-plan-creation-with-plan-environment-json

* tripleo-common: Add plan export workflow

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/plan-export-workflow

* python-tripleoclient: Add plan export command

  blueprint: https://blueprints.launchpad.net/tripleo/+spec/plan-export-command

* tripleo-ui: Integrate plan export into UI

  bluerpint: https://blueprints.launchpad.net/tripleo/+spec/plan-export-gui

Note: We don't need any additional UI (neither GUI nor CLI) for plan import - the
existing GUI elements and CLI command for plan creation can be used for import
as well.


Dependencies
============

None


Testing
=======

The changes should be covered by unit tests in tripleo-ui, tripleo-common and
python-tripleoclient.


Documentation Impact
====================

User documentation should be enhanced by adding instructions on how these two
features are to be used.


References
==========

None