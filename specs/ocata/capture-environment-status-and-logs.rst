..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Tool to Capture Environment Status and Logs
===========================================

https://blueprints.launchpad.net/tripleo/+spec/capture-environment-status-and-logs

To aid in troubleshooting, debugging, and reproducing issues we should create
or integrate with a tool that allows an operator or developer to collect and
generage a single bundle that provides the state and history of a deployed
environment.

Problem Description
===================

Currently there is no single command that can be run via either the
tripleoclient or via the UI that will generage a single artifact to be used
to report issues when failures occur.

* tripleo-quickstart_, tripleo-ci_ and operators collect the logs for bug
  reports in different ways.

* When a failure occurs, many different peices of information must be collected
  to be able to understand where the failure occured. If the logs required are
  not asked for, an operator may not know to what to provide for
  troubleshooting.


Proposed Change
===============

Overview
--------

TripleO should provide a unified method for collecting status and logs from the
undercloud and overcloud nodes.  The tripleoclient should support executing a
workflow to run status and log collection processes via sosreport_. The output
of the sosreport_ should be collected and bundled together in a single location.

Alternatives
------------

Currently, various shell scripts and ansible tasks are used by the CI processes
to perform log collection. These scripts are not maintained in combination with
the core TripleO and may require additional artifacts that are not installed by
default with a TripleO environment.

tripleo-quickstart_ uses ansible-role-tripleo-collect-logs_ to collect logs.

tripleo-ci_ uses bash scripts to collect the logs.

Fuel uses timmy_.

Security Impact
---------------

The logs and status information may be considered sensitive information. The
process to trigger status and logs should require authentication. Additionally
we should provide a basic password protection mechanism for the bundle of logs
that is created by this process.

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

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  alex-schultz


Work Items
----------

* Ensure OpenStack `sosreport plugins`_ are current.
* Write a TripleO sosreport plugin.
* Write a `Mistral workflow`_ to execute sosreport and collect artifacts.
* Write python-tripleoclient_ integration to execute Mistral workflows.
* Update documentation and CI scripts to leverage new collection method.


Dependencies
============

None.

Testing
=======

As part of CI testing, the new tool should be used to collect environment logs.

Documentation Impact
====================

Documentation should be updated to reflect the standard ways to collect the logs
using the tripleo client.

References
==========

.. _ansible-role-tripleo-collect-logs: https://github.com/redhat-openstack/ansible-role-tripleo-collect-logs
.. _Mistral workflow: http://docs.openstack.org/developer/mistral/terminology/workflows.html
.. _python-tripleoclient: https://github.com/openstack/python-tripleoclient
.. _tripleo-ci: https://github.com/openstack-infra/tripleo-ci
.. _tripleo-quickstart: https://github.com/openstack/tripleo-quickstart
.. _sosreport: https://github.com/sosreport/sos
.. _sosreport plugins: https://github.com/sosreport/sos/tree/master/sos/plugins
.. _timmy: https://github.com/openstack/timmy
