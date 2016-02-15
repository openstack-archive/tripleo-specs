============================================================
Library support for TripleO Overcloud Deployment Via Mistral
============================================================

We need a TripleO library that supports the overcloud deployment workflow.

Problem Description
===================

TripleO has an overcloud deployment workflow that uses Heat templates and uses
the following steps:

* The user edits the templates and environment file.  These can be stored
  anywhere.
* Templates may be validated by Heat.
* Templates and environment are sent to Heat for overcloud deployment.

This workflow is already supported by the CLI.

However from a GUI perspective, although the workflow is straightforward, it is
not simple.  Here are some of the complications that arise:

* Some of the business logic in this workflow is contained in the CLI itself,
  making it difficult for other UIs to use.
* If the TripleO overcloud deployment workflow changes, it is easy for the CLI
  and GUI approach to end up on divergent paths - a dangerous situation.
* The CLI approach allows open-ended flexibility (the CLI doesn't care where
  the templates come from) that is detrimental for a GUI (the GUI user doesn't
  care where the templates are stored, but consistency in approach is desirable
  to prevent divergence among GUIs and CLIs).

There is a need to create common code that accommodates the flexibility of the
CLI with the ease-of-use needs of GUI consumers.

Proposed Change
===============

In order to solve this problem, we propose to create a Mistral-integrated
deployment with the following:

* Encapsulate the business logic involved in the overcloud deployment workflow
  within the tripleo-common library utilizing Mistral actions and workflows.
* Provide a simplified workflow to hide unneeded complexity from GUI consumers
* Update the CLI to use this code where appropriate to prevent divergence with
  GUIs.

The first three points deserve further explanation.  First, let us lay out the
proposed GUI workflow.

1. A user pushes the Heat deployment templates into swift.
2. The user defines values for the template resource types given by Heat
   template capabilities which are stored in an environment[1]. Note that this
   spec will be completed by mitaka at the earliest.  A workaround is discussed
   below.
3. Now that the template resource types are specified, the user can configure
   deployment parameters given by Heat.  Edited parameters are updated and are
   stored in an environment.  'Roles' can still be derived from available Heat
   parameters[2].
4. Steps 2 and 3 can be repeated.
5. With configuration complete, the user triggers the deployment of the
   overcloud.  The templates and environment file are taken from Swift
   and sent to Heat.
6. Once overcloud deployment is complete, any needed post-deploy config is
   performed.

The CLI and GUI will both use the Swift workflow and store the templates into
Swift.  This would facilitate the potential to switch to the UI from a CLI based
deployment and vice-versa.

Mistral Workflows are composed of Tasks, which group together one or more
Actions to be executed with a Workflow Execution.  The Action is implemented as
a class with an initialization method and a run method.  The run method provides
a single execution point for Python code.  Any persistence of state required for
Actions or Workflows will be stored in a Mistral Environment object.

In some cases, an OpenStack Service may be missing a feature needed for TripleO
or it might only be accessible through its associated Python client.  To
mitigate this issue in the short term, some of the Actions will need to be
executed directly with an Action Execution [3] which calls the Action directly and
returns instantly, but also doesn't have access to the same context as a
Workflow Execution.  In theory, every action execution should be replaced by an
OpenStack service API call.

Below is a summary of the intended Workflows and Actions to be executed from the
CLI or the GUI using the python-mistralclient or Mistral API.  There may be
additional actions or library code necessary to enable these operations that
will not be intended to be consumed directly.

Workflows:

 * Node Registration
 * Node Introspection
 * Plan Creation
 * Plan Deletion
 * Deploy
 * Validation Operations

Actions:

 * Plan List
 * Get Capabilites
 * Update Capabilities
 * Get Parameters
 * Update Parameters
 * Roles List

For Flavors and Image management, the Nova and Glance APIs will be used
respectively.

The registration and introspection of nodes will be implemented within a
Mistral Workflow.  The logic is currently in tripleoclient and will be ported,
as certain node configurations are specified as part of the logic (ramdisk,
kernel names, etc.) so the user does not have to specify those.  Tagging,
listing and deleting nodes will happen via the Ironic/Inspectors APIs as
appropriate.

A deployment plan consists of a collection of heat templates in a Swift
container, combined with data stored in a Mistral Environment.  When the plan is
first created, the capabilities map data will be parsed and stored in the
associated Mistral Environment.  The templates will need to be uploaded to a
Swift container with the same name as the stack to be created.  While any user
could use a raw POST request to accomplish this, the GUI and CLI will provide
convenience functions improve the user experience.  The convenience functions
will be implemented in an Action that can be used directly or included in a
Workflow.

The deletion of a plan will be implemented in a Workflow to ensure there isn't
an associated stack before deleting the templates, container and Mistral
Environment.  Listing the plans will be accomplished by calling
'mistral environment-list'.

To get a list of the available Heat environment files with descriptions and
constraints, the library will have an Action that returns the information about
capabilities added during plan creation and identifies which Heat environment
files have already been selected.  There will also be an action that accepts a
list of user selected Heat environment files and stores the information in the
Mistral Environment.  It would be inconvenient to use a Workflow for these
actions as they just read or update the Mistral Environment and do not require
additional logic.

The identification of Roles will be implemented in a Workflow that calls out to
Heat.

To obtain the deployment parameters, Actions will be created that will call out
to heat with the required template information to obtain the parameters and set
the parameter values to the Environment.

To perform TripleO validations, Workflows and associated Actions will be created
to support list, start, stop, and results operations.  See the spec [4] for more
information on how the validations will be implemented with Mistral.

Alternatives
------------

One alternative is to force non-CLI UIs to re-implement the business logic
currently contained within the CLI.  This is not a good alternative.  Another
possible alternative would be to create a REST API [5] to abstract TripleO
deployment logic, but it would require considerably more effort to create and
maintain and has been discussed at length on the mailing list. [6][7]

Security Impact
---------------

Other End User Impact
---------------------

The --templates workflow will end up being modified to use the updated
tripleo-common library.

Integrating with Mistral is a straightforward process and this may result in
increased usage.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Rather than write workflow code in python-tripleoclient directly developers will
now create Mistral Actions and Workflows that help implement the requirements.

Right now, changing the overcloud deployment workflow results in stress due to
the need to individually update both the CLI and GUI code.  Converging the two
makes this a far easier proposition.  However developers will need to have this
architecture in mind and ensure that changes to the --templates or --plan
workflow are maintained in the tripleo-common library (when appropriate) to
avoid unneeded divergences.

Implementation
==============

Assignee(s)
-----------
Primary assignees:

* rbrady
* jtomasek
* dprince

Work Items
----------
The work items required are:

* Develop the tripleo-common Mistral actions that provide all of the
  functionality required for our deployment workflows.
* This involves moving much of the code out of python-tripleoclient and into
  generic, narrowly focused, Mistral actions that can be consumed via the
  Mistral API.
* Create new Mistral workflows to help with high level things like deployment,
  introspection, node registration, etc.
* tripleo-common is more of an internal library, and its logic is meant to be
  consumed (almost) solely by using Mistral
  actions. Projects should not attempt to circumvent the API by using
  tripleo-common as a library as much as possible.
  There may be some exceptions to this for common polling functions, etc. but in
  general all core workflow logic should be API driven.
* Update the CLI to consume these Mistral actions directly via
  python-mistralclient.

All patches that implement these changes must pass CI and add additional tests
as needed.

Dependencies
============

None


Testing
=======

The TripleO CI should be updated to test the updated tripleo-common library.

Our intent is to make tripleoclient consume Mistral actions as we write them.
Because all of the existing upstream Tripleo CI release on tripleoclient taking
this approach ensures that our all of our workflow actions always work. This
should get us coverage on 90% of the Mistral actions and workflows and allow us
to proceed with the implementation iteratively/quickly. Once the UI is installed
and part of our upstream CI we can also rely on coverage there to ensure we
don't have breakages.

Documentation Impact
====================

Mistral Actions and Workflows are sort of self-documenting and can be easily
introspected by running 'mistral workflow-list' or 'mistral action-list' on the
command line.  The updated library however will have to be well-documented and
meet OpenStack standards.  Documentation will be needed in both the
tripleo-common and tripleo-docs repositories.

References
==========

[1] https://specs.openstack.org/openstack/heat-specs/specs/mitaka/resource-capabilities.html

[2] https://specs.openstack.org/openstack/heat-specs/specs/liberty/nested-validation.html

[3] http://docs.openstack.org/developer/mistral/terminology/executions.html

[4] https://review.openstack.org/#/c/255792/

[5] http://specs.openstack.org/openstack/tripleo-specs/specs/mitaka/tripleo-overcloud-deployment-library.html

[6] http://lists.openstack.org/pipermail/openstack-dev/2016-January/083943.html

[7] http://lists.openstack.org/pipermail/openstack-dev/2016-January/083757.html

