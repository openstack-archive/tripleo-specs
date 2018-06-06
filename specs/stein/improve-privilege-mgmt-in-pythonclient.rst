..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Improve privilege mgmt in Python Client Code
============================================

https://blueprints.launchpad.net/tripleo/+spec/improve-privilege-mgmt-in-tripleo

Currently python-tripleoclient doesn't properly manage privilege escalation,
and requires "sudo" calls either inside its code, or when calling the scripts.

In other python library and script, we also encounter this kind of issues.

This ends up with either NOPASSWD:ALL in sudoers, or in an endless list of
rights we can't maintain in a convenient way.

This discussion has started on an etherpad ([0]_)


Problem Description
===================

We do not want to keep on that track - it's neither a good practice nor secure.

* Customers might have concerns with such loose rights management on their
  systems

* System operators might want to try to reduce the rights and break the whole
  thing

* Documentation is lacking regarding the usage of such rights, and does not
  address security concerns


Proposed Change
===============

Overview
--------

The Oslo library proposes an interesting approach with two complementary tools:

* oslo.rootwrap ([1]_)

* oslo.privsep ([2]_)

They allow to get a simple sudoer file, while ensuring with complex filter
support the intended use only is allowed.

Integrating the privsep library will require a huge code refactoring, but this
is for the best, as it will ensure we have a proper, common way to do privilege
escalation. Converging the code is the best way to get a clean, maintainable,
understandable code.

More over, as it seems the use of privsep is quiet new, we still can be
involved in the best practices establishment. There are already a couple of
blog posts covering them, and it would be a good thing to follow them, and if
needed, make them better suited.

Alternatives
------------

* Do Nothing: If we don't address this, end users are impacted as the
  current implementation will be rejected by security compliance
  standards such as ANSSI, DoD STIG etc

* Use "plain" sudo calls in the python script. This can of course be a real
  alternative, but probably not the smartest, as rootwrap allows a finer
  control over the context and command allowed. More over, it does not
  depend on the user running the script - so we have only one entry in sudo
  per user being able to run the python code.

Security Impact
---------------

* Will improve security, only allowed actions will be running with elevated
  privileges.

* Will allow to get a common way to make privilege escalations across the
  projects.

* Might push good practices in python devel regarding privilege escalation and
  its issues.

Other End User Impact
---------------------

None

Performance Impact
------------------

The use of rootwrap should not cause any performance impact.

Other Deployer Impact
---------------------

Modify the sudoers in order to allow calls to rootwrap only.

Developer Impact
----------------

* The code will need a refactoring in order to integrate oslo.privsep.
  In addition new features will need to follow the (good) practices in place in
  order to ensure code readability and overall comprehension.

* Following existing examples, all functions/configuration for privsep should be
  in a dedicated directory, and we should avoid aliasing the imports.

* When adding a new function needing privilege escalation, an update to the
  rootwrap configuration must be done

* Proper testing must be added in order to ensure privileges are really used
  and correct


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lhinds@redhat.com

Other contributors:
  cjeanner@redhat.com
  jaosorior@redhat.com

Work Items
----------

1. List all the calls in the libraries and scripts. This step is critical, as it
   will require a proper understanding of the code, as well as a proper
   understanding of "who IS" running, and "who SHOULD" run parts of the
   functions/calls.

2. Refactor the code, especially following the proposed practice ([3]_).

3. Proper testing must be added, following the already existing examples.

4. Documentation must be provided for other developpers in order to ensure they
   correctly implement the privilege escalation in their patch.


Dependencies
============

* oslo.privsep

Testing
=======

Unit and functionality tests must be updated in order to take into account the
privilege escalation, and ensure it is actually working as expected.

Further tests should be done in order to ensure the allowed scope in rootwrap
isn't too wide.

Documentation Impact
====================

Documentation about privsep usage must be provided.

References
==========

.. [0] https://etherpad.openstack.org/p/tripleo-heat-admin-security
.. [1] https://docs.openstack.org/oslo.rootwrap/latest/
.. [2] https://docs.openstack.org/oslo.privsep/latest/
.. [3] https://www.madebymikal.com/adding-oslo-privsep-to-a-new-project-a-worked-example/
