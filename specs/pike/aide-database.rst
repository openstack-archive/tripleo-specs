..
  This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
AIDE - Intrustion Detection Database
====================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-aide-database

AIDE (Advanced Intrusion Detection Environment) is a file and directory
integrity verification system. It computes a checksum of object
attributes, which are then stored into a database. Operators can then
run periodic checks against the current state of defined objects and
verify if any attributes have been changed (thereby suggesting possible
malicious / unauthorised tampering).

Problem Description
===================

Security Frameworks such as DISA STIG [1] / CIS [3] require that AIDE be
installed and configured on all Linux systems.

To enable OpenStack operators to comply with the aforementioned security
requirements, they require a method of automating the installation of
AIDE and initialization of AIDE's integrity Database. They also require
a means to perform a periodic integrity verification run.

Proposed Change
===============

Overview
--------

Introduce a puppet-module to manage the AIDE service and ensure the AIDE
application is installed, create rule entries and a CRON job to allow
a periodic check of the AIDE database or templates to allow monitoring
via Sensu checks as part of OpTools.

Create a tripleo-heat-template service to allow population of hiera data
to be consumed by the puppet-module managing AIDE.

The proposed puppet-module is lhinds-aide [2] as this module will accept
rules declared in hiera data, initialize the Database and enables CRON
entries. Other puppet AIDE modules were missing hiera functionality or
other features (such as CRON population).

Within tripleo-heat-templates, a composable service will be created to
feed a rule hash into the AIDE puppet module as follows:

    AIDERules:
        description: Mapping of AIDE config rules
        type: json
        default: {}

The Operator can then source an environment file and provide rule
information as a hash:

    parameter_defaults:
      AIDERules:
          'Monitor /etc for changes':
            content: '/etc p+sha256'
            order  : 1
          'Monitor /boot for changes':
            content: '/boot p+u+g+a'
            order  : 2

Ops Tool Integration
--------------------

In order to allow active monitoring of AIDE events, a sensu check can
be created to perform an interval based verification of AIDE monitored
files (set using ``AIDERules``) against the last initialized database.

Results of the Sensu activated AIDE verification checks will then be fed
to the sensu server for alerting and archiving.

The Sensu clients (all overcloud nodes) will be configured with a
standalone/passive check via puppet-sensu module which is already
installed on overcloud image.

If the Operator should choose not to use OpTools, then they can still
configure AIDE using the traditional method by means of a CRON entry.

Alternatives
------------

Using a puppet-module coupled with a TripleO service is the most
pragmatic approach to populating AIDE rules and managing the AIDE
service.

Security Impact
---------------

AIDE is an integrity checking application and therefore requires
Operators insure the security of AIDE's database is protected from
tampering. Should an attacker get access to the database, they could
attempt to hide malicious activity by removing records of file integrity
hashes.

The default location is currently `/var/lib/aide/$database` which
puppet-aide sets with privileges of `0600` and ownership of
`root \ root`.

AIDE itself introduces no security impact to any OpenStack projects
and has no interaction with any OpenStack services.

Other End User Impact
---------------------

The service interaction will occur via heat templates and the TripleO
UI (should a capability map be present).

Performance Impact
------------------

No Performance Impact

Other Deployer Impact
---------------------

The service will be utlised by means of an environment file. Therefore,
should a deployer not reference the environment template using the
`openstack overcloud deploy -e` flag, there will be no impact.

Developer Impact
----------------

No impact on other OpenStack Developers.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lhinds

Work Items
----------

1. Add puppet-aide [1] to RDO as a puppet package

2. Create TripleO Service for AIDE

3. Create Capability Map

4. Create CI Job

5. Submit documentation to tripleo-docs.


Dependencies
============

Dependency on lhinds-aide Puppet Module.

Testing
=======

Will be tested in TripleO CI by adding the service and an environment
template to a TripleO CI scenario.

Documentation Impact
====================

Documentation patches will be made to explain how to use the service.

References
==========

Original Launchpad issue: https://bugs.launchpad.net/tripleo/+bug/1665031

[1] https://www.stigviewer.com/stig/red_hat_enterprise_linux_6/2016-07-22/finding/V-38489

[2] https://forge.puppet.com/lhinds/aide

[3]
file:///home/luke/project-files/tripleo-security-hardening/CIS_Red_Hat_Enterprise_Linux_7_Benchmark_v2.1.0.pdf

[3]
file:///home/luke/project-files/tripleo-security-hardening/CIS_Red_Hat_Enterprise_Linux_7_Benchmark_v2.1.0.pdf
