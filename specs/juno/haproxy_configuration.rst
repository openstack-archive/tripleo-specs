..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Haproxy ports and related services configuration
================================================

Blueprint: https://blueprints.launchpad.net/tripleo/+spec/tripleo-haproxy-configuration

Current spec provides options for HA endpoints delivery via haproxy.


Problem Description
===================

Current tripleo deployment scheme binds services on 0.0.0.0:standard_port,
with stunnel configured to listen on ssl ports.

This configuration has some drawbacks and wont work in ha, for several reasons:

* haproxy cant bind on <vip_address>:<service_port> - openstack services are
  bound to 0.0.0.0:<service_port>

* services ports hardcoded in many places (any_service.conf, init-keystone),
  so changing them and configuring from heat would be a lot of pain

* the non-ssl endpoint is reachable from outside the local host,
  which could potentially confuse users and expose them to an insecure connection


Proposed Change
===============

The most convenient setup in my opinion, and also fits nicely in current code
(requires minor refactoring), is to bind haproxy, stunnel (ssl), openstack services
on same ports with different ipaddress settings.

In case of using stunnel:

vip_address = 192.0.2.21
node_address = 192.0.2.24

1. haproxy
   bind vip_address:80
   bind vip_address:443

   listen horizon
   server node_1 node_address:80

2. stunnel
   accept node_address:80
   connect 127.0.0.1:80

3. horizon
   bind 127.0.0.1:80


Alternatives
------------

There are several alternatives which do not cover all the requirements for
security or extensibility

Option 1: Assignment of different ports for haproxy, stunnel, openstack services on 0.0.0.0

* requires additional firewall configuration
* security issue with non-ssl services endpoints

1. haproxy
   bind :80

   listen horizon
   server node_1 node_address:8800

2. stunnel
   accept :8800
   connect :8880

3. horizon
   bind :8880

Option 2: Using only haproxy ssl termination is suboptimal:

* 1.5 is still in devel phase -> potential stability issues
* we would have to get this into supported distros
* this also means that there is no SSL between haproxy and real service
* security issue with non-ssl services endpoints

1. haproxy
   bind vip_address:80

   listen horizon
   server node_1 node_address:80

2. horizon
   bind node_address:80

Option 3: Add additional ssl termination before load-balancer

* not useful in current configuration because load balancer (haproxy)
  and openstack services installed on same nodes

Security Impact
---------------

* Only ssl protected endpoints are publicly available
* Minimal firewall configuration
* Not forwarding decrypted traffic over non-localhost connections
* compromise of a control node exposes all external traffic (future and possibly past)
  to decryption and/or spoofing

Other End User Impact
---------------------

Several services will listen on same port, but it will be quite easy
to understand if user (operator) will know some context.


Performance Impact
------------------

No differences between approaches.

Other Deployer Impact
---------------------
None

Developer Impact
----------------
None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dshulyak


Work Items
----------

tripleo-incubator:
* build overcloud-control image with haproxy element

tripleo-image-elements:

* openstack-ssl element refactoring

* refactor services configs to listen on 127.0.0.1:
  horizon apache configuration, glance, nova, cinder, swift, ceilometer,
  neutron, heat, keystone, trove

tripleo-heat-templates:
* add haproxy metadata to heat-templates


Dependencies
============
None


Testing
=======
CI testing dependencies:

* use vip endpoints in overcloud scripts

* add haproxy element to overcloud-control image (maybe with stats enabled) before
  adding haproxy related metadata to heat templates


Documentation Impact
====================

* update incubator manual

* update elements README.md


References
==========

http://haproxy.1wt.eu/download/1.4/doc/configuration.txt

https://www.stunnel.org/howto.html
