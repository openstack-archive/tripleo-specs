..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Control mechanism for os-apply-config
=====================================

Problem Description
===================

We require a control mechanism in os-apply-config (oac). This could be used,
for example, to:

 * Not create an empty target
 * Set permissions on the target

Proposed Change
===============

The basic proposal is to parameterise oac with maps (aka dictionaries)
containing control data. These maps will be supplied as YAML in companion
control files. Each file will be named after the template it relates to, with a
".oac" suffix. For example, the file "abc/foo.sh" would be controlled by
"abc/foo.sh.oac".

Only control files with matching templates files will be respected, IE the file
"foo" must exist for the control file "foo.oac" to have any effect. A dib-lint
check will be added to look for file control files without matching templates,
as this may indicate a template has been moved without its control file.

Directories may also have control files. In this case, the control file must be
inside the directory and be named exactly "oac". A file either named "oac" or
with the control file suffix ".oac" will never be considered as templates.

The YAML in the control file must evaluate to nothing or a mapping. The former
allows for the whole mapping having been commented out. The presence of
unrecognised keys in the mapping is an error. File and directory control keys
are distinct but may share names. If they do, they should also share similar
semantics.

Example control file::

    key1: true
    key2: 0700
    # comment
    key3:
      - 1
      - 2

To make the design concrete, one file control key will be offered initially:
allow_empty. This expects a Boolean value and defaults to true. If it is true,
oac will behave as it does today. Otherwise, if after substitutions the
template body is empty, no file will be created at the target path and any
existing file there will be deleted.

allow_empty will also be allowed as a directory control key. Again, it will
expect a Boolean value and default to true. Given a nested structure
"A/B/C/foo", where "foo" is an empty file with allow_empty=false:

 * C has allow_empty=false: A/B/ is created, C is not.
 * B has allow_empty=false: A/B/C/ is created.
 * B and C have allow_empty=false: Only A/ is created.

It is expected that additional keys will be proposed soon after this spec is
approved.

Alternatives
------------

A fenced header could be used rather than a separate control file. Although
this aids visibility of the control data, it is less consistent with control
files for directories and (should they be added later) symlinks.

The directory control file name has been the subject of some debate.
Alternatives to control "foo/" include:

 * foo/.oac (not visible with unmodified "ls")
 * foo/oac.control (longer)
 * foo/control (generic)
 * foo.oac (if foo/ is empty, can't be stored in git)
 * foo/foo.oac (masks control file for foo/foo)

Security Impact
---------------

None. The user is already in full control of the target environment. For
example, they could use the allow_empty key to delete a critical file. However
they could already simply provide a bash script to do the same. Further, the
resulting image will be running on their (or their customer's) hardware, so it
would be their own foot they'd be shooting.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

It will no longer be possible to create files named "oac" or with the suffix
".oac" using oac. This will not affect any elements currently within
diskimage-builder or tripleo-image-elements.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  alexisl (aka lxsli, Alexis Lee)

Other contributors:
  None

Work Items
----------

 * Support file control files in oac
 * Support the allow_empty file control key
 * Add dib-lint check for detached control files
 * Support directory control files in oac
 * Support the allow_empty directory control key
 * Update the oac README

Dependencies
============

None.

Testing
=======

This change is easily tested using standard unit test techniques.

Documentation Impact
====================

The oac README must be updated.

References
==========

There has already been some significant discussion of this feature:
    https://blueprints.launchpad.net/tripleo/+spec/oac-header

There is a bug open for which an oac control mechanism would be useful:
    https://bugs.launchpad.net/os-apply-config/+bug/1258351
