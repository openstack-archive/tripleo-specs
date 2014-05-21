..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Tuskar Plan REST API Specification
==================================

Blueprint:
https://blueprints.launchpad.net/tuskar/+spec/tripleo-juno-tuskar-plan-rest-api

In Juno, the Tuskar API is moving towards a model of being a large scale
application planning service. Its initial usage will be to deploy OpenStack
on OpenStack by leveraging TripleO Heat Templates and fitting into the
greater TripleO workflow.

As compared to Icehouse, Tuskar will no longer make calls to Heat for creating
and updating a stack. Instead, it will serve to define and manipulate the Heat
templates for describing a cloud. Tuskar will be the source for the cloud
planning while Heat is the source for the state of the live cloud.

Tuskar employs the following concepts:

* *Deployment Plan* - The description of an application (for example,
  the overcloud) being planned by Tuskar. The deployment plan keeps track of
  what roles will be present in the deployment and their configuration values.
  In TripleO terms, each overcloud will have its own deployment plan that
  describes what services will run and the configuration of those services
  for that particular overcloud. For brevity, this is simply referred to as
  the "plan" elsewhere in this spec.
* *Role* - A unit of functionality that can be added to a plan. A role
  is the definition of what will run on a single server in the deployed Heat
  stack. For example, an "all-in-one" role may contain all of the services
  necessary to run an overcloud, while a "compute" role may provide only the
  nova-compute service.

Put another way, Tuskar is responsible for assembling
the user-selected roles and their configuration into a Heat environment and
making the built Heat templates and files available to the caller (the
Tuskar UI in TripleO but, more generally, any consumer of the REST API) to send
to Heat.

Tuskar will ship with the TripleO Heat Templates installed to serve as its
roles (dependent on the conversions taking place this release [4]_).
For now it is assumed those templates are installed as part of the TripleO's
installation of Tuskar. A different spec will cover the API calls necessary
for users to upload and manipulate their own custom roles.

This specification describes the REST API clients will interact with in
Tuskar, including the URLs, HTTP methods, request, and response data, for the
following workflow:

* Create an empty plan in Tuskar.
* View the list of available roles.
* Add roles to the plan.
* Request, from Tuskar, the description of all of the configuration values
  necessary for the entire plan.
* Save user-entered configuration values with the plan in Tuskar.
* Request, from Tuskar, the Heat templates for the plan, which includes
  all of the files necessary to deploy the configured application in Heat.

The list roles call is essential to this workflow and is therefore described
in this specification. Otherwise, this specification does not cover the API
calls around creating, updating, or deleting roles. It is assumed that the
installation process for Tuskar in TripleO will take the necessary steps to
install the TripleO Heat Templates into Tuskar. A specification will be filed
in the future to cover the role-related API calls.


Problem Description
===================

The REST API in Tuskar seeks to fulfill the following needs:

* Flexible selection of an overcloud's functionality and deployment strategy.
* Repository for discovering what roles can be added to a cloud.
* Help the user to avoid having to manually manipulate Heat templates to
  create the desired cloud setup.
* Storage of a cloud's configuration without making the changes immediately
  live (future needs in this area may include offering a more structured
  review and promotion lifecycle for changes).


Proposed Change
===============

**Overall Concepts**

* These API calls will be added under the ``/v2/`` path, however the v1 API
  will not be maintained (the model is being changed to not contact Heat and
  the existing database is being removed [3]_).
* All calls have the potential to raise a 500 if something goes horribly wrong
  in the server, but for brevity this is omitted from the list of possible
  response codes in each call.
* All calls have the potential to raise a 401 in the event of a failed user
  authentication and have been similarly omitted from each call's
  documentation.

----

.. _retrieve-single-plan:

**Retrieve a Single Plan**

URL: ``/plans/<plan-uuid>/``

Method: ``GET``

Description: Returns the details of a specific plan, including its
list of assigned roles and configuration information.

Notes:

* The configuration values are read from Tuskar's stored files rather than
  Heat itself. Heat is the source for the live stack, while Tuskar is the
  source for the plan.

Request Data: None

Response Codes:

* 200 - if the plan is found
* 404 - if there is no plan with the given UUID

Response Data:

JSON document containing the following:

* Tuskar UUID for the given plan.
* Name of the plan that was created.
* Description of the plan that was created.
* The timestamp of the last time a change was made.
* List of the roles (identified by name and version) assigned to the plan.
  For this sprint, there will be no pre-fetching of any more role information
  beyond name and version, but can be added in the future while maintaining
  backward compatibility.
* List of parameters that can be configured for the plan, including the
  parameter name, label, description, hidden flag, and current value if
  set.

Response Example:

.. code-block:: json

 {
   "id" : "dd4ef003-c855-40ba-b5a6-3fe4176a069e",
   "name" : "dev-cloud",
   "description" : "Development testing cloud",
   "last_modified" : "2014-05-28T21:11:09Z",
   "roles" : [
     {
       "id" : "55713e6a-79f5-42e1-aa32-f871b3a0cb64",
       "name" : "compute",
       "version" : "1",
       "links" : {
         "href" : "http://server/v2/roles/55713e6a-79f5-42e1-aa32-f871b3a0cb64/",
         "rel" : "bookmark"
       }
     },
     {
       "id" : "2ca53130-b9a4-4fa5-86b8-0177e8507803",
       "name" : "controller",
       "version" : "1",
       "links" : {
         "href" : "http://server/v2/roles/2ca53130-b9a4-4fa5-86b8-0177e8507803/",
         "rel" : "bookmark"
       }
     }
   ],
   "parameters" : [
     {"name" : "database_host",
      "label" : "Database Host",
      "description" : "Hostname of the database server",
      "hidden" : "false",
      "value" : "10.11.12.13"
     }
   ],
   "links" : [
     {
        "href" : "http://server/v2/clouds/dd4ef003-c855-40ba-b5a6-3fe4176a069e/",
        "rel" : "self"
     }
   ]
 }

----

.. _retrieve-plan-template:

**Retrieve a Plan's Template Files**

URL: ``/plans/<plan-uuid>/templates/``

Method: ``GET``

Description: Returns the set of files to send to Heat to create or update
the planned application.

Notes:

* The Tuskar service will build up the entire environment into a single
  file suitable for sending to Heat. The contents of this file are returned
  from this call.

Request Data: None

Response Codes:

* 200 - if the plan's templates are found
* 404 - if no plan exists with the given ID

Response Data: <Heat template>

----

.. _list-plans:

**List Plans**

URL: ``/plans/``

Method: ``GET``

Description: Returns a list of all plans stored in Tuskar. In the future when
multi-tenancy is added, this will be scoped to a particular tenant.

Notes:

* The detailed information about a plan, including its roles and configuration
  values, are not returned in this call. A follow up call is needed on the
  specific plan. It may be necessary in the future to add a flag to pre-fetch
  this information during this call.

Request Data: None (future enhancement will require the tenant ID and
potentially support a pre-fetch flag for more detailed data)

Response Codes:

* 200 - if the list can be retrieved, even if the list is empty

Response Data:

JSON document containing a list of limited information about each plan.
An empty list is returned when no plans are present.

Response Example:

.. code-block:: json

  [
     {
       "id" : "3e61b4b2-259b-4b91-8344-49d7d6d292b6",
       "name" : "dev-cloud",
       "description" : "Development testing cloud",
       "links" : {
         "href" : "http://server/v2/clouds/3e61b4b2-259b-4b91-8344-49d7d6d292b6/",
         "rel" : "bookmark"
       }
     },
     {
       "id" : "135c7391-6c64-4f66-8fba-aa634a86a941",
       "name" : "qe-cloud",
       "description" : "QE testing cloud",
       "links" : {
         "href" : "http://server/v2/clouds/135c7391-6c64-4f66-8fba-aa634a86a941/",
         "rel" : "bookmark"
       }
     }
   ]


----

.. _create-new-plan:

**Create a New Plan**

URL: ``/plans/``

Method: ``POST``

Description: Creates an entry in Tuskar's storage for the plan. The details
are outside of the scope of this spec, but the idea is that all of the
necessary Heat environment infrastructure files and directories will be
created and stored in Tuskar's storage solution [3]_.

Notes:

* Unlike in Icehouse, Tuskar will not make any calls into Heat during this
  call. This call is to create a new (empty) plan in Tuskar that
  can be manipulated, configured, saved, and retrieved in a format suitable
  for sending to Heat.
* This is a synchronous call that completes when Tuskar has created the
  necessary files for the newly created plan.
* As of this time, this call does not support a larger batch operation that
  will add roles or set configuration values in a single call. From a REST
  perspective, this is acceptable, but from a usability standpoint we may want
  to add this support in the future.

Request Data:

JSON document containing the following:

* Name - Name of the plan being created. Must be unique across all plans
  in the same tenant.
* Description - Description of the plan to create.

Request Example:

.. code-block:: json

 {
   "name" : "dev-cloud",
   "description" : "Development testing cloud"
 }

Response Codes:

* 201 - if the create is successful
* 409 - if there is an existing plan with the given name (for a particular
  tenant when multi-tenancy is taken into account)

Response Data:

JSON document describing the created plan.
The details are the same as for the GET operation on an individual plan
(see :ref:`Retrieve a Single Plan <retrieve-single-plan>`).


----

.. _delete-plan:

**Delete an Existing Plan**

URL: ``/plans/<plan-uuid>/``

Method: ``DELETE``

Description: Deletes the plan's Heat templates and configuration values from
Tuskar's storage.

Request Data: None

Response Codes:

* 200 - if deleting the plan entries from Tuskar's storage was successful
* 404 - if there is no plan with the given UUID

Response Data: None


----

.. _add-plan-role:

**Adding a Role to a Plan**

URL: ``/plans/<plan-uuid>/roles/``

Method: ``POST``

Description: Adds the specified role to the given plan.

Notes:

* This will cause the parameter consolidation to occur and entries added to
  the plan's configuration parameters for the new role.
* This call will update the ``last_modified`` timestamp to indicate a change
  has been made that will require an update to Heat to be made live.

Request Data:

JSON document containing the name and version of the role to add.

Request Example:

.. code-block:: json

 {
   "id" : "651c26f6-63e2-4e76-9b60-614b51249677"
 }

Response Codes:

* 201 - if the addition is successful
* 404 - if there is no plan with the given UUID
* 409 - if the plan already has the specified role

Response Data:

The same document describing the plan as from
:ref:`Retrieve a Single Plan <retrieve-single-plan>`. The newly added
configuration parameters will be present in the result.


----

.. _remove-cloud-plan:

**Removing a Role from a Plan**

URL: ``/plans/<plan-uuid>/roles/<role-name>/<role-version>/``

Method: ``DELETE``

Description: Removes a role from the given plan.

Notes:

* This will cause the parameter consolidation to occur and entries to be
  removed from the plan's configuration parameters.
* This call will update the ``last_modified`` timestamp to indicate a change
  has been made that will require an update to Heat to be made live.

Request Data: None

Response Codes:

* 200 - if the removal is successful
* 404 - if there is no plan with the given UUID or it does not have the
  specified role and version combination

Response Data:

The same document describing the cloud as from
:ref:`Retrieve a Single Plan <retrieve-single-plan>`. The configuration
parameters will be updated to reflect the removed role.


----

.. _changing-plan-configuration:

**Changing a Plan's Configuration Values**

URL: ``/plans/<plan-uuid>/``

Method: ``PATCH``

Description: Sets the values for one or more configuration parameters.

Notes:

* This call will update the ``last_modified`` timestamp to indicate a change
  has been made that will require an update to Heat to be made live.

Request Data: JSON document containing the parameter keys and values to set
for the plan.

Request Example:

.. code-block:: json

 {
   "parameters" : {
     "database_host" : "10.11.12.13",
     "database_password" : "secret"
   }
 }

Response Codes:

* 200 - if the update was successful
* 400 - if one or more of the new values fails validation
* 404 - if there is no plan with the given UUID

Response Data:

The same document describing the plan as from
:ref:`Retrieve a Single Plan <retrieve-single-plan>`.


----

.. _list-roles:

**Retrieving Possible Roles**

URL: ``/roles/``

Method: ``GET``

Description: Returns a list of all roles available in Tuskar.

Notes:

* There will be a separate entry for each version of a particular role.

Request Data: None

Response Codes:

* 200 - containing the available roles

Response Data: A list of roles, where each role contains:

* Name
* Version
* Description

Response Example:

.. code-block:: json

 [
   {
     "id" : "3d46e510-6a63-4ed1-abd0-9306a451f8b4",
     "name" : "compute",
     "version" : "1",
     "description" : "Nova Compute"
   },
   {
     "id" : "71d6c754-c89c-4293-9d7b-c4dcc57229f0",
     "name" : "compute",
     "version" : "2",
     "description" : "Nova Compute"
   },
   {
     "id" : "651c26f6-63e2-4e76-9b60-614b51249677",
     "name" : "controller",
     "version" : "1",
     "description" : "Controller Services"
   }
 ]


Alternatives
------------

There are currently no alternate schemas proposed for the REST APIs.

Security Impact
---------------

These changes should have no additional security impact.

Other End User Impact
---------------------

None

Performance Impact
------------------

The potential performance issues revolve around Tuskar's solution for storing
the cloud files [3]_.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

After being merged, there will be a period where the Tuskar CLI is out of date
with the new calls. The Tuskar UI will also need to be updated for the changes
in flow.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jdob

Work Items
----------

* Implement plan CRUD APIs
* Implement role retrieval API
* Write REST API documentation


Dependencies
============

These API changes are dependent on the rest of the Tuskar backend being
implemented, including the changes to storage and the template consolidation.

Additionally, the assembly of roles (provider resources) into a Heat
environment is contingent on the conversion of the TripleO Heat templates [4]_.


Testing
=======

Tempest testing should be added as part of the API creation.


Documentation Impact
====================

The REST API documentation will need to be updated accordingly.


References
==========

.. [1] https://wiki.openstack.org/wiki/TripleO/TuskarJunoPlanning/TemplateBackend
.. [2] https://etherpad.openstack.org/p/juno-summit-tripleo-tuskar-planning
.. [3] https://review.openstack.org/#/c/97553/
.. [4] https://review.openstack.org/#/c/97939/
