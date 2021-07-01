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
Currently, the ansible playbooks logging as shown during a deploy or day-2
operations such us upgrade, update, scaling is either too verbose, or not
enough.

Furthermore, since we're moving to ephemeral services on the Undercloud (see
`ephemeral heat`_ for instance), getting information about the state, content
and related things is a bit less intuitive. A proper logging, with associated
CLI, can really improve that situation and provide a better user experience.


Requirements for the solution
=============================
No new service addition
-----------------------
We are already trying to remove things from the Undercloud, such as Mistral,
it's not in order to add new services.

No increase in deployment and day-2 operations time
---------------------------------------------------
The solution must not increase the time taken for deploy, update, upgrades,
scaling and any other day-2 operations. It must be 100% transparent to the
operator.

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

Since callback aren't cheap (but, hopefully not expensive either), proper PoC
must be conducted in order to gather metrics about CPU, RAM and time. Please
see Performance Impact section.

Direct feedback
---------------
The direct feedback will tell the operator about the current task being done
and, when it ends, if it's a success or not.

Using a callback might provide a "human suited" output.

File logging
------------
Here, we must define multiple things, and take into account we're running
multiple playbooks, with multiple calls to ansible-runner.

File location
.............
Nowadays, most if not all of the deploy related files are located in the
user home directory (i.e. ~/overcloud-deploy/<stack>/).
It therefore sounds reasonable to get the log in the same location, or a
subdirectory in that location.

Keeping this location also solves the potential access right issue, since a
standard home directory has a 0700 mode, preventing any other user to access
its content.

We might even go a bit deeper, and enforce a 0600 mode, just to be sure.

Remember, logs might include sensitve data, especially when we're running with
extra debugging.

File format convention
......................
In order to make the logs easily usable by automated tools, and since we
already heavily rely on JSON, the log output should be formated as JSON. This
would allow to add some new CLI commands such as "history list", "history show"
and so on.

Also, JSON being well known by logging services such as ElasticSearch, using it
makes sending them to some central logging service really easy and convenient.

While JSON is nice, it will more than probably prevent a straight read by the
operator - but with a working CLI, we might get something closer to what we
have in the `Validation Framework`_, for instance (see `this example`_). We
might even consider a CLI that will allow to convert from JSON to whatever
the operator might want, including but not limited to XML, plain text or JUnit
(Jenkins).

There should be a new parameter allowing to switch the format, from "plain" to
"json" - the default value is still subject to discussion, but providing this
parameter will ensure Operators can do whetever they want with the default
format. A concensus seems to indicate "default to plain".

Filename convention
...................
As said, we're running multiple playbooks during the actions, and we also want
to have some kind of history.

In order to do that, the easiest way to get a name is to concatenate the time
and the playbook name, something like:

* *timestamp*-*playbookname*.json

Use systemd/journald instead of files
.....................................
One might want to use systemd/journald instead of plain files. While this
sounds appealing, there are multiple potential issues:

#. Sensitive data will be shown in the system's journald, at hand of any other
   user
#. Journald has rate limitations and threshold, meaning we might hit them, and
   therefore lose logs, or prevent other services to use journald for their
   own logging
#. While we can configure a log service (rsyslog, syslog-ng, etc) in order to
   output specific content to specific files, we will face access issues on
   them

Therefore, we shouldn't use journald.

Does it meet the requirements?
------------------------------
* No service addition: yes - it's only a change in the CLI, no new dependecy is
  needed (tripleoclient already depends on validations-common, which depends on
  validations-libs)
* No increase in operation time: this has to be proven with proper PoC and
  metrics gathering/comparison.
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
  manage inputs queues. Since it's also using a callback, we have to account
  for the potential resources used by it.
* Existing tool: yes
* Actively maintained: yes
* KISS: yes, but it adds new dependencies (DB backend, Web server, ARA service,
  and so on)

Note on the "new dependencies": while ARA can be launched
`without any service`_, it seems to be only for devel purpose, according to the
informative note we can read on the documentation page::

  Good for small scale usage but inefficient and contains a lot of small files
  at a large scale.

Therefore, we shouldn't use ARA.

Proposed Roadmap
================
In Xena:

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
There are two impacts to the End User:

* CLI output will be reworked in order to provide useful information (see
  Direct Feedback above)
* Log location will change a bit for the ansible part (see File Logging above)

Performance Impact
==================
A limited impact is to be expected - but proper PoC with metrics must be
conducted to assess the actual change.

Multiple deploys must be done, with different Overcloud design, in order to
see the actual impact alongside the number of nodes.

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

.. _ephemeral heat: https://specs.openstack.org/openstack/tripleo-specs/specs/wallaby/ephemeral-heat-overcloud.html
.. _Validation Framework: https://specs.openstack.org/openstack/tripleo-specs/specs/stein/validation-framework.html
.. _this example: https://asciinema.org/a/283645
.. _python-validations-libs: https://opendev.org/openstack/validations-libs
.. _ARA Records Ansible: https://ara.recordsansible.org/
.. _without any service: https://ara.readthedocs.io/en/latest/cli.html#ara-manage-generate
.. _ansible "acl": https://docs.ansible.com/ansible/latest/modules/acl_module.html
