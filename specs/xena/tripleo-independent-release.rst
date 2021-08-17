
..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Moving TripleO repos to independent release model
=================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo

This spec proposes that we move all tripleo repos to the independent release
model. The proposal was first raised during tripleo irc meetings [1]_ and then
also on the openstack-discuss mailing list [2]_.

Problem Description
===================

The TripleO repos [3]_ mostly follow the cycle-with-intermediary release
model, for example tripleo-heat-templates at [4]_. Mostly because some of
tripleo repos use the independent release model, for example tripleo-upgrade
at [5]_. A description of the different release models can be found at [6]_.

By following the cycle-with-intermediary release model, TripleO is bound to
produce a release for each OpenStack development cycle and a corresponding
stable/branch in the tripleo repos. However as we have seen this causes an
ongoing maintenance burden; consider that currently TripleO supports 5
active branches - Train, Ussuri, Victoria, Wallaby and Xena (current master).
In fact until very recently that list contained 7 branches, including Stein
and Queens (currently moving to End Of Life [7]_).

This creates an ongoing maintenance and resource burden where for each
branch we are backporting changes, implementing, running and maintaining
upstream CI and ensuring compatibility with the rest of OpenStack with 3rd
party CI and the component and integration promotion pipelines [8]_, on an
ongoing bases.

Finally, changes in the underlying OS between branches means that for some
branches we maintain two "types" of CI job; for stable/train we have to support
both Centos 7 and Centos 8. With the coming stable/xena, we would likely have
to support Centos-Stream-8 as well as Centos-Stream-9 in the event that
Stream-9 is not fully available by the xena release, which further compounds
the resource burden. By adopting the proposal laid out here we can choose to
skip the Xena branch thus avoiding this increased CI and maintenance cost.

Proposed Change
===============

Overview
--------

The proposal is for all TripleO repos that are currently using the
cycle-with-intermediary release model to switch to independent. This will
allow us to choose to skip a particular release and more importantly skip
the creation of the given stable/branch on those repos.

This would allow the TripleO community to focus our resources on those branches
that are most 'important' to us, namely the 'FFU branches'. That is, the
branches that are part of the TripleO Fast Forward Upgrade chain (currently
these are Train -> Wallaby -> Z?). For example it is highly likely that we
would not create a Xena branch.

Developers will be freed from having to backport changes across stable/branches
and this will have a dramatic effect on our upstream CI resource consumption
and maintenance burden.

Alternatives
------------

We can continue to create all the stable/branches and use the same release
model we currently have. This would mean we would continue to have an increased
maintenance burden and would have to address that with increased resources.

Security Impact
---------------

None

Upgrade Impact
--------------

For upgrades it would mean that TripleO would no longer directly support all
OpenStack stable branches. So if we decide not to create stable/xena for example
then you cannot upgrade from wallaby to xena using TripleO. In some respects
this would more closely match reality since the focus of the active tripleo
developer community has typically been on ensuring the Fast Forward Upgrade
(e.g. train to wallaby) and less so on ensuring the point to point upgrade
between 2 branches.

Other End User Impact
---------------------

TripleO would no longer be able to deploy all versions of OpenStack. One idea
that was brough forth in the discussions around this topic thus far, is that
we can attempt to address this by designating a range of git tags as compatible
with a particular OpenStack stable branch.

For example if TripleO doesn't create a stable/xena, but during the xena cycle
makes releases for the various Tripleo repos then *those* releases will be
compatible for deploying OpenStack stable/xena. We can maintain and publicise
a set of compatible tags for each of the affected repos (e.g.,
tripleo-heat-templates versions 15.0.0 to 15.999.999 are compatible with
OpenStack stable/xena).

Some rules around tagging will help us. Generally we can keep doing what we
currently do with respect to tagging; For major.minor.patch (e.g. 15.1.1) in
the release tag, we will always bump major to signal a new stable branch.

One problem with this solution is that there is no place to backport fixes to.
For example if you are using tripleo-heat-templates 15.99.99 to deploy
OpenStack Xena (and there is no stable/xena for tht) then you'd have to apply
any fixes to the top of the 15.99.99 tag and use it. There would be no way
to commit these fixes into the code repo.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

There were concerns raised in the openstack-discuss thread [2] about RDO
packaging and how it would be affected by this proposal. As was discussed
there are no plans for RDO to stop building packages for any branch. For the
building of tripleo repos we would have to rely on the latest compatible
git tag, as outlined above in `Other End User Impact`_.

Developer Impact
----------------

Will have less stable/branches to backport fixes to. It is important to note
however that by skipping some branches, resulting backports across multiple
branches will result in a larger code diff and so be harder for developers to
implement. That is, there will be increased complexity in resulting backports if
we skip intermediate branches.

As noted in the `Other End User Impact`_ section above, for those branches that
tripleo decides not to create, there will be no place for developers to commit
any branch specific fixes to. They can consume particular tagged releases of
TripleO repos that are compatible with the given branch, but will not be able
to commit those changes to the upstream repo since the branch will not exist.

Implementation
==============

Assignee(s)
-----------

Wesley Hayutin <weshay@redhat.com>
Marios Andreou <marios@redhat.com>

Work Items
----------

Besides posting the review against the releases repo [9]_ we will need to
update documentation to reflect and inform about this change.

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

Yes we will at least need to add some section to the docs to explain this.
We may also add some landing page to show the currently 'active' or supported
TripleO branches.

References
==========

.. [1] `Tripleo IRC meeting logs 25 May 2021 <https://meetings.opendev.org/meetings/tripleo/2021/tripleo.2021-05-25-14.00.html>`_
.. [2] `openstack-discuss thread '[tripleo] Changing TripleO's release model' <http://lists.openstack.org/pipermail/openstack-discuss/2021-June/thread.html#22959>`_
.. [3] `TripleO section in governance projects.yaml <https://opendev.org/openstack/governance/src/commit/8dcac06d702ccff89b19c73b0c1d5ae7620b9a7b/reference/projects.yaml#L3044-L3177>`_
.. [4] `tripleo-heat-templates wallaby release file <https://opendev.org/openstack/releases/src/commit/e1b3fa10962cefad3220ae41e1c81a0ae0fd0fd5/deliverables/wallaby/tripleo-heat-templates.yaml#L3>`_
.. [5] `tripleo-upgrade independent release file <https://opendev.org/openstack/releases/src/commit/e1b3fa10962cefad3220ae41e1c81a0ae0fd0fd5/deliverables/_independent/tripleo-upgrade.yaml>`_
.. [6] `OpenStack project release models <https://releases.openstack.org/reference/release_models.html>`_
.. [7] `openstack-discuss [TripleO] moving stable/stein and stable/queens to End of Life <http://lists.openstack.org/pipermail/openstack-discuss/2021-June/023409.html>`_
.. [8] `TripleO Docs - TripleO CI Promotions <https://docs.openstack.org/tripleo-docs/latest/ci/stages-overview.html>`_
.. [9] `opendev.org openstack/releases git repo <https://opendev.org/openstack/releases/>`_
