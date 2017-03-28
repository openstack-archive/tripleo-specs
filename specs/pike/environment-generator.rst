..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Sample Environment Generator
============================

A common tool to generate sample Heat environment files would be beneficial
in two main ways:

* Consistent formatting and details.  Every environment file would include
  parameter descriptions, types, defaults, etc.

* Ease of updating.  The parameters can be dynamically read from the templates
  which allows the sample environments to be updated automatically when
  parameters are added or changed.

Problem Description
===================

Currently our sample environments are hand written, with no consistency in
terms of what is included.  Most do not include a description of what all
the parameters do, and almost none include the types of the parameters or the
default values for them.

In addition, the environment files often get out of date because developers
have to remember to manually update them any time they make a change to the
parameters for a given feature or service.  This is tedious and error-prone.

The lack of consistency in environment files is also a problem for the UI,
which wants to use details from environments to improve the user experience.
When environments are created manually, these details are likely to be missed.

Proposed Change
===============

Overview
--------

A new tool, similar to the oslo.config generator, will allow us to eliminate
these problems.  It will take some basic information about the environment and
use the parameter definitions in the templates to generate the sample
environment file.

The resulting environments should contain the following information:

* Human-readable Title
* Description
* parameter_defaults describing all the available parameters for the
  environment
* Optional resource_registry with any necessary entries

Initially the title and description will simply be comments, but eventually we
would like to get support for those fields into Heat itself so they can be
top-level keys.

Ideally the tool would be able to update the capabilities map automatically as
well.  At some point there may be some refactoring done there to eliminate the
overlap, but during the transition period this will be useful.

This is also a good opportunity to impose some organization on the environments
directory of tripleo-heat-templates.  Currently it is mostly a flat directory
that contains all of the possible environments.  It would be good to add
subdirectories that group related environments so they are easier to find.

The non-generated environments will either be replaced by generated ones,
when that makes sense, or deprecated in favor of a generated environment.
In the latter case the old environments will be left for a cycle to allow
users transition time to the new environments.

Alternatives
------------

We could add more checks to the yaml-validate tool to ensure environment files
contain the required information, but this still requires more developer
time and doesn't solve the maintenance problems as parameters change.

Security Impact
---------------

None

Other End User Impact
---------------------

Users should get an improved deployment experience through more complete and
better documented sample environments.  Existing users who are referencing
the existing sample environments may need to switch to the new generated
environments.

Performance Impact
------------------

No runtime performance impact.  Initial testing suggests that it may take a
non-trivial amount of time to generate all of the environments, but it's not
something developers should have to do often.

Other Deployer Impact
---------------------

See End User Impact

Developer Impact
----------------

Developers will need to write an entry in the input file for the tool rather
than directly writing sample environments.  The input format of the tool will
be documented, so this should not be too difficult.

When an existing environment is deprecated in favor of a generated one, a
release note should be written by the developer making the change in order to
communicate it to users.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bnemec

Other contributors:
  jtomasek

Work Items
----------

* Update the proposed tool to reflect the latest design decisions
* Convert existing environments to be generated


Dependencies
============

No immediate dependencies, but in the long run we would like to have some
added functionality from Heat to allow these environments to be more easily
consumed by the UI.  However, it was agreed at the PTG that we would proceed
with this work and make the Heat changes in parallel so we can get some of
the benefits of the change as soon as possible.


Testing
=======

Any environments used in CI should be generated with the tool.  We will want
to add a job that exercises the tool as well, probably a job that ensures any
changes in the patch under test are reflected in the environment files.


Documentation Impact
====================

We will need to document the format of the input file.


References
==========

`Initial proposed version of the tool
<https://review.openstack.org/#/c/253638/>`_

https://etherpad.openstack.org/p/tripleo-environment-generator
