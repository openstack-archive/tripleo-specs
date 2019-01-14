..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Integrate os_tempest role with TripleO
======================================

Launchpad Blueprint:

https://blueprints.launchpad.net/tripleo/+spec/os-tempest-tripleo

Tempest provides a set of API and integrations tests with batteries
included in order to validate the OpenStack Deployment. In TripleO
project, we are working towards using a unified tempest role i.e.
`os_tempest` provided by OpenStack Ansible project in TripleO CI
in order to foster collaboration with multiple deployment tools and
improve our testing strategies within OpenStack Community.

Problem Description
===================

In the OpenStack Ecosystem, we have multiple *ansible based* deployment tools
that use their own roles for install/configure and running tempest testing.
Each of these roles is trying to do similar stuff tied to the different
deployment tools. For example: `validate-tempest` ansible role on TripleO CI
provides most of the stuff but it is tied with the TripleO deployment and
provides some nice feature (Like: bugcheck, failed tests email notification,
stackviz, python-tempestconf support for auto tempest.conf generation) which
are missing in other roles. It is leading to duplication and reduces what
tempest tests are not working across them, leading to no collaboration on
the Testing side.

The OpenStack Ansible team provides `os_tempest` role for installing/
configuring/running tempest and post tempest results processing and there
is a lot of duplication between their work and the roles used for testing
by the various deployment tools.It almost provides most of the stuff
provided by each of the deployment tool specific tempest roles. There are
few stuffs which are missing can be added in the role and make it useable
so that other deployment tools can consume it.

Proposed Change
===============

Using unified `os_tempest` ansible role in TripleO CI will help to maintain
one less role within TripleO project and help us to collaborate with
openstack-ansible team in order to share/improve tests strategies across
OpenStack ecosystem and solve tempest issues fastly.

In order to achieve that, we need:
 * Improve `os_tempest` role to add support for package/container install,
   python-tempestconf, stackviz, skip list, bugcheck, tempest
   log collection at the proper place.

 * Have a working CI job on standalone running tempest from `os_tempest`
   role as well as on OSA side.

 * Provide an easy migration path from validate-tempest role.

Alternatives
------------

If we do not use the existing `os_tempest` role then we need to re-write the
`validate-tempest` role which will result in again duplication and it will
cost too much time and it also requires another set of efforts for adoption
in the community which does not seems to feasible.

Security Impact
---------------

None

Upgrade Impact
--------------

None

Other End User Impact
---------------------

We need to educate users for migrating to `os_tempest`.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Helps more collaboration and improves testing.

Implementation
==============

Assignee(s)
-----------


Primary assignee:
  * Arx Cruz (arxcruz)
  * Chandan Kumar (chkumar246)
  * Martin Kopec (mkopec)


Work Items
----------

* Install tempest and it's dependencies from Distro packages
* Running tempest from containers
* Enable stackviz
* python-tempestconf support
* skiplist management
* Keeping all tempest related files at one place
* Bugcheck
* Standalone based TripleO CI job consuming os_tempest role
* Migration path from validate-tempest to os_tempest role
* Documentation update on How to use it
* RDO packaging

Dependencies
============

Currently, os_tempest role depends on `python_venv_build` role when
tempest is installed from source (git, pip, venv). We need to package it in RDO.

Testing
=======

The unified tempest role `os_tempest` will replace validate-tempest
role with much more improvements.


Documentation Impact
====================

Documentation on how to consume `os_tempest` needs to be updated.


References
==========

* Unified Tempest role creation & calloboration email:
  http://lists.openstack.org/pipermail/openstack-dev/2018-August/133838.html

* os_tempest role:
  http://git.openstack.org/cgit/openstack/openstack-ansible-os_tempest
