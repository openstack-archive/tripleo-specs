================================================
Library support for TripleO Overcloud Deployment
================================================

We need a TripleO library that supports the overcloud deployment workflow.

Problem Description
===================

With Tuskar insufficient for complex overcloud deployments, TripleO has moved to
an overcloud deployment workflow that bypasses Tuskar.  This workflow can be
summarized as follows:

 * The user edits the templates and environment file.  These can be stored
   anywhere.
 * Templates may be validated by Heat.
 * Templates and environment are sent to Heat for overcloud deployment.
 * Post-deploy, overcloud endpoints are configured.

This workflow is already supported by the CLI.

However from a GUI perspective, although the workflow is straightforward, it is
not simple.  Here are some of the complications that arise:

 * Some of the business logic in this workflow is contained in the CLI itself,
   making it difficult for other UIs to use.
 * If the TripleO overcloud deployment workflow changes, it is easy for the CLI
   and GUI approach to end up on divergent paths - a dangerous situation.
 * The CLI approach allows open-ended flexibility (the CLI doesn't care where the
   templates come from) that is detrimental for a GUI (the GUI user doesn't care
   where the templates are stored, but consistency in approach is desirable to
   prevent divergence among GUIs).

There is a need to create common code that accommodates the flexibility of the
CLI with the ease-of-use needs of Python-based GUI consumers.  Note that an API
will eventually be needed in order to accommodate non-Python GUIs.  The work
there will be detailed in a separate spec.

Proposed Change
===============

In order to solve this problem, we propose the following:

  * Encapsulate the business logic involved in the overcloud deployment workflow
    within the tripleo-common library.
  * Provide a simplified workflow to hide unneeded complexity from GUI consumers
    - for example, template storage.
  * Update the CLI to use this code where appropriate to prevent divergence with
    GUIs.

The first two points deserve further explanation.  First, let us lay out the
proposed GUI workflow.  We will refer to the Heat files the user desires to use
for the overcloud deployment as a 'plan'.

1. A user creates a plan by pushing a copy of the Heat deployment templates into
   a data store.
2. The user defines values for the template resource types given by Heat
   template capabilities.  This results in an updated resource registry in an
   environment file saved to the data store.
   (https://review.openstack.org/#/c/196656/7/specs/liberty/resource-capabilities.rst)
   Note that this spec will be completed by mitaka at the earliest.  A
   workaround is discussed below.
3. Now that the template resource types are specified, the user can configure
   deployment parameters given by Heat.  Edited parameters are updated and an
   updated environment file is saved to the data store.  'Roles' no longer exist
   in Tuskar, but can still be derived from available Heat parameters.
   (https://review.openstack.org/#/c/197199/5/specs/liberty/nested-validation.rst)
4. Steps 2 and 3 can be repeated.
5. With configuration complete, the user triggers the deployment of the
   overcloud.  The templates and environment file are taken from the data store
   and sent to Heat.
6. Once overcloud deployment is complete, any needed post-deploy config is
   performed.

In order to fulfill this workflow, we propose to initially promote the use of
Swift as the template data store.  This usage will be abstracted away behind
the tripleo-common library, and later updates may allow the use of other data
stores.

Note that the Swift-workflow is intended to be an alternative to the current CLI
'--templates' workflow.  Both would end up being options under the CLI; a user
could choose '--templates' or '--plan'.  However they would both be backed by
common tripleo-common library code, with the '--plan' option simply calling
additional functions to pull the plan information from Swift.  And GUIs that
expect a Swift-backed deployment would lose functionality if the deployment
is deployed using the '--templates' CLI workflow.

The tripleo-common library functions needed are:

 * **Plan CRUD**

   * **create_plan(plan_name, plan_files)**: Creates a plan by creating a Swift
     container matching plan_name, and placing all files needed for that plan
     into that container (for Heat that would be the 'parent' templates, nested
     stack templates, environment file, etc).  The Swift container will be
     created with object versioning active to allow for versioned updates.
   * **get_plan(plan_name)**: Retrieves the Heat templates and environment file
     from the Swift container matching plan_name.
   * **update_plan(plan_name, plan_files)**: Updates a plan by updating the
     plan files in the Swift container matching plan_name.  This may necessitate
     an update to the environment file to add and/or remove parameters. Although
     updates are versioned, retrieval of past versions will not be implemented
     until the future.
   * **delete_plan(plan_name)**: Deletes a plan by deleting the Swift container
     matching plan_name, but only if there is no deployed overcloud that was
     deployed with the plan.

 * **Deployment Options**

   * **get_deployment_plan_resource_types(plan_name)**: Determine available
     template resource types by retrieving plan_name's templates from Swift and
     using the proposed Heat resource-capabilities API
     (https://review.openstack.org/#/c/196656/7/specs/liberty/resource-capabilities.rst).
     If that API is not ready in the required timeframe, then we will implement
     a temporary workaround - a manually created map between templates and
     provider resources.  We would work closely with the spec developers to try
     and ensure that the output of this method matches their proposed output, so
     that once their API is ready, replacement is easy.
   * **update_deployment_plan_resource_types(plan_name, resource_types)**:
     Retrieve plan_name's environment file from Swift and update the
     resource_registry tree according to the values passed in by resource_types.
     Then update the environment file in Swift.

 * **Deployment Configuration**

   * **get_deployment_parameters(plan_name)**: Determine available deployment
     parameters by retrieving plan_name's templates from Swift and using the
     proposed Heat nested-validation API call
     (https://review.openstack.org/#/c/197199/5/specs/liberty/nested-validation.rst).
   * **update_deployment_parameters(plan_name, deployment_parameters)**:
     Retrieve plan_name's environment file from Swift and update the parameters
     according to the values passed in by deployment_parameters.  Then update the
     environment file in Swift.
   * **get_deployment_roles(plan_name)**: Determine available deployment roles.
     This can be done by retrieving plan_name's deployment parameters and
     deriving available roles from parameter names; or by looking at the top-
     level ResourceGroup types.

 * **Deployment**

   * **validate_plan(plan_name)**: Retrieve plan_name's templates and environment
     file from Swift and use them in a Heat API validation call.
   * **deploy_plan(plan_name)**: Retrieve plan_name's templates and environment
     file from Swift and use them in a Heat API call to create the overcloud
     stack.  Perform any needed pre-processing of the templates, such as the
     template file dictionary needed by Heat.  This function will return a Heat
     stack ID that can be used to monitor the status of the deployment.

 * **Post-Deploy**

   * **postdeploy_plan(plan_name)**: Initialize the API endpoints of the
     overcloud corresponding to plan_name.

Alternatives
------------

The alternative is to force non-CLI UIs to re-implement the business logic
currently contained within the CLI.  This is not a good alternative.

Security Impact
---------------

Other End User Impact
---------------------

The --templates workflow will end up being modified to use the updated
tripleo-common library.

Python-based code would find it far easier to adapt the TripleO method of
deployment.  This may result in increased usage.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Right now, changing the overcloud deployment workflow results in stress due to
the need to individually update both the CLI and GUI code.  Converging the two
makes this a far easier proposition.  However developers will need to have this
architecture in mind and ensure that changes to the --templates or --plan
workflow are maintained in the tripleo-common library (when appropriate) to
avoid unneeded divergences.

Another important item to note is that we will need to keep the TripleO CI
updated with changes, and will be responsible for fixing the CI as needed.


Implementation
==============

Assignee(s)
-----------
Primary assignees:

* tzumainn
* akrivoka
* jtomasek
* dmatthews

Work Items
----------

The work items required are:

 * Develop the tripleo-common library to provide the functionality described
   above.  This also involves moving code from the CLI to tripleo-common.
 * Update the CLI to use the tripleo-common library.

All patches that implement these changes must pass CI and add additional tests as
needed.


Dependencies
============

We are dependent upon two HEAT specs:

 * Heat resource-capabilities API
   (https://review.openstack.org/#/c/196656/7/specs/liberty/resource-capabilities.rst)
 * Heat nested-validation API
   (https://review.openstack.org/#/c/197199/5/specs/liberty/nested-validation.rst)

Testing
=======

The TripleO CI should be updated to test the updated tripleo-common library.

Documentation Impact
====================

The updated library with its Swift-backed workflow will have to be well-
documented and meet OpenStack standards.  Documentation will be needed in both
the tripleo-common and tripleo-docs repositories.

References
==========
