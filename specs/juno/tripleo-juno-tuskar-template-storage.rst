..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
TripleO Template and Deployment Plan Storage
============================================

This design specification describes a storage solution for a deployment plan.
Deployment plans consist of a set of roles, which in turn define a master Heat
template that can be used by Heat to create a stack representing the deployment
plan; and an environment file that defines the parameters needed by the master
template.

This specification is principally intended to be used by Tuskar.

https://blueprints.launchpad.net/tuskar/+spec/tripleo-juno-tuskar-template-storage

.. _tripleo_juno_tuskar_template_storage_problem:

Problem Description
===================

.. note:: The terminology used in this specification is defined in the `Tuskar
          REST API`_ specification.

.. _Tuskar REST API: https://blueprints.launchpad.net/tuskar/+spec/tripleo-juno-tuskar-plan-rest-api

In order to accomplish the goal of this specification, we need to first define
storage domain models for roles, deployment plans, and associated concepts.
These associated concepts include Heat templates and environment files.  The
models must account for requirements such as versioning and the appropriate
relationships between objects.

We also need to create a storage mechanism for these models.  The storage
mechanism should be distinct from the domain model, allowing the latter to be
stable while the former retains enough flexibility to use a variety of backends
as need and availability dictates.  Storage requirements for particular models
include items such as versioning and secure storage.


Proposed Change
===============

**Change Summary**

The following proposed change is split into three sections:

- Storage Domain Models: Defines the domain models for templates, environment
  files, roles, and deployment plans.
- Storage API Interface: Defines Python APIs that relate the models to
  the underlying storage drivers; is responsible for translating stored content
  into a model object and vice versa.  Each model requires its own storage
  interface.
- Storage Drivers: Defines the API that storage backends need to implement in
  order to be usable by the Python API Interface.  Plans for initial and future
  driver support are discussed here.

It should be noted that each storage interface will be specified by the user as
part of the Tuskar setup.  Thus, the domain model can assume that the appropriate
storage interfaces - a template store, an environment store, etc - are defined
globally and accessible for use.


**Storage Domain Models**

The storage API requires the following domain models:

- Template
- Environment File
- Role
- Deployment Plan

The first two map directly to Heat concepts; the latter two are Tuskar concepts.

Note that each model will also contain a save method. The save method will call
create on the store if the uuid isn't set, and will call update on the store
if the instance has a uuid.


**Template Model**

The template model represents a Heat template.

.. code-block:: python

    class Template:
        uuid = UUID string
        name = string
        version = integer
        description = string
        content = string
        created_at = datetime

        # This is derived from the content from within the template store.
        parameters = dict of parameter names with their types and defaults


**Environment File Model**

The environment file defines the parameters and resource registry for a Heat
stack.

.. code-block:: python

    class EnvironmentFile:
        uuid = UUID string
        content = string
        created_at = datetime
        updated_at = datetime

        # These are derived from the content from within the environment file store.
        resource_registry = list of provider resource template names
        parameters = dict of parameter names and their values

        def add_provider_resource(self, template):
            # Adds the specified template object to the environment file as a
            # provider resource.  This updates the parameters and resource registry
            # in the content.  The provider resource type will be derived from the
            # template file name.

        def remove_provider_resource(self, template):
            # Removes the provider resource that matches the template from the
            # environment file.  This updates the parameters and resource registry
            # in the content.

        def set_parameters(self, params_dict):
            # The key/value pairs in params_dict correspond to parameter names/
            # desired values.  This method updates the parameters section in the
            # content to the values specified in params_dict.


**Role Model**

A role is a scalable unit of a cloud.  A deployment plan specifies one or more
roles.  Each role must specify a primary role template.  It must also specify
the dependencies of that template.

.. code-block:: python

    class Role:
        uuid = UUID string
        name = string
        version = integer
        description = string
        role_template_uuid = Template UUID string
        dependent_template_uuids = list of Template UUID strings
        created_at = datetime

        def retrieve_role_template(self):
            # Retrieves the Template with uuid matching role_template_uuid

        def retrieve_dependent_templates(self):
            # Retrieves the list of Templates with uuids matching
            # dependent_template_uuids


**Deployment Plan Model**

The deployment plan defines the application to be deployed.  It does so by
specifying a list of roles.  Those roles are used to construct an environment
file that contains the parameters that are needed by the roles' templates and
the resource registry that register each role's primary template as a provider
resource.  A master template is also constructed so that the plan can be
deployed as a single Heat stack.

.. code-block:: python

    class DeploymentPlan:
        uuid = UUID string
        name = string
        description = string
        role_uuids = list of Role UUID strings
        master_template_uuid = Template UUID string
        environment_file_uuid = EnvironmentFile UUID string
        created_at = datetime
        updated_at = datetime

        def retrieve_roles(self):
            # Retrieves the list of Roles with uuids matching role_uuids

        def retrieve_master_template(self):
            # Retrieves the Template with uuid matching master_template_uuid

        def retrieve_environment_file(self):
            # Retrieves the EnvironmentFile with uuid matching environment_file_uuid

        def add_role(self, role):
            # Adds a Role to the plan.  This operation will modify the master
            # template and environment file through template munging operations
            # specified in a separate spec.

        def remove_role(self, role):
            # Removes a Role from the plan.  This operation will modify the master
            # template and environment file through template munging operations
            # specified in a separate spec.

        def get_dependent_templates(self):
            # Returns a list of dependent templates.  This consists of the
            # associated role templates.


**Storage API Interface**

Each of the models defined above has their own Python storage interface. These
are manager classes that query and perform CRUD operations against the storage
drivers and return instances of the models for use (with the exception of delete
which returns ``None``). The storage interfaces bind the models to the driver
being used; this allows us to store each model in a different location.

Note that each store also contains a serialize method and a deserialize method.
The serialize method takes the relevant object and returns a dictionary
containing all value attributes; the deserialize method does the reverse.

The drivers are discussed in
:ref:`the next section<tripleo_juno_tuskar_template_storage_drivers>`.


**Template API**

.. code-block:: python

    class TemplateStore:

        def create(self, name, content, description=None):
            # Creates a Template.  If no template exists with a matching name,
            # the template version is set to 0; otherwise it is set to the
            # greatest existing version plus one.

        def retrieve(self, uuid):
            # Retrieves the Template with the specified uuid.  Queries a Heat
            # template parser for template parameters and dependent template names.

        def retrieve_by_name(self, name, version=None):
            # Retrieves the Template with the specified name and version.  If no
            # version is specified, retrieves the latest version of the Template.

        def delete(self, uuid):
            # Deletes the Template with the specified uuid.

        def list(self, only_latest=False):
            # Returns a list of all Templates.  If only_latest is True, filters
            # the list to the latest version of each Template name.


**Environment File API**

The environment file requires secure storage to protect parameter values.

.. code-block:: python

    class EnvironmentFileStore:

        def create(self):
            # Creates an empty EnvironmentFile.

        def retrieve(self, uuid):
            # Retrieves the EnvironmentFile with the specified uuid.

        def update(self, model):
            # Updates an EnvironmentFile.

        def delete(self, uuid):
            # Deletes the EnvironmentFile with the specified uuid.

        def list(self):
            # Returns a list of all EnvironmentFiles.


**Role API**

.. code-block:: python

    class RoleStore:

        def create(self, name, role_template, description=None):
                   version=None, template_uuid=None):
            # Creates a Role.  If no role exists with a matching name, the
            # template version is set to 0; otherwise it is set to the greatest
            # existing version plus one.
            #
            # Dependent templates are derived from the role_template.  The
            # create method will take all dependent template names from
            # role_template, retrieve the latest version of each from the
            # TemplateStore, and use those as the dependent template list.
            #
            # If a dependent template is missing from the TemplateStore, then
            # an exception is raised.

        def retrieve(self, uuid):
            # Retrieves the Role with the specified uuid.

        def retrieve_by_name(self, name, version=None):
            # Retrieves the Role with the specified name and version.  If no
            # version is specified, retrieves the latest version of the Role.

        def update(self, model):
            # Updates a Role.

        def delete(self, uuid):
            # Deletes the Role with the specified uuid.

        def list(self, only_latest=False):
            # Returns a list of all Roles.  If only_latest is True, filters
            # the list to the latest version of each Role.


**Deployment Plan API**

.. code-block:: python

    class DeploymentPlanStore:

        def create(self, name, description=None):
            # Creates a DeploymentPlan.  Also creates an associated empty master
            # Template and EnvironmentFile; these will be modified as Roles are

        def retrieve(self, uuid):
            # Retrieves the DeploymentPlan with the specified uuid.

        def update(self, model):
            # Updates a DeploymentPlan.

        def delete(self, uuid):
            # Deletes the DeploymentPlan with the specified uuid.

        def list(self):
            # Retrieves a list of all DeploymentPlans.


.. _tripleo_juno_tuskar_template_storage_drivers:

**Storage Drivers**

Storage drivers operate by storing object dictionaries.  For storage solutions
such as Glance these dictionaries are stored as flat files.  For a storage
solution such as a database, the dictionary is translated into a table row.  It
is the responsibility of the driver to understand how it is storing the object
dictionaries.

Each storage driver must provide the following methods.

.. code-block:: python

    class Driver:

        def create(self, filename, object_dict):
            # Stores the specified content under filename and returns the resulting
            # uuid.

        def retrieve(self, uuid):
            # Returns the object_dict matching the uuid.

        def update(self, uuid, object_dict):
            # Updates the object_dict specified by the uuid.

        def delete(self, uuid):
            # Deletes the content specified by the uuid.

        def list(self):
            # Return a list of all content.


For Juno, we will aim to use a combination of a relational database and Heat.
Heat will be used for the secure storage of sensitive environment parameters.
Database tables will be used for everything else. The usage of Heat for secure
stores relies on `PATCH support`_ to be added the Heat API. This bug is
targeted for completion by Juno-2.

.. _PATCH support: https://bugs.launchpad.net/heat/+bug/1224828

This is merely a short-term solution, as it is understood that there is some
reluctance in introducing an unneeded database dependency.  In the long-term we
would like to replace the database with Glance once it is updated from an image
store to a more general artifact repository.  However, this feature is currently
in development and cannot be relied on for use in the Juno cycle.  The
architecture described in this specification should allow reasonable ease in
switching from one to the other.


.. _tripleo_juno_tuskar_template_storage_alternatives:

Alternatives
------------

**Modeling Relationships within Heat Templates**

The specification proposes modeling relationships such as a plan's associated
roles or a role's dependent templates as direct attributes of the object.
However, this information would appear to be available as part of a plan's
environment file or by traversing the role template's dependency graph.  Why
not simply derive the relationships in that way?

A role is a Tuskar abstraction.  Within Heat, it corresponds to a template used
as a provider resource; however, a role has added requirements, such as the
versioning of itself and its dependent templates, or the ability to list out
available roles for selection within a plan.  These are not requirements that
Heat intends to fulfill, and fulfilling them entirely within Heat feels like an
abuse of mechanics.

From a practical point of view, modeling relationships within Heat templates
requires the in-place modification of Heat templates by Tuskar to deal with
versioning.  For example, if version 1 of the compute role specifies
{{compute.yaml: 1}, {compute-config.yaml: 1}}, and version 2 of the role
specifies {{compute.yaml: 1}, {compute-config.yaml: 2}}, the only way to
allow both versions of the role to be used is to allow programmatic
modification of compute.yaml to point at the correct version of
compute-config.yaml.


**Swift as a Storage Backend**

Swift was considered as an option to replace the relational database but was
ultimately discounted for two key reasons:

- The versioning system in Swift doesn't provide a static reference to the
  current version of an object. Rather it has the version "latest" and this is
  dynamic and changes when a new version is added, therefore there is no way to
  stick a deployment to a version.
- We need to create a relationship between the provider resources within a Role
  and swift doesn't support relationships between stored objects.

Having said that, after seeking guidance from the Swift team, it has been
suggested that a naming convention or work with different containers may
provide us with enough control to mimic a versioning system that meets our
requirements. These suggestions have made Swift more favourable as an option.


**File System as a Storage Backend**

The filesystem was briefly considered and may be included to provide a simpler
developer setup. However, to create a production ready system with versioning,
and relationships this would require re-implementing much of what other
databases and services provide for us. Therefore, this option is reserved only
for a development option which will be missing key features.


**Secure Driver Alternatives**

Barbican, the OpenStack secure storage service, provides us with an alternative
if PATCH support isn't added to Heat in time.

Currently the only alternative other than Barbican is to implement our own
cryptography with one of the other options listed above. This isn't a
favourable choice as it adds a technical complexity and risk that should be
beyond the scope of this proposal.

The other option with regards to sensitive data is to not store any. This would
require the REST API caller to provide the sensitive information each time a
Heat create (and potentially update) is called.


Security Impact
---------------

Some of the configuration values, such as service passwords, will be sensitive.
For this reason, Heat or Barbican will be used to store all configuration
values.

While access will be controlled by the Tuskar API large files could be provided
in the place of provider resource files or configuration files. These should be
verified against a reasonable limit.


Other End User Impact
---------------------

The template storage will be primarily used by the Tuskar API, but as it may be
used directly in the future it will need to be documented.


Performance Impact
------------------

Storing the templates in Glance and Barbican will lead to API calls over the
local network rather than direct database access. These are likely to have
higher overhead. However, the read and writing used in Tuskar is expected to be
infrequent and will only trigger simple reads and writes when manipulating a
deployment plan.


Other Deployer Impact
---------------------

None


Developer Impact
----------------

TripleO will have access to sensitive and insensitive storage through the
storage API.


Implementation
==============


Assignee(s)
-----------

Primary assignee:
  d0ugal

Other contributors:
  tzumainn


Work Items
----------

- Implement storage API
- Create Glance and Barbican based storage driver
- Create database storage driver


Dependencies
============

- Glance
- Barbican


Testing
=======

- The API logic will be verified with a suite of unit tests that mock the
  external services.
- Tempest will be used for integration testing.


Documentation Impact
====================

The code should be documented with docstrings and comments. If it is used
outside of Tuskar further user documentation should be developed.


References
==========

- https://blueprints.launchpad.net/glance/+spec/artifact-repository-api
- https://blueprints.launchpad.net/glance/+spec/metadata-artifact-repository
- https://bugs.launchpad.net/heat/+bug/1224828
- https://docs.google.com/document/d/1tOTsIytVWtXGUaT2Ia4V5PWq4CiTfZPDn6rpRm5In7U
- https://etherpad.openstack.org/p/juno-hot-artifacts-repository-finalize-design
- https://etherpad.openstack.org/p/juno-summit-tripleo-tuskar-planning
- https://wiki.openstack.org/wiki/Barbican
- https://wiki.openstack.org/wiki/TripleO/TuskarJunoPlanning
- https://wiki.openstack.org/wiki/TripleO/TuskarJunoPlanning/TemplateBackend
