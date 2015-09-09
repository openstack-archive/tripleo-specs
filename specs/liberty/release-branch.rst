..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Release Branch proposal for TripleO
==========================================

To date, the majority of folks consuming TripleO have been doing so via the
master branches of the various repos required to allow TripleO to deploy
an OpenStack cloud.  This proposes an alternative "release branch" methodology
which should enable those consuming stable OpenStack releases to deploy
more easily using TripleO.


Problem Description
===================

Historically strong guarantees about deploying the current stable OpenStack
release have not been made, and it's not something we've been testing in
upstream CI.  This is fine from a developer perspective, but it's a major
impediment to those wishing to deploy production clouds based on the stable
OpenStack releases/branches.

Proposed Change
===============

I propose we consider supporting additional "release" branches, for selected
TripleO repos where release-specific changes are required.

The model will be based on the stable branch model[1] used by many/most
OpenStack projects, but with one difference, "feature" backports will be
permitted provided they are 100% compatible with the currently released
OpenStack services.

Overview
--------

The justification for allowing features is that many/most TripleO features are
actually enabling access to features of OpenStack services which will exist in
the stable branches of the services being deployed.  Thus, the target audience
of this branch will likely want to consume such "features" to better access
features and configurations which are appropriate to the OpenStack release they
are consuming.

The other aspect of justification is that projects are adding features
constantly, thus it's unlikely TripleO will be capable of aligning with every
possible new feature for, say Liberty, on day 1 of the release being made.  The
recognition that we'll be playing "catch up", and adopting a suitable branch
policy should mean there is scope to continue that alignment after the services
themselves have been released, which will be of benefit to our users.

Changes landing on the master branch can be considered as valid candidates for
backport, unless:

* The patch requires new features of an OpenStack service (that do not exist
  on the stable branches) to operate. E.g if a tripleo-heat-templates change
  needs new-for-liberty Heat features it would *not* be allowed for release/kilo.

* The patch enables Overcloud features of an OpenStack service that do not
  exist on the stable branches of the supported Overcloud version (e.g for
  release/kilo we only support kilo overcloud features).

* User visible interfaces are modified, renamed or removed - removal of
  deprecated interfaces may be allowed on the master branch (after a suitable
  deprecation period), but these changes would *not* be valid for backport as
  they could impact existing users without warning.  Adding new interfaces
  such as provider resources or parameters would be permitted provided the
  default behavior does not impact existing users of the release branch.

* The patch introduces new dependencies or changes the current requirements.txt.

To make it easier to identify not-valid-for-backport changes, it's proposed
that a review process be adopted whereby a developer proposing a patch to
master would tag a commit if it doesn't meet the criteria above, or there is
some other reason why the patch would be unsuitable for backport.

e.g:

  No-Backport: This patch requires new for Mitaka Heat features


Alternatives
------------

The main alternative to this is to leave upstream TripleO as something which
primarily targets developer/trunk-chasing users, and leave maintaining a
stable branch of the various components to downstream consumers of TripleO,
rdo-manager for example.

The disadvantage of this approach is it's an impediment to adoption and
participation in the upstream project, so I feel it'd be better to do this work
upstream, and improve the experience for those wishing to deploy via TripleO
using only the upstream tools and releases.


Security Impact
---------------

We'd need to ensure security related patches landing in master got
appropriately applied to the release branches (same as stable branches for all
other projects).

Other End User Impact
---------------------

This should make it much easier for end users to stand up a TripleO deployed
cloud using the stable released versions of OpenStack services.

Other Deployer Impact
---------------------

This may reduce duplication of effort when multiple downstream consumers of
TripleO exist.

Developer Impact
----------------

The proposal of valid backports will ideally be made by the developer
proposing a patch to the master branch, but avoid creating an undue barrier to
entry for new contributors this will not be mandatory, but will be reccomended
and encouraged via code review comments.

Standard stable-maint processes[1] will be observed when proposing backports.

We need to consider if we want a separate stable-maint core (as is common on
most other projects), or if all tripleo-core members can approve backports.
Initially it is anticipated to allow all tripleo-core, potentially with the
addition of others with a specific interest in branch maintenance (e.g
downstream package maintainers).

Implementation
==============

Initially the following repos will gain release branches:

* openstack/tripleo-common
* openstack/tripleo-docs
* openstack/tripleo-heat-templates
* openstack/tripleo-puppet-elements
* openstack/python-tripleoclient
* openstack/instack-undercloud

These will all have a new branch created, ideally near the time of the upcoming
liberty release, and to avoid undue modification to existing infra tooling,
e.g zuul, they will use the standard stable branch naming, e.g:

* stable/liberty

If any additional repos require stable branches, we can add those later when
required.

It is expected that any repos which don't have a stable/release branch must
maintain compatibility such that they don't break deploying the stable released
OpenStack version (if this proves impractical in any case, we'll create
branches when required).

Also, when the release branches have been created, we will explicitly *not*
require the master branch for those repos to observe backwards compatibility,
with respect to consuming new OpenStack features. For example, new-for-mitaka
Heat features may be consumed on the master branch of tripleo-heat-templates
after we have a stable/liberty branch for that repo.

Assignee(s)
-----------

Primary assignee:
  shardy

Other contributors:
  TBC

Work Items
----------

1. Identify the repos which require release branches
2. Create the branches
3. Communicate need to backport to developers, consider options for automating
4. CI jobs to ensure the release branch stays working
5. Documentation to show how users may consume the release branch

Testing
=======

We'll need CI jobs configured to use the TripleO release branches, deploying
the stable branches of other OpenStack projects.  Hopefully we can make use of
e.g RDO packages for most of the project stable branch content, then build
delorean packages for the tripleo release branch content.

Ideally in future we'd also test upgrade from one release branch to another
(e.g current release from the previous, and/or from the release branch to
master).

As a starting point derekh has suggested we create a single centos job, which
only tests HA, and that we'll avoid having a tripleo-ci release branch,
ideally using the under development[2] tripleo.sh developer script to abstract
any differences between deployment steps for branches.

Documentation Impact
====================

We'll need to update the docs to show:

1. How to deploy an undercloud node from the release branches using stable
OpenStack service versions
2. How to build images containing content from the release branches
3. How to deploy an overcloud using only the release branch versions

References
==========

We started discussing this idea in this thread:

http://lists.openstack.org/pipermail/openstack-dev/2015-August/072217.html

[1] https://wiki.openstack.org/wiki/StableBranch
[2] https://review.openstack.org/#/c/225096/
