..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
In-flight Validations for the overcloud
=======================================


https://blueprints.launchpad.net/tripleo/+spec/inflight-validations

Currently, we don't have any way to run validations inside a deploy run. This
spec aims to provide the necessary information on how to implement such
in-flight validations for an overcloud deploy.

Problem Description
===================

Currently, operators and developers have to wait a long time before getting an
error in case a service isn't running as expected.

This leads to loss of time and resources.

Proposed Change
===============

Overview
--------

After each container/service is started, a new step is added to run one or more
validations on the deployed host in order to ensure the service is actually
working as expected at said step.

These validations must not use Mistral Workflow, in order to provide support
for the undercloud/standalone case.

The best way to push those validations would be through the already existing
``deploy_steps_tasks`` keywork. A validation should be either at the start
of the next step, or at the end of the current step we want to check.

The validations should point to an external playbook, for instance hosted in
``tripleo-validations``. If there isn't real use to create a playbook for the
validation, it might be inline - but it must be short, for example a single test
for an open port.

Alternatives
------------

There isn't really other alternative. We might think running the validation
ansible playbook directly is a good idea, but it will break the wanted
convergence with the UI.

For now, there isn't such validations, we can start fresh.

Security Impact
---------------

No security impact.

Upgrade Impact
--------------

If a service isn't starting properly, the upgrade might fail. This is also true
for a fresh deploy.

We might want different validation tasks/workflows if we're in an upgrade
state.

Other End User Impact
---------------------

End user will get early failure in case of issues detected by the validations.
This is an improvement, as for now it might fail at a later step, and might
break things due to the lack of valid state.

Performance Impact
------------------

Running in-flight validation WILL slow the overall deploy/upgrade process, but
on the other hand, it will ensure we have a clean state before each step.

Other Deployer Impact
---------------------

No other deployer impact.

Developer Impact
----------------

Validations will need to be created and documented in order to get proper runs.


Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  cjeanner

Other contributors:
  <launchpad-id or None>

Work Items
----------

* Add new hook for the ``validation_tasks``
* Provide proper documentation on its use

Dependencies
============

* Please keep in mind the Validation Framework spec when implementing things:
  https://review.openstack.org/589169


Testing
=======

TBD


Documentation Impact
====================

What is the impact on the docs? Don't repeat details discussed above, but
please reference them here.


References
==========

* https://review.openstack.org/589169
