..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
TripleO LLDP Validation
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/tripleo-lldp-validation

The Link Layer Discovery Protocol (LLDP) is a vendor-neutral link layer
protocol in the Internet Protocol Suite used by network devices for
advertising their identity, capabilities, and neighbors on an
IEEE 802 local area network, principally wired Ethernet. [1]

The Link Layer Discover Protocol (LLDP) helps identify layer 1/2
connections between hosts and switches. The switch port, chassis ID,
VLANs trunked, and other info is available, for planning or
troubleshooting a deployment. For instance, a deployer may validate
that the proper VLANs are supplied on a link, or that all hosts
are connected to the Provisioning network.

Problem Description
===================

A detailed description of the problem:

* Deployment networking is one of the most difficult parts of any
  OpenStack deployment. A single misconfigured port or loose cable
  can derail an entire multi-rack deployment.

* Given the first point, we should work to automate validation and
  troubleshooting where possible.

* Work is underway to collect LLDP data in ironic-python-agent,
  and we have an opportunity to make that data useful [2].


Proposed Change
===============

Overview
--------

The goal is to expose LLDP data that is collected during
introspection, and provide this data in a format that is useful for the
deployer. This work depends on the LLDP collection work being done
in ironic-python-agent [3].

There is work being done to implement LLDP data collection for Ironic/
Neutron integration. Although this work is primarily focused on features
for bare-metal Ironic instances, there will be some overlap with the
way TripleO uses Ironic to provision overcloud servers.

Alternatives
------------

There are many network management utilities that use CDP or LLDP data to
validate the physical networking. Some of these are open source, but none
are integrated with OpenStack.

Alternative approaches that do not use LLDP are typically vendor-specific
and require specific hardware support. Cumulus has a solution which works
with multiple vendors' hardware, but that solution requires running their
custom OS on the Ethernet switches.

Another approach which is common is to perform collection of the switch
configurations to a central location, where port configurations can be
viewed, or in some cases even altered and remotely pushed. The problem
with this approach is that the switch configurations are hardware and
vendor-specific, and typically a network engineer is required to read
and interpret the configuration. A unified approach that works for all
common switch vendors is preferred, along with a unified reporting format.

Security Impact
---------------

The physical network report provides a roadmap to the underlying network
structure. This could prove handy to an attacker who was unaware of the
existing topology. On the other hand, the information about physical
network topology is less valuable than information about logical topology
to an attacker. LLDP contains some information about both physical and
logical topology, but the logical topology is limited to VLAN IDs.

The network topology report should be considered sensitive but not
critical. No credentials or shared secrets are revealed in the data
collected by ironic-inspector.

Other End User Impact
---------------------

This report will hopefully reduce the troubleshooting time for nodes
with failed network deployments.

Performance Impact
------------------

If this report is produced as part of the ironic-inspector workflow,
then it will increase the time taken to introspect each node by a
negligible amount, perhaps a few seconds.

If this report is called by the operator on demand, it will have
no performance impact on other components.

Other Deployer Impact
---------------------

Deployers may want additional information than the per-node LLDP report.
There may be some use in providing aggregate reports, such as the number
of nodes with a specific configuration of interfaces and trunked VLANs.
This would help to highlight outliers or misconfigured nodes.

There have been discussions about adding automated switch configuration
in TripleO. This would be a mechanism whereby deployers could produce the
Ethernet switch configuration with a script based on a configuration
template. The deployer would provide specifics like the number of nodes
and the configuration per node, and the script would generate the switch
configuration to match. In that case, the LLDP collection and analysis
would function as a validator for the automatically generated switch
port configurations.

Developer Impact
----------------

The initial work will be to fill in fixed fields such as Chassis ID
and switch port. An LLDP packet can contain additional data on a
per-vendor basis, however.

The long-term plan is to store the entire LLDP packet in the
metadata. This will have to be parsed out. We may have to work with
switch vendors to figure out how to interpret some of the data if
we want to make full use of it.

Implementation
==============

Some notes about implementation:

* This Python tool will access the introspection data and produce
  reports on various information such as VLANs per port, host-to-port
  mapping, and MACs per host.

* The introspection data can be retrieved with the Ironic API [4] [5].

* The data will initially be a set of fixed fields which are retrievable
  in the JSON in the Ironic introspection data. Later, the entire
  LLDP packet will be stored, and will need to be parsed outisde of the
  Ironic API.

* Although the initial implementation can return a human-readable report,
  other outputs should be available for automation, such as YAML.

* The tool that produces the LLDP report should be able to return data
  on a single host, or return all of the data.

* Some basic support for searching would be a nice feature to have.

* This data will eventually be used by the GUI to display as a validation
  step in the deployment workflow.

Assignee(s)
-----------

Primary assignee:
  dsneddon <dsneddon@redhat.com>

Other contributors:
  bfournie <bfournie@redhat.com>

Work Items
----------

* Create the Python script to grab introspection data from Swift using
  the API.

* Create the Python code to extract the relevant LLDP data from the
  data JSON.

* Implement per-node reports

* Implement aggregate reports

* Interface with UI developers to give them the data in a form that can
  be consumed and presented by the TripleO UI.

* In the future, when the entire LLDP packet is stored, refactor logic
  to take this into account.

Testing
=======

Since this is a report that is supposed to benefit the operator, perhaps
the best way to include it in CI is to make sure that the report gets
logged by the Undercloud. Then the report can be reviewed in the log
output from the CI run.

In fact, this might benefit the TripleO CI process, since hardware issues
on the network would be easier to troubleshoot without having access to
the bare metal console.


Documentation Impact
====================

Documentation will need to be written to cover making use of the new
LLDP reporting tool. This should cover running the tool by hand and
interpreting the data.


References
==========
* [1] - Wikipedia entry on LLDP:
  https://en.wikipedia.org/wiki/Link_Layer_Discovery_Protocol

* [2] - Blueprint for Ironic/Neutron integration:
  https://blueprints.launchpad.net/ironic/+spec/ironic-ml2-integration

* [3] - Review: Support LLDP data as part of interfaces in inventory
  https://review.openstack.org/#/c/320584/

* [4] - Accessing Ironic Introspection Data
  http://tripleo.org/advanced_deployment/introspection_data.html

* [5] - Ironic API - Get Introspection Data
  http://docs.openstack.org/developer/ironic-inspector/http-api.html#get-introspection-data