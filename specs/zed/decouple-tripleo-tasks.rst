..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Decouple TripleO Tasks
======================

https://blueprints.launchpad.net/tripleo/+spec/decouple-tripleo-tasks

This spec proposes decoupling tasks across TripleO by organizing tasks in a way
that they are grouped as a function of what they manage. The desire is to be
able to better isolate and minimize what tasks need to be run for specific
management operations. The process of decoupling tasks is implemented through
moving tasks into standalone native ansible roles and playbooks in tripleo-ansible.


Problem Description
===================

TripleO presently manages the entire software configuration of the overcloud at
once each time ``openstack overcloud deploy`` is executed. Regardless of
whether nodes were already deployed, require a full redeploy for some reason,
or are new nodes (scale up, replacement) all tasks are executed. The
functionality of only executing needed tasks lies within Ansible.

The problem with relying entirely on Ansible to determine if any changes are
needed is that it results in long deploy times. Even if nothing needs to be
done, it can take hours just to have Ansible check each task in order to make
that determination.

Additionally, TripleO's reliance on external tooling (Puppet, container config
scripts, bootstrap scripts, etc) means that tasks executing those tools
**must** be executed by Ansible as Ansible does not have the necessary data
needed in order to determine if those tasks need to be executed or not. These
tasks often have cascading effects in determining what other tasks need to be
run. This is a general problem across TripleO, and is why the model of just
executing all tasks on each deploy has been the accepted pattern.


Proposed Change
===============

The spec proposes decoupling tasks and separating them out as needed to manage
different functionality within TripleO. Depending on the desired management
operation, tripleoclient will contain the necessary functionality to trigger
the right tasks. Decoupling and refactoring tasks will be done by migrating to
standalone ansible role and playbooks within tripleo-ansible. This will allow
for reusing the standalone ansible artifacts from tripleo-ansible to be used
natively with just ``ansible-playbook``. At the same time, the
``tripleo-heat-templates`` interfaces are maintained by consuming the new roles
and playbooks from ``tripleo-ansible``.

Overview
--------

There are 3 main changes proposed to implement this spec:

#. Refactor ansible tasks from ``tripleo-heat-templates`` into standalone roles
   in tripleo-ansible.
#. Develop standalone playbooks within tripleo-ansible to consume the
   tripleo-ansible roles.
#. Update tripleo-heat-templates to use the standalone roles and playbooks from
   ``tripleo-ansible`` with new ``role_data`` interfaces to drive specific
   functionality with new ``openstack overcloud`` commands.

Writing standalone roles in ``tripleo-ansible`` will largely be an exercise of
copy/paste from tasks lists in ``tripleo-heat-templates``. As tasks are moved
into standalone roles, tripleo-heat-templates can be directly updated to run
tasks from the those roles using ``include_role``. This pattern is already well
established in tripleo-heat-templates with composable services that use
existing standalone roles.

New playbooks will be developed within tripleo-ansible to drive the standalone
roles using pure ``ansible-playbook``. These playbooks will offer a native
ansible experience for deploying with tripleo-ansible.

The design principles behind the standalone role and playbooks are:

#. Native execution with ansible-playbook, an inventory, and variable files.
#. No Heat. While Heat remains part of the TripleO architecture, it has no
   bearing on how the native ansible is developed in tripleo-ansible.
   tripleo-heat-templates can consume the standalone ansible playbooks and
   roles from tripleo-ansible, but it does not dictate the interface. The
   interface should be defined for native ansible best practices.
#. No puppet. As the standalone roles are developed, they will not rely on
   puppet for configuration or any other tasks. To allow integration with
   tripleo-heat-templates and existing TripleO interfaces (Hiera, Heat
   parameters), the roles will allow skipping config generation and other parts
   that use puppet so that pieces can be overridden by
   ``tripleo-heat-templates`` specific tasks. When using native Ansible,
   templated config files and native ansible tasks will be used instead of
   Puppet.
#. While the decoupled tasks will allow for cleaner interfaces for executing
   just specific management operations, all tasks will remain idempotent. A
   full deployment that re-runs all tasks will still work, and result in no
   effective changes for an already deployed cloud with the same set of inputs.

The standalone roles will use separated task files for each decoupled
management interface exposed. The playbooks will be separated by management
interface as well to allow for executing just specific management functionality.

The decoupled management interfaces are defined as:

* bootstrap
* install
* pre-network
* network
* configure
* container-config
* service-bootstrap

New task interfaces in ``tripleo-heat-templates`` will be added under
``role_data`` to correspond with the new management interfaces, and consume the
standalone ansible from tripleo-ansible. This will allow executing just
specific management interfaces and using the standalone playbooks from
tripleo-ansible directly.

New subcommands will be added to tripleoclient to trigger the new management
interface operations, ``openstack overcloud install``, ``openstack overcloud
configure``, etc.

``openstack overcloud deploy`` would continue to function as it presently does
by doing a full assert of the system state with all tasks. The underlying
playbook, ``deploy-steps-playbook.yaml`` would be updated as necessary to
include the other playbooks so that all tasks can be executed.

Alternatives
------------

:Alternative 1 - Use --tags/--skip-tags:

With ``--tags`` / ``--skip-tags``, tasks could be selectively executed. In the
past this has posed other problems within TripleO. Using tags does not allow
for composing tasks to the level needed, and often results in running tasks
when not needed or forgetting to tag needed tasks. Having to add the special
cased ``always`` tag becomes necessary so that certain tasks are run when
needed. The tags become difficult to maintain as it is not apparent what tasks
are tagged when looking at the entire execution. Additionally, not all
operations within TripleO map to Ansible tasks one to one. Container startup
are declared in a custom YAML format, and that format is then used as input to
a task. It is not possible to tag individual container startups unless tag
handling logic was added to the custom modules used for container startup.

:Alternative 2 - Use --start-at-task:

Using ``--start-at-task`` is likewise problematic, and it does not truly
partition the full set of tasks. Tasks would need to be reordered anyway across
much of TripleO so that ``--start-at-task`` would work. It would be more
straightforward to separate by playbook if a significant number of tasks need
to be reordered.

Security Impact
---------------

Special consideration should be given to security related tasks to ensure that
the critical tasks are executed when needed.

Upgrade Impact
--------------

Upgrade and update tasks are already separated out into their own playbooks.
There is an understanding that the full ``deploy_steps_playbook.yaml`` is
executed after an update or upgrade however. This full set of tasks could end
up being reduced if tasks are sufficiently decoupled in order to run the
necessary pieces in isolation (config, bootstrap, etc).

Other End User Impact
---------------------

Users will need to be aware of the limitations of using the new management
commands and playbooks. The expectation within TripleO has always been the
entire state of the system is re-asserted on scale up and configure operations.

While the ability to still do a full assert would be present, it would no
longer be required. Operators and users will need to understand that only
running certain management operations may not fully apply a desired change. If
only a reconfiguration is done, it may not imply restarting containers. With
the move to standalone and native ansible components, with less
``config-download`` based generation, it should be more obvious what each
playbooks is responsible for managing. The native ansible interfaces will help
operators reason about what needs to be run and when.

Performance Impact
------------------

Performance should be improved for the affected management operations due to
having to run less tasks, and being able to run only the tasks needed for a
given operation.

There should be no impact when running all tasks. Tasks must be refactored in
such a way that the overall deploy process when all tasks are run is not made
slower.

Other Deployer Impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example a flag that other hypervisor drivers might want to
  implement as well)? Are the default values ones which will work well in
  real deployments?

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

Developer Impact
----------------

TripleO developers will be responsible for updating the service templates that
they maintain in order to refactor the tasks.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  James Slagle <jslagle@redhat.com>

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

None.

Testing
=======

Existing CI jobs would cover changes to task refactorings.
New CI jobs could be added for the new isolated management operations.

Documentation Impact
====================

New commands and playbooks must be documented.


References
==========
`standalone-roles POC <https://review.opendev.org/q/topic:standalone-roles>`_
