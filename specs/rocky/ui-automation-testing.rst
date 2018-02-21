..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
UI Automation Testing
==========================================

https://blueprints.launchpad.net/tripleo/+spec/automated-ui-testing

We would like to introduce a suite of automated integration tests for the
TripleO UI.  This will prevent regressions, and will lead to more stable
software.

Problem Description
===================

At the moment, upstream CI only tests for lint errors, and runs our unit tests.
We'd like to add more integration tests for tripleo-ui to the CI pipeline.  This
will include a selenium-based approach.  This allows us to simulate a browser by
using a headless browser when running in CI, and we can detect a lot more
problems than we ever could with just unit testing.

Proposed Change
===============

Overview
--------

We would like write a Tempest plugin for tripleo-ui which uses Selenium to drive
a headless browser to execute the tests.  We chose Tempest because it's a
standard in OpenStack, and gives us nice error reporting.

We already have the `tempest-tripleo-ui`_ project set up.

We plan to write a CI job to run our code in Tempest.  In the initial
implementation, this will only cover checking for presence of certain UI
elements, and no deployments will actually be run.

Alternatives
------------

The alternative is that we do all of our testing manually, waste time, have
lower velocity, and have more bugs.

Security Impact
---------------

The security impact of this is minimal as it's CI-specific, and not user-facing.

Other End User Impact
---------------------

End users won't interact with this feature.

Performance Impact
------------------

This feature will only consume CI resources.  There should be no negative
resource impact on the End User.

Other Deployer Impact
---------------------

Our goal is to produce software that is more stable.  But we're not changing any
features, per se.

Developer Impact
----------------

Developers will gain a higher degree of confidence in their software.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hpokorny

Other contributors:
  ukalifon
  akrivoka

Work Items
----------

* Write Selenium tests
* Write Tempest plugin code to run Selenium tests
* Write a new openstack-infra job to run the Tempest plugin on ``check`` and
  ``gate``.  At first, this will be a simple sanity job to make sure that the UI
  has been rendered.  The CI job won't run a deployment.

Dependencies
============

* Tempest
* Selenium

Testing
=======

This is a bit meta.

Documentation Impact
====================

We will document how a developer who is new to the tripleo-ui project can get
started with writing new integration tests.

References
==========

.. _tempest-tripleo-ui: https://github.com/openstack/tempest-tripleo-ui

openstack-dev mailing list discussion:

* http://lists.openstack.org/pipermail/openstack-dev/2017-June/119185.html
* http://lists.openstack.org/pipermail/openstack-dev/2017-July/119261.html
