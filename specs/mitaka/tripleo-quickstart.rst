==================
TripleO Quickstart
==================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-quickstart

We need a common way for developers/CI systems to quickly stand up a virtual
environment.

Problem Description
===================

The tool we currently document for this use case is instack-virt-setup.
However this tool has two major issues, and some missing features:

* There is no upstream CI using it. This means we have no way to test changes
  other than manually. This is a huge barrier to adding the missing features.

* It relies on a maze of bash scripts in the incubator repository[1] in order
  to work. This is a barrier to new users, as it can take quite a bit of time
  to find and then navigate that maze.

* It has no way to use a pre-built undercloud image instead of starting from
  scratch and redoing the same work that CI and every other tripleo developer
  is doing on every run. Starting from a pre-built undercloud with overcloud
  images prebaked can be a significant time savings for both CI systems as well
  as developer test environments.

* It has no way to create this undercloud image either.

* There are other smaller missing features like automatically tagging the fake
  baremetals with profile capability tags via instackenv.json. These would not
  be too painful to implement, but without CI even small changes carry some
  amount of pain.

Proposed Change
===============

Overview
--------

* Import the tripleo-quickstart[2] tool that RDO is using for this purpose.
  This project is a set of ansible roles that can be used to build an
  undercloud.qcow2, or alternatively to consume it. It was patterned after
  instack-virt-setup, and anything configurable via instack-virt-setup is
  configurable in tripleo-quickstart.

* Use third-party CI for self-gating this new project. In order to setup an
  environment similar to how developers and users can use this tool, we need
  a baremetal host. The CI that currently self gates this project is setup on
  ci.centos.org[3], and setting this up as third party CI would not be hard.

Alternatives
------------

* One alternative is to keep using instack-virt-setup for this use case.
  However, we would still need to add CI for instack-virt-setup. This would
  still need to be outside of tripleoci, since it requires a baremetal host.
  Unless someone is volunteering to set that up, this is not really a viable
  alternative.

* Similarly, we could use some other method for creating virtual environments.
  However, this alternative is similarly constrained by needing third-party CI
  for validation.

Security Impact
---------------

None

Other End User Impact
---------------------

Using a pre-built undercloud.qcow2 drastically symplifies the virt-setup
instructions, and therefore is less error prone. This should lead to a better
new user experience of TripleO.

Performance Impact
------------------

Using a pre-built undercloud.qcow2 will shave 30+ minutes from the CI
gate jobs.

Other Deployer Impact
---------------------

There is no reason this same undercloud.qcow2 could not be used to deploy
real baremetal environments. There have been many production deployments of
TripleO that have used a VM undercloud.

Developer Impact
----------------

The undercloud.qcow2 approach makes it much easier and faster to reproduce
exactly what is run in CI. This leads to a much better developer experience.

Implementation
==============

Assignee(s)
-----------
Primary assignees:

* trown

Work Items
----------

* Import the existing work from the RDO community to the openstack namespace
  under the TripleO umbrella.

* Setup third-party CI running in ci.centos.org to self-gate this new project.
  (We can just update the current CI[3] to point at the new upstream location)

* Documentation will need to be updated for the virtual environment setup.

Dependencies
============

Currently, the only undercloud.qcow2 available is built in RDO. We would
either need to build one in tripleo-ci, or use the one built in RDO.

Testing
=======

We need a way to CI the virtual environment setup. This is not feasible within
tripleoci, since it requires a baremetal host machine. We will need to rely on
third party CI for this.

Documentation Impact
====================

Overall this will be a major simplification of the documentation.

References
==========

[1] https://github.com/openstack/tripleo-incubator/tree/master/scripts
[2] https://github.com/redhat-openstack/tripleo-quickstart
[3] https://ci.centos.org/view/rdo/job/tripleo-quickstart-gate-mitaka-delorean-minimal/
