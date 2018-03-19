..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
TripleO tools for testing HA deployments
=============================================

We need a way to verify a Highly Available TripleO deployment with proper tests
that check if the HA bits are behaving correctly.

Problem Description
===================

Currently, we test HA behavior of TripleO deployments only by deploying
environments with three controllers and see if we're able to spawn an instance,
but this is not enough.

There should be a way to verify the HA capabilities of deployments, and if the
behavior of the environment is still correct after inducted failures,
simulated outages and so on.

This tool should be a standalone component to be included by the user if
necessary, without breaking any of the dynamics present in TripleO.

Proposed Change
===============

Overview
--------

The proposal is to create an Ansible based project named tripleo-ha-utils that
will be consumable by the various tools that we use to deploy TripleO
environments like tripleo-quickstart or infrared or by manual deployments.

The project will initially cover three principal roles:

* **stonith-config**: a playbook used to automate the creation of fencing
  devices in the overcloud;
* **instance-ha**: a playbook that automates the seventeen manual steps needed
  to configure instance HA in the overcloud, test them via rally and verify
  that instance HA works appropriately;
* **validate-ha**: a playbook that runs a series of disruptive actions in the
  overcloud and verifies it always behaves correctly by deploying a
  heat-template that involves all the overcloud components;

Today the project exists outside the TripleO umbrella, and it is named
tripleo-quickstart-utils [1]  (see "Alternatives" for the historical reasons of
this name). It is used internally inside promotion pipelines, and has
also been tested with success in RDOCloud.

Pluggable implementation
~~~~~~~~~~~~~~~~~~~~~~~~

The base principle of the project is to give people the ability to integrate
the first roles with whatever kind of test. For example, today we're using
a simple bash framework to interact with the cluster (so pcs commands and
other interactions), rally to test instance-ha and Ansible itself to simulate
full power outage scenarios.
The idea is to keep this pluggable approach leaving the final user the choice
about what to use.

Retro compatibility
~~~~~~~~~~~~~~~~~~~

One of the aims of this project is to be retro-compatible with the previous
version of OpenStack. Starting from Liberty, we cover instance-ha and
stonith-config Ansible playbooks for all the releases.
The same happens while testing HA since all the tests are plugged in depending
on the release.

Alternatives
------------

While evaluating alternatives, the first thing to consider is that this
project aims to be a TripleO-centric set of tools for HA, not a generic
OpenStack's one.
We want tools to help the user answer questions like "Is the Galera bundle
cluster resource able to tolerate a stop and a consecutive start without
affecting the environment capabilities?" or "Is the environment able to
evacuate instances after being configured for Instance HA?". And the answer we
want is YES or NO.

* *tripleo-validations*: the most logical place to put this, at least
looking at the name, would be tripleo-validations. By talking with folks
working on it, it came out that the meaning of tripleo-validations project is
not doing disruptive tests. Integrating this stuff would be out of scope.

* *tripleo-quickstart-extras*: apart from the fact that this is not
something meant just for quickstart (the project supports infrared and
"plain" environments as well) even if we initially started there, in the
end, it came out that nobody was looking at the patches since nobody was
able to verify them. The result was a series of reviews stuck forever.
So moving back to extras would be a step backward.

Other End User Impact
---------------------

None. The good thing about this solution is that there's no impact for anyone
unless the solution gets loaded inside an existing project. Since this will be
an external project, it will not impact anything of the current stuff.

Performance Impact
------------------

None. Unless the deployments, the CI runs or whatever include the roles there
will be no impact, and so the performances will not change.

Implementation
==============

Primary assignees:

* rscarazz

Work Items
----------

* Import the tripleo-quickstart-utils [1] as a new repository and start new
  deployments from there.

Testing
=======

Due to the disruptive nature of these tests, the TripleO CI should not be
updated to include these tests, mostly because of timing issues.
This project should remain optionally usable by people when needed, or in
specific CI environments meant to support longer than usual jobs.

Documentation Impact
====================

All the implemented roles are today fully documented in the
tripleo-quickstart-utils [1] project, so importing its repository as is will
also give its full documentation.

References
==========

[1] Original project to import as new
    https://github.com/redhat-openstack/tripleo-quickstart-utils
