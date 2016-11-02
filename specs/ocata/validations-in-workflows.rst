..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Validations in TripleO Workflows
================================

https://blueprints.launchpad.net/tripleo/+spec/validations-in-workflows

The Newton release introduced TripleO validations -- a set of
extendable checks that identify potential deployment issues early and
verify that the deployed OpenStack is set up properly. These
validations are automatically being run by the TripleO UI, but there
is no support for the command line workflow and they're not being
exercised by our CI jobs either.


Problem Description
===================

When enabled, TripleO UI runs the validations at the appropriate phase
of the planning and deployment. This is done within the TripleO UI
codebase and therefore not available to python-tripleoclient or
the CI.

The TripleO deployer can run the validations manually, but they need
to know at which point to do so and they will need to do it by calling
Mistral directly.

This causes a disparity between the command line and GUI experience
and complicates the efforts to exercise the validations by the CI.


Proposed Change
===============

Overview
--------

Each validation already advertises where in the planning/deployment
process it should be run. This is under the ``vars/metagata/groups``
section. In addition, the ``tripleo.validations.v1.run_groups``
Mistral workflow lets us run all validations belonging to a given
group.

For each validation group (currently ``pre-introspection``, ``pre-deployment``
and ``post-deployment``) we will update the appropriate workflow in
tripleo-common to optionally call ``run_groups``.

Each of the workflows above will receive a new Mistral input called
``run_validations``. It will be a boolean value that indicates whether
the validations ought to be run as part of that workflow or not.

To expose this functionality to the command line user, we will add an
option for enabling/disabling validations into python-tripleoclient
(which will set the ``run_validations`` Mistral input) and a way to
show the results of each validation to the screen output.

When the validations are run, they will report their status to Zaqar
and any failures will block the deployment. The deployer can disable
validations if they wish to proceed despite failures.

One unresolved question is the post-deployment validations. The Heat
stack create/update Mistral action is currently asynchronous and we
have no way of calling actions after the deployment has finished.
Unless we change that, the post-deployment validations may have to be
run manually (or via python-tripleoclient).


Alternatives
------------

1. Document where to run each group and how and leave it at that. This
   risks that the users already familiar with TripleO may miss the
   validations or that they won't bother.

   We would still need to find a way to run validations in a CI job,
   though.

2. Provide subcommands to run validations (and groups of validations)
   into python-tripleoclient and rely on people running them manually.

   This is similar to 1., but provides an easier way of running a
   validation and getting its result.

   Note that this may be a useful addition even if with the proposal
   outlined in this specification.

3. Do what the GUI does in python-tripleoclient, too. The client will
   know when to run which validation and will report the results back.

   The drawback is that we'll need to implement and maintain the same
   set of rules in two different codebases and have no API to do them.
   I.e. what the switch to Mistral is supposed to solve.



Security Impact
---------------

None

Other End User Impact
---------------------

We will need to modify python-tripleoclient to be able to display the
status of validations once they finished. TripleO UI already does this.

The deployers may need to learn about the validations.

Performance Impact
------------------

Running a validation can take about a minute (this depends on the
nature of the validation, e.g. does it check a configuration file or
does it need to log in to all compute nodes).

This may can be a concern if we run multiple validations at the same
time.

We should be able to run the whole group in parallel. It's possible
we're already doing that, but this needs to be investigated.
Specifically, does ``with-items`` run the tasks in sequence or in
parallel?

There are also some options that would allow us to speed up the
running time of a validation itself, by using common ways of speeding
up Ansible playbooks in general:

* Disabling the default "setup" task for validations that don't need
  it (this task gathers hardware and system information about the
  target node and it takes some time)
* Using persistent SSH connections
* Making each validation task run independently (by default, Ansible
  runs a task on all the nodes, waits for its completion everywhere
  and then moves on to another task)
* Each validation runs the ``tripleo-ansible-inventory`` script which
  gathers information about deployed servers and configuration from
  Mistral and Heat. Running this script can be slow. When we run
  multiple validations at the same time, we should generate the
  inventory only once and cache the results.

Since the validations are going to be optional, the deployer can
always choose not to run them. On the other hand, any slowdown should
ideally outweigh the time spent investigating failed deployments.

We will also document the actual time difference. This information
should be readily available from our CI environments, but we should
also provide measurements on the bare metal.


Other Deployer Impact
---------------------

Depending on whether the validations will be run by default or not,
the only impact should be an option that lets the deployer to run them
or not.


Developer Impact
----------------

The TripleO developers may need to learn about validations, where to
find them and how to change them.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tsedovic

Other contributors:
  None

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.

* Add ``run_validations`` input and call ``run_groups`` from the
  deployment and node registration workflows
* Add an option to run the validations to python-tripleoclient
* Display the validations results with python-tripleoclient
* Add or update a CI job to run the validations
* Add a CI job to tripleo-validations


Dependencies
============

None


Testing
=======

This should make the validations testable in CI. Ideally, we would
verify the expected success/failure for the known validations given
the CI environment. But having them go through the testing machinery
would be a good first step to ensure we don't break anything.


Documentation Impact
====================

We will need to document the fact that we have validations, where they
live and when and how are they being run.


References
==========

* http://docs.openstack.org/developer/tripleo-common/readme.html#validations
* http://git.openstack.org/cgit/openstack/tripleo-validations/
* http://docs.openstack.org/developer/tripleo-validations/
