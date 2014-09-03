..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Backwards compatibility and TripleO
==========================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-backwards-compat

TripleO has run with good but not perfect backwards compatibility since
creation. It's time to formalise this in a documentable and testable fashion.

TripleO will follow Semantic Versioning (aka semver_) for versioning all
releases. We will strive to avoid breaking backwards compatibility at all, and
if we have to it will be because of extenuating circumstances such as security
fixes with no other way to fix things.

Problem Description
===================

TripleO has historically run with an unspoken backwards compatibility policy
but we now have too many people making changes - we need to build a single
clear policy or else our contributors will have to rework things when one
reviewer asks for backwards compat when they thought it was not needed (or vice
versa do the work to be backwards compatible when it isn't needed.

Secondly, because we haven't marked any of our projects as 1.0.0 there is no
way for users or developers to tell when and where backwards compatibility is
needed / appropriate.

Proposed Change
===============

Adopt the following high level heuristics for identifying backwards
incompatible changes:

* Making changes that break user code that scripts or uses a public interface.

* Becoming unable to install something we could previously.

* Being unable to install something because someone else has altered things -
  e.g. being unable to install F20 if it no longer exists on the internet
  is not an incompatible change - if it were returned to the net, we'd be able
  to install it again. If we remove the code to support this thing, then we're
  making an incompatible change. The one exception here is unsupported
  projects - e.g. unsupported releases of OpenStack, or Fedora, or Ubuntu.
  Because unsupported releases are security issues, and we expect most of our
  dependencies to do releases, and stop supporting things, we will not treat
  cleaning up code only needed to support such an unsupported release as
  backwards compatible. For instance, breaking the ability to deploy a previous
  *still supported* OpenStack release where we had previously been able to
  deploy it is a backwards incompatible change, but breaking the ability to
  deploy an *unsupported* OpenStack release is not.

Corollaries to these principles:

* Breaking a public API (network or Python). The public API of a project is
  any released API (e.g. not explicitly marked alpha/beta/rc) in a version that
  is >= 1.0.0. For Python projects, a \_ prefix marks a namespace as non-public
  e.g. in ``foo.\_bar.quux`` ``quux`` is not public because it's in a non-public
  namespace. For our projects that accept environment variables, if the
  variable is documented (in the README.md/user documentation) then the variable
  is part of the public interface. Otherwise it is not.

* Increasing the set of required parameters to Heat templates. This breaks
  scripts that use TripleO to deploy. Note that adding new parameters which
  need to be set when deploying *new* things is fine because the user is
  doing more than just pulling in updated code.

* Decreasing the set of accepted parameters to Heat templates. Likewise, this
  breaks scripts using the Heat templates to do deploys. If the parameters are
  no longer accepted because they are for no longer supported versions of
  OpenStack then that is covered by the carve-out above.

* Increasing the required metadata to use an element except when both Tuskar
  and tripleo-heat-templates have been updated to use it. There is a
  bi-directional dependency from t-i-e to t-h-t and back - when we change
  signals in the templates we have to update t-i-e first, and when we change
  parameters to elements we have to alter t-h-t first. We could choose to make
  t-h-t and t-i-e completely independent, but don't believe that is a sensible
  use of time - they are closely connected, even though loosely coupled.
  Instead we're treating them a single unit: at any point in time t-h-t can
  only guarantee to deploy images built from some minimum version of t-i-e,
  and t-i-e can only guarantee to be deployed with some minimum version of
  t-h-t. The public API here is t-h-t's parameters, and the link to t-i-e
  is equivalent to the dependency on a helper library for a Python
  library/program: requiring new minor versions of the helper library is not
  generally considered to be an API break of the calling code. Upgrades will
  still work with this constraint - machines will get a new image at the same
  time as new metadata, with a rebuild in the middle. Downgrades / rollback
  may require switching to an older template at the same time, but that was
  already the case.

* Decreasing the accepted metadata for an element if that would result in an
  error or misbehaviour.

Other sorts of changes may also be backwards incompatible, and if identified
will be treated as such - that is, this list is not comprehensive.

We don't consider the internal structure of Heat templates to be an API, nor
any test code within the TripleO codebases (whether it may appear to be public
or not).

TripleO's incubator is not released and has no backwards compatibility
guarantees - but a point in time incubator snapshot interacts with ongoing
releases of other components - and they will be following semver, which means
that a user wanting stability can get that as long as they don't change the
incubator.

TripleO will promote all its component projects to 1.0 within one OpenStack
release cycle of them being created. Projects may not become dependencies of a
project with a 1.0 or greater version until they are at 1.0 themselves. This
restriction serves to prevent version locking (makes upgrades impossible) by
the depending version, or breakage (breaks users) if the pre 1.0 project breaks
compatibility. Adding new projects will involve creating test jobs that test
the desired interactions before the dependency is added, so that the API can
be validated before the new project has reached 1.0.

Adopt the following rule on *when* we are willing to [deliberately] break
backwards compatibility:

* When all known uses of the code are for no longer supported OpenStack
  releases.

* If the PTL signs off on the break. E.g. a high impact security fix for which
   we cannot figure out a backwards compatible way to deliver it to our users
   and distributors.

We also need to:

* Set a timeline for new codebases to become mature (one cycle). Existing
  codebases will have the clock start when this specification is approved.

* Set rules for allowing anyone to depend on new codebases (codebase must be
  1.0.0).

* Document what backwards compatible means in the context of heat templates and
  elements.

* Add an explicit test job for deploying Icehouse from trunk, because that will
  tell us about our ability to deploy currently supported OpenStack versions
  which we could previously deploy - that failing would indicate the proposed
  patch is backwards incompatible.

* If needed either fix Icehouse, or take a consensus decision to exclude
  Icehouse support from this policy.

* Commit to preserving backwards compatibility.

* When we need alternate codepaths to support backwards compatibility we will
  mark them clearly to facilitate future cleanup::

    # Backwards compatibility: <....>
    if ..
        # Trunk
        ...
    elif
        # Icehouse
        ...
    else
        # Havana
        ...

Alternatives
------------

* We could say that we don't do backwards compatibility and release like the
  OpenStack API services do, but this makes working with us really difficult
  and it also forces folk with stable support desires to work from separate
  branches rather than being able to collaborate on a single codebase.

* We could treat tripleo-heat-templates and tripleo-image-elements separately
  to the individual components and run them under different rules - e.g. using
  stable branches rather than semver. But there have been so few times that
  backwards compatibility would be hard for us that this doesn't seem worth
  doing.

Security Impact
---------------

Keeping code around longer may have security considerations, but this is a
well known interaction.

Other End User Impact
---------------------

End users will love us.

Performance Impact
------------------

None anticipated. Images will be a marginally larger due to carrying backwards
compat code around.

Other Deployer Impact
---------------------

Deployers will appreciate not having to rework things. Not that they have had
to, but still.

Developer Impact
----------------

Developers will have clear expectations set about backwards compatibility which
will help them avoid being asked to rework things. They and reviewers will need
to look out for backward incompatible changes and special case handling of
them to deliver the compatibility we aspire to.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lifeless

Other contributors:

Work Items
----------

* Draft this spec.

* Get consensus around it.

* Release all our non-incubator projects as 1.0.0.

* Add Icehouse deploy test job. (Because we could install Icehouse at the start
  of Juno, and if we get in fast we can keep being able to do so).

Dependencies
============

None. An argument could be made for doing a quick cleanup of stuff, but the
reality is that it's not such a burden we've had to clean it up yet.

Testing
=======

To ensure we don't accidentally break backwards compatibility we should look
at the oslo cross-project matrix eventually - e.g. run os-refresh-config
against older releases of os-apply-config to ensure we're not breaking
compatibility. Our general policy of building releases of things and using
those goes a long way to giving us good confidence though - we can be fairly
sure of no single-step regressions (but will still have to watch out for
N-step regressions unless some mechanism is put in place).

Documentation Impact
====================

The users manual and developer guides should reflect this.

References
==========

.. _semver: http://docs.openstack.org/developer/pbr/semver.html
