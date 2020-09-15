..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Disable Swift from the Undercloud
=================================

The goal of this proposal is to introduce the community to the idea of
disabling Swift on the TripleO Undercloud. Within this propose we intend
to provide a high-level overview of how we can accomplish this goal.


Problem Description
===================

Swift is being used to store objects related to the deployment which are
managed entirely on the Undercloud. In the past, there was an API / UI to
interact with the deployment tooling; however, with the deprecation of the UI
and the removal of Mistral this is no longer the case. The Undercloud is
assumed to be a single node which is used to deploy OpenStack clouds, and
requires the user to login to the node to run commands. Because we're no longer
attempting to make the Undercloud a distributed system there's no need for an
API'able distributed storage service. Swift, in it's current state, is
under-utilized and carries unnecessary operational and resource overhead.


Proposed Change
===============

Overview
--------

Decommission Swift from the Undercloud.

To decommission Swift, we'll start by removing all of the `tripleoclient` Swift
interactions. These interactions are largely storing and retrieving YAML files
which provide context to the user for current deployment status. To ensure
we're not breaking deployment expectations, we'll push everything to the local
file system and retain all of the file properties wherever possible. We will
need coordinate with tripleo-ansible to ensure we're making all direct Swift
client and module interactions optional.

Once we're able to remove the `tripleoclient` Swift interactions, we'll move to
disable Swift interactions from tripleo-common. These interactions are similar
to the ones found within the `tripleoclient`, though tripleo-common has some
complexity; we'll need to ensure we're not breaking expectations we've created
with our puppet deployment methodologies which have some Swift assumptions.


Alternatives
------------

We keep everything as-is.


Security Impact
---------------

There should be no significant security implications when disabling Swift.
It could be argued that disabling Swift might make the deployment more secure,
it will lessen the attack surface; however, given the fact that Swift on the
Undercloud is only used by director I would consider any benefit insignificant.


Upgrade Impact
--------------

There will be no upgrade impact; this change will be transparent to the
end-user.


Other End User Impact
---------------------

None.


Performance Impact
------------------

Disabling Swift could make some client interactions faster; however, the
benefit should be negligible. That said, disabling Swift would remove a
service on the Undercloud, which would make setup faster and reduce the
resources required to run the Undercloud.


Other Deployer Impact
---------------------

Operationally we should see an improvement as it will no longer be required to
explore a Swift container, and download files to debug different parts of the
deployment. All deployment related file artifacts housed within Swift will
exist on the Undercloud using the local file system, and should be easily
interacted with.


Developer Impact
----------------

None, if anything disabling Swift should make the life of a TripleO developer
easier.


Implementation
==============

Excising Swift client interactions will be handled directly in as few reviews
as possible; hopefully allowing us to backport this change, should it be deemed
valuable to stable releases.

All of the objects stored within Swift will be stored in
`/var/lib/tripleo/{named_artifact_directories}`. This will allow us to
implement all of the same core logic in our various libraries just without the
use of the API call to store the object.

In terms of enabling us to eliminate swift without having a significant impact
on the internal API we'll first start by trying to replace the swift object
functions within tripleo-common with local file system calls. By using the
existing functions and replacing the backend we'll ensure API compatibility and
lessen the likely hood of creating regressions.

.. note::

  We'll need to collaborate with various groups to ensure we're porting assumed
  functionality correctly. While this spec will not go into the specifics
  implementation details for porting assumed functionality, it should be known
  that we will be accountable for ensuring existing functionality is ported
  appropriately.


Assignee(s)
-----------

Primary assignee:
  cloudnull

Other contributors:

- emilien
- ekultails

Work Items
----------

The work items listed here are high level, and not meant to provide specific
implementation details or timelines.

* Enumerate all of the Swift interactions
* Create a space on the Undercloud to house the files
* This location will be on the local file system and will be created into a
  git archive; git is used for easier debug, rapid rollback, and will
  provide simple versioning.
* Create an option to disable Swift on the Undercloud.
* Convert client interactions to using the local file system
* Ensure all tripleo-ansible Swift client calls are made optional
* Convert tripleo-common Swift interactions to using the local file system
* Disable Swift on the Undercloud


Dependencies
============

Before Swift can be disabled on the Undercloud we will need ensure the
deployment methodology has been changed to Metalsmith.


Testing
=======

The Swift tests will need to be updated to use the local file system, however
the existing tests and test structure will be reused.


Documentation Impact
====================

There are several references to Swift in our documentation which we will need to
update.


References
==========

* https://etherpad.opendev.org/p/tripleo-heat-swift-removal-undercloud
* http://paste.openstack.org/show/798208
