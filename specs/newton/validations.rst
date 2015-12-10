..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
TripleO Deployment Validations
==============================

We need ways in TripleO for performing validations at various stages of the
deployment.

Problem Description
===================

TripleO deployments, and more generally all OpenStack deployments, are complex,
error prone, and highly dependent on the environment. An appropriate set of
tools can help engineers to identify potential problems as early as possible
and fix them before going further with the deployment.

People have already developed such tools [1], however they appear more like
a random collection of scripts than a well integrated solution within TripleO.
We need to expose the validation checks from a library so they can be consumed
from the GUI or CLI without distinction and integrate flawlessly within TripleO
deployment workflow.

Proposed Change
===============

We propose to extend the TripleO Overcloud Deployment Mistral workflow [2] to
include Actions for validation checks.

These actions will need at least to:

* List validations
* Run and stop validations
* Get validation status
* Persist and retrieve validation results
* Permit grouping validations by 'deployment stage' and execute group operations

Running validations will be implemented in a workflow to ensure the nodes meet
certain expectations. For example, a baremetal validation may require the node
to boot on a ramdisk first.

Mistral workflow execution can be started with the `mistral execution-create`
command and can be stopped with the `mistral execution-update` command by
setting the workflow status to either SUCCESS or ERROR.

Every run of the workflow (workflow execution) is stored in Mistral's DB and
can be retrieved for later use. The workflow execution object contains all
information about the workflow and its execution, including all output data and
statuses for all the tasks composing the workflow.

By introducing a reasonable validation workflows naming, we are able to use
workflow names to identify stage at which the validations should run and
trigger all validations of given stage (e.g.
tripleo.validation.hardware.undercloudRootPartitionDiskSizeCheck)

Using the naming conventions, the user is also able to register a new
validation workflow and add it to the existing ones.

Alternatives
------------

One alternative is to ship a collection of scripts within TripleO to be run by
engineers at different stages of the deployment. This solution is not optimal
because it requires a lot of manual work and does not integrate with the UI.

Another alternative is to build our own API, but it would require significantly
more effort to create and maintain. This topic has been discussed at length on
the mailing list.

Security Impact
---------------

The whole point behind the validations framework is to permit running scripts
on the nodes, thus providing access from the control node to the deployed nodes
at different stages of the deployment. Special care needs to be taken to grant
access to the target nodes using secure methods and ensure only trusted scripts
can be executed from the library.

Other End User Impact
---------------------

We expect reduced deployment time thanks to early issue detection.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

Developers will need to keep the TripleO CI updated with changes, and will be
responsible for fixing the CI as needed.


Implementation
==============

Assignee(s)
-----------

Primary assignees:

* shadower
* mandre

Work Items
----------

The work items required are:

* Develop the tripleo-common Mistral actions that provide all of the
  functionality required for the validation workflow.
* Write an initial set of validation checks based on real deployment
  experience, starting by porting existing validations [1] to work with the
  implemented Mistral actions.

All patches that implement these changes must pass CI and add additional tests as
needed.


Dependencies
============

We are dependent upon the tripleo-mistral-deployment-library [2] work.


Testing
=======

The TripleO CI should be updated to test the updated tripleo-common library.


Documentation Impact
====================

Mistral Actions and Workflows are sort of self-documenting and can be easily
introspected by running 'mistral workflow-list' or 'mistral action-list' on the
command line.  The updated library however will have to be well-documented and
meet OpenStack standards.  Documentation will be needed in both the
tripleo-common and tripleo-docs repositories.


References
==========

* [1] Set of tools to help detect issues during TripleO deployments:
  https://github.com/rthallisey/clapper
* [2] Library support for TripleO Overcloud Deployment Via Mistral:
  https://specs.openstack.org/openstack/tripleo-specs/specs/mitaka/tripleo-mistral-deployment-library.html
