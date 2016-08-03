=============================
TripleO Undercloud NTP Server
=============================

The Undercloud should provide NTP services for when external NTP services are
not available.

Problem Description
===================

NTP services are required to deploy with HA, but we rely on external services.
This means that TripleO can't be installed without Internet access or a local
NTP server.

This has several drawbacks:

* The NTP server is a potential point of failure, and it is an external
  dependency.

* Isolated deployments without Internet access are not possible without
  additional effort (manually deploying an NTP server).

* Infra CI is dependent on an external resource, leading to potential
  false negative test runs or CI failures.

Proposed Change
===============

Overview
--------

In order to address this problem, the Undercloud installation process should
include setting up an NTP server on the local Undercloud. The use of this
NTP server would be optional, but we may wish to make it a default. Having
a default is better than none, since HA deployments will fail without time
synchronization between the controller cluster members.

The operation of the NTP server on the Undercloud would be primarily of use
in small or proof-of-concept deployments. It is expected that sufficiently
large deployments will have an infrastructure NTP server already operating
locally.

Alternatives
------------

The alternative is to continue to require external NTP services, or to
require manual steps to set up a local NTP server.

Security Impact
---------------

Since the NTP server is required for syncing the HA, a skewed clock on one
controller (in relation to the other controllers) may make it ineligable to
participate in the HA cluster. If more than one controller's clock is skewed,
the entire cluster will fail to operate. This opens up an opportunity for
denial-of-service attacks against the cloud, either by causing NTP updates
to fail, or using a man-in-the-middle attack where deliberately false NTP
responses are returned to the controllers.

Of course, operating the NTP server on the Undercloud moves that attack
vector down to the Undercloud, so sufficient security hardening should be done
on the Undercloud and/or the attached networks. We may wish to bind the NTP
server only to the provisioning (control plane) network.

Other End User Impact
---------------------

This may make the life of the installer easier, since they don't need to open
a network connection to an NTP server or set up a local NTP server.

Performance Impact
------------------

The operation of the NTP server should have a negligible impact on Undercloud
performance. It is a lightweight protocol and the daemon requires little
resources.

Other Deployer Impact
---------------------

We now require that a valid NTP server be configured either in the templates
or on the deployment command-line. This requirement would be optional if we had
a default pointing to NTP services on the Undercloud.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------
Primary assignees:

* dsneddon@redhat.com
* bfournie@redhat.com

Work Items
----------

The TripleO Undercloud installation scripts will have to be modified to include
the installation and configuration of an NTP server. This will likely be done
using a composable service for the Undercloud, with configuration data taken
from undercloud.conf. The configuration should include a set of default NTP
servers which are reachable on the public Internet for when no servers are
specified in undercloud.conf.

Implement opening up iptables for NTP on the control plane network (bound to
only one IP/interface [ctlplane]  if possible).

Dependencies
============

The NTP server RPMs must be installed, and upstream NTP servers must be
identified (although we might configure a default such as pool.ntp.org)

Testing
=======

Since proper operation of the NTP services are required for successful
deployment of an HA overcloud, this functionality will be tested every time
a TripleO CI HA job is run.

We may also want to implement a validation that ensures that the NTP server
can reach its upstream stratum 1 servers. This will ensure that the NTP
server is serving up the correct time. This is optional, however, since the
only dependency is that the overcloud nodes agree on the time, not that it
be correct.

Documentation Impact
====================

The setup and configuration of the NTP server should be documented. Basic NTP
best practices should be communicated.

References
==========

* [1] - Administration Guide Draft/NTP - Fedora Project
  https://fedoraproject.org/wiki/Administration_Guide_Draft/NTP
