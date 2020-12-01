..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Ephemeral Heat Stack for all deployments
========================================

https://blueprints.launchpad.net/tripleo/+spec/ephemeral-heat-overcloud

This spec proposes using the ephemeral Heat stack model for all deployments
types, including the overcloud.  Using ephemeral Heat is already done for
standalone deployments with the "tripleo deploy" command, and for the
undercloud install as well. Expanding its use to overcloud deployments will
align the different deployment methods into just a single method. It will also
make the installation process more stateless and with better predictability
since there is no Heat stack to get corrupted or possibly have bad state or
configuration.


Problem Description
===================

* Maintaining the Heat stack can be problematic due to corruption via either
  user or software error. Backups are often not available, and even when they
  exist, they are no guarantee to recover the stack. Corruption or loss of the
  Heat stack, such as accidental deletion, requires custom recovery procedures
  or re-deployments.

* The Heat deployment itself must be maintained, updated, and upgraded. These
  tasks are not large efforts, but they are areas of maintenance that would be
  eliminated when using ephemeral Heat instead.

* Relying on the long lived Heat process makes the deployment less portable in
  that there are many assumptions in TripleO that all commands are run
  directly from the undercloud. Using ephemeral Heat would at least allow for
  the stack operation and config-download generation to be entirely portable
  such that it could be run from any node with python-tripleoclient installed.

* There are large unknowns in the state of each Heat stack that exists for all
  current deployments. These unknowns can cause issues during update/upgrade as
  we can't possibly account for all of these items, such as out of date
  parameter usage or old/incorrect resource registry mappings. Having each
  stack operation create a new stack will eliminate those issues.


Proposed Change
===============

Overview
--------

The ephemeral Heat stack model involves starting a short lived heat process
using a database engine for the purposes of creating the stack. The initial
proposal assumes using the MySQL instance already present on the undercloud as
the database engine. To maintain compatibility with the already implemented
"tripleo deploy" code path, SQLite will also be supported for single node
deployments.  SQLite may also be supported for other deployments of
sufficiently small size so as that SQLite is not a bottleneck.

After the stack is created, the config-download workflow is run to download and
render the ansible project directory to complete the deployment. The short
lived heat process is killed and the database is deleted, however, enough
artifacts are saved to reproduce the Heat stack if necessary including the
database dump. The undercloud backup and restore procedure will be modified to
account for the removal of the Heat database.

This model is already used by the "tripleo deploy" command for the standalone
and undercloud installations and is well proven for those use cases. Switching
the overcloud deployment to also use ephemeral Heat aligns all of the different
deployments to use Heat the same way.

We can scale the ephemeral Heat processes by using a podman pod that
encapsulates containers for heat-api, heat-engine, and any other process we
needed. Running separate Heat processes containerized instead of a single
heat-all process will allow starting multiple engine workers to allow for
scale. Management and configuration of the heat pod will be fairly prescriptive
and it will use default podman networking as we do not need the Heat processes
to scale beyond a single host. Moving forward, undercloud minions will no
longer install heat-engine process as a means for scale.

As part of this change, we will also add the ability to run Heat commands
against the saved database from a given deployment. This will give
operators a way to inspect the Heat stack that was created for debugging
purposes.

Managing the templates used during the deployment becomes even more important
with this change, as the templates and environments passed to the "overcloud
deploy" command are the entire source of truth to recreate the deployment. We
may consider further management around the templates, such as a git repository
but that is outside the scope of this spec.

There are some cases where the saved state in the stack is inspected before a
deployment operation. Two examples are comparing the Ceph fsid's between the
input and what exists in the stack, as well as checking for a missing
network-isolation.yaml environment.

In cases such as these, we need a way to perform these checks outside of
inspecting the Heat stack itself. A straightforward way to do these types of
checks would be to add ansible tasks that check the existing deployed overcloud
(instead of the stack) and then cause an error that will stop the deployment if
an invalid change is detected.

Alternatives
------------

The alternative is to make no changes and continue to use Heat as we do today
for the overcloud deployment. With the work that has already been done to
decouple Heat from Nova, Ironic, and now Neutron, it instead seems like the
next iterative step is to use ephemeral Heat for all of our deployment types.

Security Impact
---------------

The short lived ephemeral heat process uses no authentication. This is in
contrast to the Heat process we have on the undercloud today that uses Keystone
for authentication. In reality, this change has little effect on security as
all of the sensitive data is actually passed into Heat from the templates. We
should however make sure that the generated artifacts are secured
appropriately.

Since the Heat process is ephemeral, no change related to SRBAC (Secure RBAC)
is needed.

Upgrade Impact
--------------

When users upgrade to Wallaby, the Heat processes will be shutdown on the
undercloud, and further stack operations will use ephemeral Heat.

Upgrade operations for the overcloud will work as expected as all of the update
and upgrade tasks are entirely generated with config-download on each stack
operation. We will however need to ensure proper upgrade testing to be sure
that all services can be upgraded appropriately using ephemeral Heat.

Other End User Impact
---------------------

End users will no longer have a running instance of Heat to interact with or
run heat client commands against. However, we will add management around
starting an ephemeral Heat process with the previously used database for
debugging inspection purposes (stack resource list/show, etc).

Performance Impact
------------------

The ephemeral Heat process is presently single threaded. Addressing this
limitation by using a podman pod for the Heat processes will allow the
deployment to scale to meet overcloud deployment needs, while keeping the
process ephemeral and easy to manage with just a few commands.

Using the MySQL database instead of SQLite as the database engine should
alleviate any impact around the database being a bottleneck. After the
database is backed up after a deployment operation, it would be wiped from
MySQL so that no state is saved outside of the produced artifacts from the
deployment.

Alternatively, we can finish the work started in `Scaling with the Ansible
inventory`_. That work will enable deploying the Heat stack with a count of 1
for each role. With that change, the Heat stack operation times will scale with
the number of roles in the deployment, and not the number of nodes, which will
allow for similar performance as currently exists. Even while using the
inventory to scale, we are still likely to have worse performance with a single
heat-all process than we do today. With just a few roles, using just heat-all
becomes a bottleneck.

Other Deployer Impact
---------------------

Initially, deployers will have the option to enable using the ephemeral Heat
model for overcloud deployments, until it becomes the default.

Developer Impact
----------------

Developers will need to be aware of the new commands that will be added to
enable inspecting the Heat stack for debugging purposes.

In some cases, some service template updates may be required where there are
instances that those templates rely on saved state in the Heat stack.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  james-slagle

Work Items
----------

The plan is to start prototyping this effort and have the option in place to
use it for a default overcloud deployment in Wallaby. There may be additional
fine tunings that we can finish in the X release, with a plan to backport to
Wallaby. Ideally, we would like to make this the default behavior in Wallaby.
To the extent that is possible will be determined by the prototype work.

* Add management of Heat podman pod to tripleoclient
* Add option to "overcloud deploy" to use ephemeral Heat
* Use code from "tripleo deploy" for management of ephemeral Heat
* Ensure artifacts from the deployment are saved in known locations and
  reusable as needed
* Update undercloud backup/restore to account for changes related to Heat
  database.
* Add commands to enable running Heat commands with a previously used
  database
* Modify undercloud minion installer to no longer install heat-engine
* Switch some CI jobs over to use the optional ephemeral Heat
* Eventually make using ephemeral Heat the default in "overcloud deploy"
* Align the functionality from "tripleo deploy" into the "overcloud deploy"
  command and eventually deprecate "tripleo deploy".

Dependencies
============

This work depends on other ongoing work to decouple Heat from management of
other OpenStack API resources, particularly the composable networks v2 work.

* Network Data v2 Blueprint - https://blueprints.launchpad.net/tripleo/+spec/network-data-v2-ports

Testing
=======

Initially, the change will be optional within the "overcloud deploy" command.
We can choose some CI jobs to switch over to opt-in. Eventually, it will become
the default behavior and all CI jobs would then be affected.

Documentation Impact
====================

Documentation updates will be necessary to detail the changes around using
ephemeral Heat. Specifically:

* User Interface changes
* How to run Heat commands to inspect the stack
* Where artifacts from the deployment were saved and how to use them

References
==========

* `Scaling with the Ansible inventory`_ specification


.. _Scaling with the Ansible inventory: https://specs.openstack.org/openstack/tripleo-specs/specs/ussuri/scaling-with-ansible-inventory.html
