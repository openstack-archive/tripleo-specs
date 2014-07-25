..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Unit Testing TripleO Projects
==========================================

https://blueprints.launchpad.net/tripleo/unit-testing

We should enable more unit testing in TripleO projects to allow better test
coverage of code paths not included in CI, make it easier for reviewers
to verify that a code change does what it is supposed to, and avoid wasting
reviewer and developer time resolving style issues.

Problem Description
===================

Right now there is very little unit testing of the code in most of the TripleO
projects.  This has a few negative effects:

- We have no test coverage of any code that isn't included in our CI runs.

- For the code that is included in CI runs, we don't actually know how much
  of that code is being tested.  There may be many code branches that are not
  used during a CI run.

- We have no way to test code changes in isolation, which makes it slower to
  iterate on them.

- Changes not covered by CI are either not tested at all or must be manually
  tested by reviewers, which is tedious and error-prone.

- Major refactorings frequently break less commonly used interfaces to tools
  because those interfaces are not tested.

Additionally, because there are few/no hacking-style checks in the TripleO
projects, many patches get -1'd for style issues that could be caught by
an automated tool.  This causes unnecessary delay in merging changes.

Proposed Change
===============

I would like to build out a unit testing framework that simplifies the
process of unit testing in TripleO.  Once that is done, we should start
requiring unit tests for new and changed features like the other OpenStack
projects do.  At that point we can also begin adding test coverage for
existing code.

The current plan is to make use of Python unit testing libraries to be as
consistent as possible with the rest of OpenStack and make use of the test
infrastructure that already exists.  This will reduce the amount of new code
required and make it easier for developers to begin writing unit tests.

For style checking, the dib-lint tool has already been created to catch
common errors in image elements.  More rules should be added to it as we
find problems that can be automatically found.  It should also be applied
to the tripleo-image-elements project.

The bash8 project also provides some general style checks that would be
useful in TripleO, so we should begin making use of it as well.  We should
also contribute additional checks when possible and provide feedback on any
checks we disagree with.

Any unit tests added should be able to run in parallel.  This both speeds up
testing and helps find race bugs.

Alternatives
------------

Shell unit testing
^^^^^^^^^^^^^^^^^^
Because of the quantity of bash code used in TripleO, we may want to
investigate using a shell unit test framework in addition to Python.  I
think this can be revisited once we are further along in the process and
have a better understanding of how difficult it will be to unit test our
scripts with Python.  I still think we should start with Python for the
reasons above and only add other options if we find something that Python
unit tests can't satisfy.

One possible benefit of a shell-specific unit testing framework is that it
could provide test coverage stats so we know exactly what code is and isn't
being tested.

If we determine that a shell unit test framework is needed, we should try
to choose a widely-used one with well-understood workflows to ease adoption.

Sandboxing
^^^^^^^^^^
I have done some initial experimentation with using fakeroot/fakechroot to
sandbox scripts that expect to have access to the root filesystem.  I was
able to run a script that writes to root-owned files as a regular user, making
it think it was writing to the real files, but I haven't gotten this working
with tox for running unit tests that way.

Another option would be to use real chroots.  This would provide isolation
and is probably more common than fakeroots.  The drawback would be that
chrooting requires root access on the host machine, so running the unit tests
would as well.

Security Impact
---------------

Many scripts in elements assume they will be running as root.  We obviously
don't want to do that in unit tests, so we need a way to sandbox those scripts
to allow them to run but not affect the test system's root filesystem.

Other End User Impact
---------------------

None

Performance Impact
------------------

Adding more tests will increase the amount of time Jenkins gate jobs take.
This should have minimal real impact though, because unit tests should run
in significantly less time than the integration tests.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Developers will need to implement unit tests for their code changes, which
will require learning the unit testing tools we adopt.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    bnemec

goneri has begun some work to enable dib-lint in tripleo-image-elements

Work Items
----------

* Provide and document a good Python framework for testing the behavior of
  bash scripts.  Use existing functionality in upstream projects where
  possible, and contribute new features when necessary.

* Gate tripleo-image-elements on dib-lint, which will require fixing any
  lint failures currently in tripleo-image-elements.

* Enable bash8 in the projects with a lot of bash scripts.

* Add unit-testing to tripleo-incubator to enable verification of things
  like ``devtest.sh --build-only``.

* Add a template validation test job to triple-heat-templates.

Dependencies
============

* bash8 will be a new test dependency.

Testing
=======

These changes should leverage the existing test infrastructure as much as
possible, so the only thing needed to enable the new tests would be changes
to the infra config for the affected projects.

Documentation Impact
====================

None of this work should be user-visible, but we may need developer
documentation to help with writing unit tests.


References
==========

bash8: http://git.openstack.org/cgit/openstack-dev/bash8/

There are some notes related to this spec at the bottom of the Summit
etherpad: https://etherpad.openstack.org/p/juno-summit-tripleo-ci
