..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================================
Unifying TripleO Orchestration with Task-Core and Directord
===========================================================

Include the URL of your launchpad blueprint:
https://blueprints.launchpad.net/tripleo/+spec/unified-orchestration

The purpose of this spec is to introduce core concepts around Task-Core and
Directord, explain their benefits, and cover why the project should migrate
from using Ansible to using Directord and Task-Core.

TripleO has long been established as an enterprise deployment solution for
OpenStack. Different task executions have been used at different times.
Originally, os-collect-config was used, then the switch to Ansible was
completed. A new task execution environment will enable moving forward
with a solution designed around the specific needs of TripleO.

The tools being introduced are Task-Core and Directord.

Task-Core_:
  A dependency management and inventory graph solution which allows operators
  to define tasks in simple terms with robust dominion over a given
  environment. Declarative dependencies will ensure that if a container/config
  is changed, only the necessary services are reloaded/restarted. Task-Core
  provides access to the right tools for a given job with provenance, allowing
  operators and developers to define outcomes confidently.

Directord_:
  A deployment framework built to manage the data center life cycle, which is
  both modular and fast. Directord focuses on consistently maintaining
  deployment expectations with a near real-time level of performance_ at almost
  any scale.


Problem Description
===================

Task execution in TripleO is:

* Slow
* Resource intensive
* Complex
* Defined in a static and sequential order
* Not optimized for scale

TripleO presently uses Ansible to achieve its task execution orchestration
goals. While the TripleO tooling around Ansible (playbooks, roles, modules,
plugins) has worked and is likely to continue working should maintainers bear
an increased burden, future changes around direction due to `Ansible Execution
Environments`_ provide an inflection point. These upstream changes within
Ansible, where it is fundamentally moving away from the TripleO use case, force
TripleO maintainers to take on more ownership for no additional benefit. The
TripleO use case is actively working against the future direction of Ansible.

Further, the Ansible lifecycle has never matched that of TripleO. A single
consistent and backwards compatible Ansible version can not be used across a
single version of TripleO without the tripleo-core team committing to maintain
that version of Ansible, or commit to updating the Ansible version in a stable
TripleO release. The cost to maintain a tool such as Ansible that the core team
does not own is high vs switching to custom tools designed specifically for the
TripleO use case.

The additional cost of maintaining Ansible as the task execution engine for
TripleO, has a high likelihood of causing a significant disruption to the
TripleO project; this is especially true as the project looks to support future
OS versions.

Presently, there are diminishing benefits that can be realized from any
meaningful performance, scale, or configurability improvments. The
simplification efforts and work around custom Ansible strategies and plugins
have reached a conclusion in terms of returns.

While other framework changes to expose scaling mechanisms, such as using
``--limit`` or partitioning of the ansible execution across multiple stacks or
roles do help with the scaling problem, they are however in the category of
work arounds as they do not directly address the inherent scaling issues with
task executions.

Proposed Change
===============

To make meaningful task execution orchestration improvements, TripleO must
simplify the framework with new tools, enable developers to build intelligent
tasks, and provide meaningful performance enhancements that scale to meet
operators' expectations. If TripleO can capitalize on this moment, it will
improve the quality of life for day one deployers and day two operations and
upgrades.

The proposal is to replace all usage of Ansible with Directord for task
execution, and add the usage of Task-Core for dynamic task dependencies.

In some ways, the move toward Task-Core and Directord creates a
General-Problem_, as it's proposing the replacement of many bespoke tools, which
are well known, with two new homegrown ones. Be that as it may, much attention
has been given to the user experience, addressing many well-known pain points
commonly associated with TripleO environments, including: scale, barrier to
entry, execution times, and the complex step process.

Overview
--------

This specification consists of two parts that work together to achieve the
project goals.

Task-Core:
  Task-Core builds upon native OpenStack libraries to create a dependency graph
  and executes a compiled solution. With Task-Core, TripleO will be able to
  define a deployment with dependencies instead of brute-forcing one. While
  powerful, Task-Core keeps development easy and consistent, reducing the time
  to deliver and allowing developers to focus on their actual deliverable, not
  the orchestration details. Task-Core also guarantees reproducible builds,
  runtime awareness, and the ability to resume when issues are encountered.

* Templates containing step-logic and ad-hoc tasks will be refactored into
  Task-Core definitions.

* Each component can have its own Task-Core purpose, providing resources and
  allowing other resources to depend on it.

* The invocation of Task-Core will be baked into the TripleO client, it will
  not have to be invoked as a separate deployment step.

* Advanced users will be able to use Task-Core to meet their environment
  expectations without fully understanding the deployment nuance of multiple
  bespoke systems.

* Employs a validation system around inputs to ensure they are correct before
  starting the deployment. While the validation wont ensure an operational
  deployment, it will remove some issues caused by incorrect user input, such
  as missing dependent services or duplicate services; providing early feedback
  to deployers so they're able to make corrections before running longer
  operations.

Directord:
  Directord provides a modular execution platform that is aware of managed
  nodes. Because Directord leverages messaging, the platform can guarantee
  availability, transport, and performance. Directord has been built from the
  ground up, making use of industry-standard messaging protocols which ensure
  pseudo-real-time performance and limited resource utilization. The built-in
  DSL provides most of what the TripleO project will require out of the box.
  Because no solution is perfect, Directord utilizes a plugin system that will
  allow developers to create new functionality without compromise or needing to
  modify core components. Additionally, plugins are handled the same, allowing
  Directord to ensure the delivery and execution performance remain consistent.

* Directord is a single application that is ideally suited for containers while
  also providing native hooks into systems; this allows Directord to operate in
  heterogeneous environments. Because Directord is a simplified application,
  operators can choose how they want to run it and are not forced into a one size
  fits all solution.

* Directord is platform-agnostic, allowing it to run across systems, versions,
  and network topologies while simultaneously guaranteeing it maintains the
  smallest possible footprint.

* Directord is built upon messaging, giving it the unique ability to span
  network topologies with varying latencies; messaging protocols compensate for
  high latency environments and will finally give TripleO the ability to address
  multiple data-centers and fully embrace "the edge."

* Directord client/server communication is secured (TLS, etc) and encrypted.

* Directord node management to address unreachable or flapping clients.

With Task-Core and Directord, TripleO will have an intelligent dependency graph
that is both easy to understand and extend. TripleO will now be aware of things
like service dependencies, making it possible to run day two operations quickly
and more efficiently (e.g, update and restart only dependent services).
Finally, TripleO will shrink its maintenance burden by eliminating Ansible.


Alternatives
------------

Stay the course with Ansible

Continuing with Ansible for task execution means that the TripleO core team
embraces maintaining Ansible for the specific TripleO use case. Additionally,
the TripleO project begins documenting the scale limitations and the boundaries
that exist due to the nature of task execution. Focus needs to shift to the
required maintenance necessary for functional expectations TripleO.  Specific
Ansible versions also need to be maintained beyond their upstream lifecycle.
This maintenance would likely include maintaining an Ansible branch where
security and bug fixes could be backported, with our own project CI to validate
functionality.

TripleO could also embrace the use of `Ansible Execution Environments`_ through
continued investigative efforts. Although, if TripleO is already maintaining
Ansible, this would not be strictly required.


Security Impact
---------------

Task-Core and Directord are two new tools and attack surfaces, which will
require a new security assessment to be performed to ensure the tooling
exceeds the standard already set. That said, steps have already been taken to
ensure the new proposed architecture is FIPS_ compatible, and enforces
`transport encryption`_.

Directord also uses `ssh-python`_ for bootstrapping tasks.

Ansible will be removed, and will no longer have a security impact within
TripleO.


Upgrade Impact
--------------

The undercloud can be upgraded in place to use Directord and Task-Core. There
will be upgrade tasks that will migrate the undercloud as necessary to use the
new tools.

The overcloud can also be upgraded in place with the new tools. Upgrade tasks
will be migrated to use the Directord DSL just like deployment tasks. This spec
proposes no changes to the overcloud architecture itself.

As part of the upgrade task migration, the tasks can be rewritten to take
advantage of the new features exposed by these tools. With the introduction of
Task-Core, upgrade tasks can use well-defined dependencies for dynamic
ordering. Just like deployment, update/upgrade times will be decreased due to
the aniticipated performance increases.


Other End User Impact
---------------------

When following the `happy path`_, the end-user, deployers, and operators will
not interact with this change as the user interface will effectively remain the
same. However the user experience will change. Operators accustomed to Ansible
tasks, logging, and output, will instead need to become familiar with those
same aspects of Directord and Task-Core.

If an operator wishes to leverage the advanced capabilities of either
Task-Core or Directord, the tooling will have documented end user interfaces
available for interfaces such as custom components and orchestrations.

It should be noted that there's a change in deployment architecture in that
Directord follows a server/client model; albeit an ephemeral one. This change
aims to be fully transparent, however, it is something that end users,
deployers, will need to be aware of.


Performance Impact
------------------

This specification will have a positive impact on performance.  Due to the
messaging architecture of Directord, near-realtime task execution will be
possible in parallel across all nodes.

* Performance_ analysis has been done comparing configurability and runtime of
  Directord vs. Ansible, the TripleO default orchestration tool. This analysis
  highlights some of the performance gains this specification will provide;
  initial testing suggests that Task-Core and Directord is more than 10x
  faster than our current tool chain, representing a potential 90% time savings
  in just the task execution overhead.

* One of the goals of this specification is to remove impediments in the time
  to work. Deployers should not be spending exorbitant time waiting for tools to
  do work; in some cases, waiting longer for a worker to be available than it
  would take to perform a task manually.

* Improvements from being able to execute more efficiently in parallel.  The
  Ansible strategy work allowed us to run tasks from a given Ansible play in
  parallel accoss the nodes. However this was limited to a effectively a single
  play per node in terms of execution.  The granularity was limited to a play
  such that an Ansible play that with 100 items of work for one role and 10
  items of work would be run in parallel on the nodes. The role with 10 items
  of work would likely finish first and the overall execution would have to
  wait until the entire play was completed everywhere. The long pole for a
  play's execution is the node with the most set of tasks.  With the transition
  to task-core and directord, the overall unit of work is an orchestration
  which may have 5 tasks. If we take the same 100 tasks for one role and split
  them up into 20 orchestrations that can be run in parallel, and the 10 items
  of work into two orchestrations for the other roles. We are able to better
  execute the work in parallel when there are no specific ordering
  requirements. Improvements are expected around host prep tasks and other
  services where we do not have specific ordering requirements. Today these
  tasks get put in a random spot within a play and have to wait on other
  unrelated tasks to complete before being run.  We expect there to be less
  execution overhead time per the other items in this section, however the
  overall improvements are limited based on how well we can remove unnecessary
  ordering requirements.

* Deployers will no longer be required to run a massive server for medium-scale
  deployment. Regardless of size, the memory footprint and compute cores needed
  to execute a deployment will be significantly reduced.


Other Deployer Impact
---------------------

Task-Core and Directord represent an unknown factor; as such, they are
**not** battle-tested and will create uncertainty in an otherwise "stable_"
project.

Deployers will experience the time savings of doing deployments.  Deployers who
implement new services will need to do so with Directord and Task-Core.

Extensive testing has been done;
all known use-cases, from system-level configuration to container pod
orchestration, have been covered, and automated tests have been created to
ensure nothing breaks unexpectedly. Additionally, for the first time, these
projects have expectations on performance, with tests backing up those claims,
even at a large scale.

At present, TripleO assumes SSH access between the Undercloud and
Overcloud is always present. Additionally, TripleO believes the infrastructure
is relatively static, making day two operations risky and potentially painful.
Task-Core will reduce the computational burden when crafting action plans, and
Directord will ensure actions are always performed against the functional
hosts.

Another improvement this specification will enhance is in the area of vendor
integrations. Vendors will be able to provide meaningful task definitions which
leverage an intelligent inventory and dependency system. No longer will TripleO
require vendors have in-depth knowledge of every deployment detail, even those
outside of the scope of their deliverable. By easing the job definitions,
simplifying the development process, and speeding up the execution of tasks are
all positive impacts on deployers.

Test clouds are still highly recommended sources of information; however,
system requirements on the Undercloud will reduce. By reducing the resources
required to operate the Undercloud, the cost of test environments, in terms of
both hardware and time, will be significantly lowered. With a lower barrier to
entry developers and operators alike will be able to more easily contribute to
the overall project.


Developer Impact
----------------

To fully realize the benefits of this specification Ansible tasks will need to
be refactored into the Task-Core scheme. While Task-Core can run Ansible and
Directord has a plugin system which easily allows developers to port legacy
modules into Directord plugins, there will be a developer impact as the TripleO
development methodology will change. It's fair to say that the potential
developer impact will be huge, yet, the shift isn't monumental. Much of the
Ansible presently in TripleO is shell-oriented, and as such, it is easily
portable and as stated, compatibility layers exist allowing the TripleO project
to make the required shift gradually. Once the Ansible tasks are
ported, the time saved in execution will be significant.

Example `Task-Core and Directord implementation for Keystone`_:
  While this implementation example is fairly basic, it does result in a
  functional Keystone environment and in roughly 5 minutes and includes
  services like MySQL, RabbitMQ, Keystone as well as ensuring that the
  operating systems is setup and configured for a cloud execution environment.
  The most powerful aspect of this example is the inclusion of the graph
  dependency system which will allow us easily externalize services.

* The use of advanced messaging protocols instead of SSH means TripleO can more
  efficiently address deployments in local data centers or at the edge

* The Directord server and storage can be easily offloaded, making it possible
  for the TripleO Client to be executed from simple environments without access
  to the overcloud network; imagine running a massive deployment from a laptop.


Implementation
==============

In terms of essential TripleO integration, most of the work will occur within
the tripleoclient_, with the following new workflow.

`Execution Workflow`_::

    ┌────┐   ┌─────────────┐   ┌────┐   ┌─────────┐   ┌─────────┬──────┐   ???????????
    │USER├──►│TripleOclient├──►│Heat├──►│Task-Core├──►│Directord│Server├──►? Network ?
    └────┘   └─────────────┘   └────┘   └─────────┘   └─────────┴──────┘   ???????????
                    ▲                                             ▲             ▲
                    │                       ┌─────────┬───────┐   |             |
                    └──────────────────────►│Directord│Storage│◄──┘             |
                                            └─────────┴───────┘                 |
                                                                                |
                                                      ┌─────────┬──────┐        |
                                                      │Directord│Client│◄───────┘
                                                      └─────────┴──────┘

* Directord|Server - Task executor connecting to client.

* Directord|Client - Client program running on remote hosts connecting back to
  the Directord|Server.

* Directord|Storage - An optional component, when not externalized, Directord will
  maintain the runtime storage internally. In this configuration Directord is
  ephemeral.

To enable a gradual transition, ansible-runner_ has been implemented within
Task-Core, allowing the TripleO project to convert playbooks into tasks that
rely upon strongly typed dependencies without requiring a complete rewrite. The
initial implementation should be transparent. Once the Task-Core hooks are set
within tripleoclient_ functional groups can then convert their tripleo-ansible_
roles or ad-hoc Ansible tasks into Directord orchestrations. Teams will have
the flexibility to transition code over time and are incentivized by a
significantly improved user experience and shorter time to delivery.


Assignee(s)
-----------

Primary assignee:
  * Cloudnull - Kevin Carter
  * Mwhahaha - Alex Schultz
  * Slagle - James Slagle


Other contributors:
  * ???


Work Items
----------

#. Migrate Directord and Task-Core to the OpenStack namespace.
#. Package all of Task-Core, Directord, and dependencies for pypi
#. RPM Package all of Task-Core, Directord, and dependencies for RDO
#. Directord container image build integration within TripleO / tcib
#. Converge on a Directord deployment model (container, system, hybrid).
#. Implement the Task-Core code path within TripleO client.
#. Port in template Ansible tasks to Directord orchestrations.
#. Port Ansible roles into Directord orchestrations.
#. Port Ansible modules and actions into pure Python or Directord components
#. Port Ansible workflows in tripleoclient into pure Python or Directord
   orchestrations.
#. Migration tooling for Heat templates, Ansible roles/modules/actions.
#. Port Ansible playbook workflows in tripleoclient to pure Python or
   Directord orchestrations.
#. Undercloud upgrade tasks to migrate to Directord + Task-Core architecture
#. Overcloud upgrade tasks to migrate to enable Directord client bootstrapping


Dependencies
============

Both Task-Core and Directord are dependencies, as they're new projects. These
dependencies may or may not be brought into the OpenStack namespace;
regardless, both of these projects, and their associated dependencies, will
need to be packaged and provided for by RDO.


Testing
=======

If successful, the implementation of Task-Core and Directord will leave the
existing testing infrastructure unchanged. TripleO will continue to function as
it currently does through the use of the tripleoclient_.

New tests will be created to ensure the Task-Core and Directord components
remain functional and provide an SLA around performance and configurability
expectations.


Documentation Impact
====================

Documentation around Ansible will need to be refactored.

New documentation will need to be created to describe the advanced
usage of Task-Core and Directord. Much of the client interactions from the
"`happy path`_" will remain unchanged.


References
==========

* Directord official documentation https://directord.com

* Ansible's decision to pivot to execution environments:
  https://ansible-runner.readthedocs.io/en/latest/execution_environments.html

.. _Task-Core: https://github.com/mwhahaha/task-core

.. _Directord: https://github.com/cloudnull/directord

.. _General-Problem: https://xkcd.com/974

.. _`legacy tooling`: https://xkcd.com/1822

.. _`transport encryption`: https://directord.com/drivers.html

.. _FIPS: https://en.wikipedia.org/wiki/Federal_Information_Processing_Standards

.. _Performance: https://directord.com/overview.html#comparative-analysis

.. _practical: https://xkcd.com/382

.. _stable: https://xkcd.com/1343

.. _validation: https://xkcd.com/327

.. _scheme: https://github.com/mwhahaha/task-core/tree/main/schema

.. _`Task-Core and Directord implementation for Keystone`: https://raw.githubusercontent.com/mwhahaha/task-core/main/examples/directord/services/openstack-keystone.yaml

.. _`happy path`: https://xkcd.com/85

.. _tripleoclient: https://github.com/openstack/python-tripleoclient

.. _`Execution Workflow`: https://review.opendev.org/c/openstack/tripleo-heat-templates/+/798747

.. _ansible-runner: https://github.com/ansible/ansible-runner

.. _tripleo-ansible: https://github.com/openstack/tripleo-ansible

.. _`Ansible Execution Environments`: https://ansible-runner.readthedocs.io/en/latest/execution_environments.html

.. _`ssh-python`: https://pypi.org/project/ssh-python
