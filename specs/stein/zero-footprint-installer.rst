..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
TripleO Zero Footprint Installer
================================

https://blueprints.launchpad.net/tripleo/+spec/zero-footprint

This spec introduces support for an installer mode which has zero
(or at least much less) dependencies than we do today. It is meant
to be an iteration of the Undercloud and All-In-One (standalone)
installers that allows you to end up with the same result without
having to install all of the TripleO dependencies on your host machine.

Problem Description
===================

Installing python-tripleoclient on a host machine currently installs
a lot of dependencies many of which may be optional for smaller
standalone type installations. Users of smaller standalone installations
can have a hard time understanding the differences between what TripleO
dependencies get installed vs which services TripleO installs.

Additionally, some developers would like a fast-track way to develop and
run playbooks without requiring local installation of an Undercloud which
in many cases is done inside a virtual machine to encapsulate the dependencies
that get installed.

Proposed Change
===============

A new zero footprint installer can help drive OpenStack Tripleoclient
commands running within a container. Using this approach you can:

1. Generate Ansible playbooks from a set of Heat templates
   (tripleo-heat-templates), Heat environments, and Heat parameters
   exactly like we do today using a Container. No local dependencies
   would be required to generate the playbooks.

2. (optionally) Execute the playbooks locally on the host machine. This would
   require some Ansible modules to be installed that TripleO depends on but
   is a much smaller footprint than what we require elsewhere today.

Alternatives
------------

Create a subpackage of python-tripleoclient which installs less dependencies.
The general footprint of required packages would still be quite high (lots
of OpenStack packages will still be installed for the client tooling).

Or do nothing and continue to use VMs to encapsulate the dependencies for
an Undercloud/All-In-One installer and generate Ansible playbooks. Setting
up a local VM requires more initial setup and dependencies however and is
heavier than just using a local container to generate the same playbooks.

Security Impact
---------------

As a container will be used to generate Ansible playbooks the user may
need to expose some local data/files to the installer container. This is
likely a minimal concern as we already require this data to be exposed to
the Undercloud and All-In-One installers.

Other End User Impact
---------------------

None

Performance Impact
------------------

Faster deployment and testing of local All-On-One setups.

Other Deployer Impact
---------------------

None


Developer Impact
----------------

Faster deployment and testing of local All-On-One setups.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dprince

Work Items
----------

* A new 'tripleoclient' container
* New project to drive the installation (Talon?)
* Continue to work on refining the Ansible playbook modules to provide a
  cleaner set of playbook dependencies. Specifically those that depend on
  the any of the traditional TripleO/Heat agent hooks and scripts.
* documentation updates

Dependencies
============

None.

Testing
=======

This new installer can likely suppliment or replace some of the testing we
are doing for All-In-One (standalone) deployments in upstream CI.

Documentation Impact
====================

Docs will need to be updated.

References
==========

None
