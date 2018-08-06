..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================================
Provide a common Validation Framework inside python-tripleoclient
=================================================================

https://blueprints.launchpad.net/tripleo/+spec/validation-framework

Currently, we're lacking a common validation framework in tripleoclient. This
framework should provide an easy way to validate environment prior deploy and
prior update/upgrade, on both undercloud and overcloud.

Problem Description
===================

Currently, we have two types of validations:

* Those launched prior the undercloud deploy, embedded into the deploy itself

* Those launched at will via a Mistral Workflow

There isn't any unified way to call any validations by itself in an easy way,
and we lack the capacity to easily add new validations for the undercloud
preflight checks.

This situation isn't good, and leads to issues when an operator wants to update
or upgrade its environment, because he's not really able to validate its state
in an independent way (i.e. they must run the deploy command that will trigger
the validations, at least from the CLI).

Moreover, there is a need to make the CLI and UI converge. The latter already
uses the full list of validations. Adding the full support of
tripleo-validations to the CLI will improve the overall quality, usability and
maintenance of the validations.

Creating a proper framework inside the CLI, based on the UI state, is the best
way to ensure every Openstack-related projects will have a chance for proper
validations.

Proposed Change
===============

Overview
--------

In order to correct the current situation, we propose to create a new
"branching" in the tripleoclient commands: `openstack validator`

This new subcommand will allow to list and run validations in an independent
way.

Doing so will allow to get a clear and clean view on the validations we can run
depending on the stage we're in.

The following subcommands should be supported:

* openstack validator list: will display all the available validations with
  a small description, like "validate network capabilities on undercloud"

* openstack validator run: will run the validations. Should take options, like:

    * --validation-name: run only the passed validation.
    * --undercloud: runs all undercloud-related validations
    * --overcloud: runs all overcloud-related validations
    * --use-mistral: runs validations through Mistral
    * --use-ansible: runs validations directly via Ansible

* openstack validator selfcheck: will run checks in order to ensure it has the
  proper services to run the other validations (ansible is working, playbooks
  are valid, Mistral is running and in good shape, and so on).

* in addition, common options for all the subcommands:

  * --extra-playbooks: path to a local directory containing validation playbook
    maintained by the operator
  * --output: points to a valid Ansible output_callback, such as the native
    *json*, or custom *validation_output*. The default one should be the latter
    as it renders a "human readable" output. More callbacks can be added later.

The default engine will be determined by the presence of Mistral: if Mistral is
present and accepting requests (meaning the Undercloud is most probably
deployed), the validator has to use it by default. If no Mistral is present, it
must fallback on the ansible-playbook.

(Note: the command path has yet to be defined - this is only a "mock-up".)

The validations should be in the form of Ansible playbook, in order to be
easily accessed from Mistral as well (as it is currently the case). It will
also allow to get a proper documentation, canvas and gives the possibility to
validate the playbook before running it (ensuring there are metadata, output,
and so on).

Speaking of the UI: currently, it runs validations through a Mistral workflow.

The new validation framework should be able to populate Mistral workflows on the
fly, by listing the existing validations. That way, the operators using the UI
will be able to do the same things as the CLI.

In the end, all the validation playbooks should be in one and only one
location: tripleo-validations. We also must ensure we can navigate and find
validations in an easy way in this repository. Thus we need a proper tree
schema, maybe per-service directories with service-dedicated validations, for
instance:

  * nova
    * validate-listen-port-xxx.yaml
    * validate-listen-socket-xxx.yaml
    * ....
  * memcached
    * validate-listen-port-xxx.yaml
    * ....
  * dashboard
    * validate-public-access.yaml
    * validate-listen-socket-xxx.yaml
    * ....
  * keystone
    * validate-listen-port-xxx.yaml
    * validate-dummy-auth.yaml
    * ....
  * ....

In addition, a proper documentation with examples describing the Good Practices
regarding the playbooks content, format and outputs should be created.

For instance, a playbook should contain a description, a "human readable error
output", and if applicable a possible solution.

We might want to add support for "nagios-compatible outputs" and exit codes,
but it is not sure running those validations through any monitoring tool is a
good idea due to the possible load it might create. This has to be discussed
later, once we get the framework in place.

Alternatives
------------

No real alternatives in fact. Currently, we have many ways to validate, but
they are all unrelated, not concerted. If we don't provide a unified framework,
we will get more and more "side validations ways" and it won't be maintainable.

Security Impact
---------------

Rights might be needed for some validations - they should be added accordingly
in the system sudoers, in a way that limits unwanted privilege escalations.


Other End User Impact
---------------------

The end user will get a proper way to validate the environment prior to any
action.
This will give more confidence in the final product, and ease the update and
upgrade processes.

It will also provide a good way to collect information about the systems in
case of failures.

If a "nagios-compatible output" is to be created (mix of ansible JSON output,
parsing and compatibility stuff), it might provide a way to get a daily report
about the health of the stack - this might be a nice feature, but not in the
current scope (will need a new stdout_callback for instance).

Performance Impact
------------------

The more validations we get, the more time it might take IF we decide to run
them by default prior any action.

The current way to disable them, either with a configuration file or a CLI
option will stay.

In addition, we can make a great use of "groups" in order to filter out greedy
validations.


Other Deployer Impact
---------------------

Providing a dedicated path for validation will make the deployment easier.

Providing a unified framework will allow an operator to run the validations
either from the UI, or from the CLI, without any surprise regarding the
validation list.

Developer Impact
----------------

A refactoring will be needed in python-tripleoclient in order to get a proper
subcommand and options.

A correct way to call Ansible from Python is to be decided (ansible-runner?).

The usage of Mistral is to be assessed (Do we really need it for the UI?).

A correct way to populate Mistral Workflows is to be decided if we keep Mistral.

In the end, the framework will allow other Openstack projects to push their own
validations, since they are the ones knowing how and what to validate in the
different services making Openstack.

All validations will be centralized in the tripleo-validations repository.
This means we might want to create a proper tree in order to avoid having
100+ validations in the same directory.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  cjeanner

Other contributors:
  ccamacho
  dpeacock
  florianf


Work Items
----------

* List current existing validations in both undercloud_preflight.py and
  openstack-tripleo-validations.

* Decide if we integrate ansible-runner as a dependency (needs to be packaged).

* Implement the undercloud_preflight validations as Ansible playbook.

* Implement a proper way to call Ansible from the tripleoclient code.

* Implement support for a configuration file dedicated for the validations.

* Implement the new subcommand tree in tripleoclient.

* Validate, Validate, Validate.


Dependencies
============

* Ansible-runner: https://github.com/ansible/ansible-runner

* Openstack-tripleo-validations: https://github.com/openstack/tripleo-validations



Testing
=======

The CI can't possibly provide the "right" environment with all the requirements.
The code has to implement a way to configure the validations so that the CI
can override the *productive* values we will set in the validations.


Documentation Impact
====================

A new entry in the documentation must be created in order to describe this new
framework (for the devs) and new subcommand (for the operators).

References
==========

* http://lists.openstack.org/pipermail/openstack-dev/2018-July/132263.html

* https://bugzilla.redhat.com/show_bug.cgi?id=1599829

* https://bugzilla.redhat.com/show_bug.cgi?id=1601739
