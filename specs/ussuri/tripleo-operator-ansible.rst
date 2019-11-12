..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================================
tripleo-operator-ansible - Ansible roles and modules to interact with TripleO
=============================================================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-operator-ansible

As an operator of a TripleO deployment, I would like to be able to comsume
supported ansible roles and modules that let me perform TripleO related
actions in my automation.

Problem Description
===================

The existing tripleo-ansible_ repository currently contains roles, plugins
and modules that are consumed by TripleO to perform the actual deployments and
configurations. As these are internal implementations to TripleO, we would not
want operators consuming these directly. The tripleo-ansible_ repository is
also branched which means that the contents within the repo and packaging
are specific to a singular release. This spec propose that we create a new
repository targeted for external automation for any supported version.

Currently Operators do not have a set of official ansible roles and modules
that can be used to deploy and manage TripleO environments. For folks who wish
to manage their TripleO environments in an automated fashion, we have seen
multiple folks implement the same roles to manage TripleO. e.g.
tripleo-quickstart_, tripleo-quickstart-extras_, infrared_, tripleo-lab_.

* TripleO should provide a set of ansible roles and modules that can be used
  by the end user to deploy and manage an Undercloud and Overcloud.

* TripleO should provide a set of ansible roles and modules that can be used
  to perform scaling actions.

* TripleO should provide a set of ansible roles and modules that can be used
  to perform update and upgrade actions.

.. _tripleo-ansible: https://opendev.org/openstack/tripleo-ansible
.. _infrared: https://github.com/redhat-openstack/infrared
.. _tripleo-quickstart: https://opendev.org/openstack/tripleo-quickstart
.. _tripleo-quickstart-extras: https://opendev.org/openstack/tripleo-quickstart-extras
.. _tripleo-lab: https://github.com/cjeanner/tripleo-lab

Proposed Change
===============

Overview
--------

TripleO should create a new repository where ansible roles, plugins and
modules that wrap TripleO actions can be stored. This repository should be
branchless so that the roles can be used with any currently supported version
of TripleO. The goal is to only provide automation for TripleO actions and not
necessarily other cloud related actions. The roles in this new repository
should only be targeted to providing an automation interface for the existing
`tripleoclient commands`_. The repository may provide basic setups actions such
as implementing a wrapper around tripleo-repos_. The roles contained in this
repository should not implement additional day 2 cloud related operations such
as creating servers, networks or other resources on the deployed Overcloud.

This new repository should be able to be packaged and distributed via an RPM
as well as being able to be published to `Ansible Galaxy`_. The structure
of this new repository should be Ansible collections_ compatible.

The target audience of the new repository would be end users (operators,
developers, etc) who want to write automation around TripleO. The new
repository and roles would be our officially supported automation artifacts.
One way to describe this would be like providing Puppet modules for a given
peice of software so that it can be consumed by users who use Puppet.  The
existing CLI will continue to function for users who do not want to use
Ansible to automate TripleO deployments or who wish to continue to use the CLI
by hand.  The roles are not a replacement for the CLI, but only provide an
official set of roles for people who use Ansible.

The integration point for Ansible users would be the roles provided via
tripleo-operator-ansible.  We would expect users to perform actions by
including our provided roles.

An example playbook for a user could be:

.. code-block:: yaml

    - hosts: undercloud
      gather_facts: true
      tasks:
        - include_role:
            role: tripleo_undercloud
            tasks_from: install
            vars:
              tripleo_undercloud_configuration:
                 DEFAULT:
                     undercloud_debug: True
                     local_ip: 192.168.50.1/24
        - name: Copy nodes.json
          copy:
            src: /home/myuser/my-environment-nodes.json
            dest: /home/stack/nodes.json
        - include_role:
            role: tripleo_baremetal
            tasks_from: introspection
            vars:
              tripleo_baremetal_nodes_file: /home/stack/nodes.json
              tripleo_baremetal_introspection_provide: True
              tripleo_baremetal_introspection_all_managable: True
        - include_role:
            role: tripleo_overcloud
            tasks_from: deploy
            vars:
              tripleo_overcloud_environment_files:
                - network_isolation.yaml
                - ceph_storage.yaml
              tripleo_overcloud_roles:
                - Controller
                - Networker
                - Compute
                - CephStorage

The internals of these roles could possibly proceed in two different paths:

* Implement simple wrappers around the invocation of the actual TripleO
  commands using execs, shell or commands. This path will likely be the fastest
  path to have an initial implementation.

.. code-block:: yaml

    - name: Install undercloud
      command: "openstack undercloud install {{ tripleo_undercloud_install_options }}"
      chdir: "{{ tripleo_undercloud_install_directory }}"


* Implement a python wrapper to call into the provided tripleoclient classes.
  This path may be a longer term goal as we may be able to provide better
  testing by using modules.

.. code-block:: python

    #!/usr/bin/python

    # import the python-tripleoclient
    # undercloud cli

    from tripleoclient.v1 import undercloud

    import sys
    import json
    import os
    import shlex

    # See the following for details
    # https://opendev.org/openstack/python-tripleoclient/src/branch/
    # master/tripleoclient/v1/undercloud.py

    # setup the osc command


    class Arg:
        verbose_level = 4


    # instantiate the
    u = undercloud.InstallUndercloud('tripleo', Arg())

    # prog_name = 'openstack undercloud install'
    tripleo_args = u.get_parser('openstack undercloud install')

    # read the argument string from the arguments file
    args_file = sys.argv[1]
    args_data = file(args_file).read()

    # For this module, we're going to do key=value style arguments.
    arguments = shlex.split(args_data)
    for arg in arguments:

        # ignore any arguments without an equals in it
        if "=" in arg:

            (key, value) = arg.split("=")

            # if setting the time, the key 'time'
            # will contain the value we want to set the time to

            if key == "dry_run":
                if value == "True":
                    tripleo_args.dry_run = True
                else:
                    tripleo_args.dry_run = False

            tripleo_args.force_stack_validations = False
            tripleo_args.no_validations = True
            tripleo_args.force_stack_update = False
            tripleo_args.inflight = False

            # execute the install via python-tripleoclient
            rc = u.take_action(tripleo_args)

            if rc != 0:
                print(json.dumps({
                    "failed": True,
                    "msg": "failed tripleo undercloud install"
                }))
                sys.exit(1)

            print(json.dumps({
                "changed": True,
                "msg": "SUCCESS"
            }))
            sys.exit(0)

.. code-block:: yaml

    - name: Install undercloud
      tripleo_undercloud:
        install: true
        foo: bar

These implementations will need to be evaluated to understand which works
best when attempting to support multiple versions of TripleO where options
may or may not be available. The example of this is where we supported one
cli parameter in versions >= Stein but not prior to this.

The goal is to have a complete set of roles to do basic deployments within
a single cycle. We should be able to itterate on the internals of the roles
once we have established basic set to prove out the concept. More complex
actions or other version support may follow on in later cycles.

.. _tripleoclient commands: https://docs.openstack.org/python-tripleoclient/latest/index.html
.. _tripleo-repos: https://opendev.org/openstack/tripleo-repos
.. _Ansible Galaxy: https://galaxy.ansible.com/
.. _collections: https://docs.ansible.com/ansible/latest/dev_guide/developing_collections.html

Alternatives
------------

* Do nothing and continue to have multiple tools re-implement the actions in
  ansible roles.

* Pick a singular implementaion from the existing set and merge them together
  within this existing tool. This however may include additional actions that
  are outside of the scope of the TripleO management.  This may also limit the
  integration by others if established interfaces are too opinionated.

Security Impact
---------------

None.

Upgrade Impact
--------------

There should be no upgrade impact other than pulling in the upgrade related
actions into this repository.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

Developers will need to ensure the supported roles are updated if the cli
or other actions are updated with new options or patterns.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mwhahaha

Other contributors:
  weshay
  emilienm
  cloudnull

Work Items
----------

The existing roles should be evaulated to see if they can be reused and pulled
into the new repository.

* Create new tripleo-operator-ansible
* Establish CI and testing framework for the new repository
* Evaulate and pull in existing roles if possible
* Initial implementation may only be a basic wrapper over the cli
* Update tripleo-quickstart to leverage the newly provided roles and remove
  previously roles.

Dependencies
============

If there are OpenStack service related actions that need to occur, we may need
to investigate the inclusion of OpenStackSDK, shade or other upstream related
tools.

Testing
=======

The new repository should have molecule testing for any new role created.
Additionally once tripleo-quickstart begins to consume the roles we will need
to ensure that other deployment related CI jobs are included in the testing
matrix.

Documentation Impact
====================

The roles should be documented (perferrably automated) for the operators to
be able to consume these new roles.

References
==========

None.
