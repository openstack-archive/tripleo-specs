..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Improve logging for ansible calls in tripleoclient
==================================================

Launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/ansible-logging-tripleoclient

Problem description
===================
Currently, the ansible playbooks logging as shown during a deploy, upgdate or
upgrade (called "actions" later) is either too verbose, or not enough.

We also don't really have any history about the actions done on the stack,
meaning we're essentially unable to say when something was changed, was it
a success or a failure, and so on.

Requirements for the solution
=============================
No new service addition
-----------------------
We are already trying to remove things from the Undercloud, such as Mistral,
it's not in order to add new services.

No increase in deployment and day-2 operations time
---------------------------------------------------
The solution must not increase the time taken for deploy, update, upgrades,
scaling and any other actions. It must be 100% transparent to the operator.

Use existing tools
------------------
In the same way we don't want to have new services, we don't want to reinvent
the wheel once more, and we must check the already huge catalog of existing
solutions.

KISS
----
Keep It Simple Stupid is a key element - code must be easy to understand and
maintain.

Proposed Change
===============

Introduction
------------
While working on the `Validation Framework`_, a big part was about the logging.
There, we found a way to get an actual computable output, and store it in a
defined location, allowing to provide a nice interface in order to list and
show validation runs.

This heavily relies on an ansible callback plugin with specific libs, which are
shipped in `python-validations-libs`_ package.

Since the approach is modular, those libs can be re-used pretty easily in other
projects.

In addition, python-tripleoclient already depends on `python-validations-libs`_
(via a dependency on validations-common), meaning we already have the needed
bits.

The Idea
--------
Since we have the mandatory code already present on the system (provided by the
new `python-validations-libs`_ package), we can modify how ansible-runner is
configured in order to inject a callback, and get the output we need in both
the shell (direct feedback to the operator) and in a dedicated file.

Direct feedback
---------------
The direct feedback will tell the operator about the current task being done
and, when it ends, if it's a success or not.

This allows the operator to actually knows something is being done instead of
waiting in front of a non-moving shell.

File logging
------------
Here, we must define multiple things, and take into account we're running
multiple playbooks, with multiple calls to ansible-runner.

File location
.............
In order to make things easy, and for the sake of consistency, the logs should
be pushed in a subdirectory located in /var/log.

Since those logs are related to tripleo actions, an understandable name might
be:

* /var/log/tripleo-actions

In order to avoid any right issues, it should be owned by root, with the group
set to the user group running the CLI, and a 0770 mode. Since it can contain
secrets, we must ensure only root and the running user (usually "stack") are
able to access this location.

An alternative to the static ownership is to use ACLs on the directory, using
`ansible "acl"`_ module. This might provide an easier way to manage the
accesses to the files while ensuring no wild access is allowed. Please note we
currently have no ACLs managed within tripleo.

In any cases, appart from "root", we shouldn't hard-code any user. We will
therefore probably need to add some new parameters in order to set a proper
access list. The ACLs might really be easier for this purpose, especially if
we need more than one non-root user to access the logs.

As it's needed on the undercloud only, a new parameter should then be added to
the undercloud.conf - maybe something like "validations_log_access" with a coma
separated list of users.

File format convention
......................
In order to make the logs easily usable by automated tools, and since we
already heavily rely on JSON, the log output should be formated as JSON. This
would allow to add some new CLI commands such as "history list", "history show"
and so on.

Also, JSON being well known by logging services such as ElasticSearch, using it
makes sending them to some central logging service really easy and convenient.

Filename convention
...................
As said, we're running multiple playbooks during the actions, and we also want
to have some kind of history.

In order to do that, the easiest way to get a name is to concatenate the time
and the playbook name, something like:

* *timestamp*-*playbookname*.json

Does it meet the requirements?
------------------------------
* No service addition: yes - it's only a change in the CLI, no new dependecy is
  needed (tripleoclient already depends on validations-common, which depends on
  validations-libs)
* No increase in operation time: yes - it's just writing a new file on the
  disk. In addition, it will make debugging tasks easier, with the ability to
  provide a history of passed runs.
* Existing Tool: yes
* Actively maintained: so far, yes - expected to be extended outside of TripleO
* KISS: yes, based on the validations-libs and simple Ansible callback

Alternatives
============

ARA
---
`ARA Records Ansible`_ provides some of the functionnalities we implemented in
the Validation Framework logging, but it lacks some of the wanted features,
such as

* CLI integration within tripleoclient
* Third-party service independency
* plain file logging in order to scrap them with SOSReport or other tools

ARA needs a DB backend - we could inject results in the existing galera DB, but
that might create some issues with the concurrent accesses happening during a
deploy for instance. Using sqlite is also an option, but it means new packages,
new file location to save, binary format and so on.

It also needs some web server in order to show the reporting, meaning yet
another httpd configuration, and the need to access to it on the undercloud.

Also, ARA being a whole service, it would require to deploy it, configure it,
and maintain it - plus ensure it is properly running before each action in
order to ensure it gets the logs.

By default, ARA doesn't affect the actual playbook output, while the goal of
this spec is mostly about it: provide a concise feedback to the operator, while
keeping the logs on disk, in files, with the ability to interact with them
through the CLI directly.

In the end, ARA might be a solution, but it will require more work to get it
integrated, and, since the Triple UI has been deprecated, there isn't real way
to integrate it in an existing UI tool.

Would it meet the requirements?
...............................
* No service addition: no, due to the "REST API" aspect. A service must answer
  API calls
* No increase in operation time: probably yes, depending on the way ARA can
  manage inputs queues
* Existing tool: yes
* Actively maintained: yes
* KISS: yes, but it adds new dependencies (DB backend, Web server, ARA service,
  and so on)

Proposed Roadmap
================
In Victoria:

* Ensure we have all the ABI capabilities within validations-libs in order to
  set needed/wanted parameters for a different log location and file naming
* Start to work on the ansible-runner calls so that it uses a tweaked callback,
  using the validations-libs capabilities in order to get the direct feedback
  as well as the formatted file in the right location

Security Impact
===============
As we're going to store full ansible output on the disk, we must ensure log
location accesses are closed to any non-wanted user. As stated while talking
about the file location, the directory mode and ownership must be set so that
only the needed users can access its content (root + stack user)

Once this is sorted out, no other security impact is to be expected - further
more, it will even make things more secure than now, since the current way
ansible is launched within tripleoclient puts an "ansible.log" file in the
operator home directory without any specific rights.

Upgrade Impact
==============
Appart from ensuring the log location exists, there isn't any major upgrade
impact. A doc update must be done in order to point to the log location, as
well as some messages within the CLI.

End User Impact
===============
There is two impacts to the End User:

* CLI output will be reworked in order to provide useful information (see
  Direct Feedback above)
* Log location will change a bit for the ansible part (see File Logging above)

Performance Impact
==================
No impact is expected here. But a PoC must be done in order to ensure the
callback doesn't do anything bad.

Deployer Impact
===============
Same as End User Impact: CLI output will be changed, and the log location will
be updated.

Developer Impact
================
The callback is enabled by default, but the Developer might want to disable it.
Proper doc should reflect this. No real impact in the end.

Implementation
==============
Contributors
------------
* Cédric Jeanneret
* Mathieu Bultel

Work Items
----------
* Modify validations-libs in order to provided the needed interface (shouldn't
  be really needed, the libs are already modular and should expose the wanted
  interfaces and parameters)
* Create a new callback in tripleo-ansible
* Ensure the log directory is created with the correct rights
* Update the ansible-runner calls to enable the callback by default
* Ensure tripleoclient outputs status update on a regular basis while the logs
  are being written in the right location
* Update/create the needed documentations about the new logging location and
  management

.. _Validation Framework: https://specs.openstack.org/openstack/tripleo-specs/specs/stein/validation-framework.html
.. _python-validations-libs: https://opendev.org/openstack/validations-libs
.. _ARA Records Ansible: https://ara.recordsansible.org/
.. _ansible "acl": https://docs.ansible.com/ansible/latest/modules/acl_module.html
