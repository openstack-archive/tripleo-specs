..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Network Data format/schema (v2)
===============================

The network data schema (``network_data.yaml``) used to define composable
networks in TripleO has had several additions since it was first introduced.
Due to legacy compatibility some additions make the schema somewhat non-
intuitive. Such as adding support for routed networks, where the ``subnets``
map was introduced.

The goal of this spec is to get discussion and settle on a new network data
(v2) format that will be used once management of network resources such
as networks, segments and subnets are moved out of the heat stack.

Problem description
===================

The current schema is somewhat inconsistent, and not as precice as it could
be. For example the ``base`` subnet being at level-0, while additional
subnets are in the ``subnets`` map. It would be more intuitive to define
all subnets in the ``subnets`` map.

Currently the network resource properties are configured via a mix of
parameters in the heat environment and network data. For example
``dns_domain``, ``admin_state_up``, ``enable_dhcp``, ``ipv6_address_mode``,
``ipv6_ra_mode`` and ``shared`` properties are configured via Heat parameters,
while other properties such as ``cidr``, ``gateway_ip``, ``host_routes`` etc.
is defined in network data.

Proposed Change
===============

Overview
--------

Change the network data format so that all network properties are managed in
network data, so that network resources can be managed outside of the heat
stack.

.. note:: Network data v2 format will only be used with the new tooling that
          will manage networks outside of the heat stack.

Network data v2 format should stay compatible with tripleo-heat-templates
jinja2 rendering outside of the ``OS::TripleO::Network`` resource and it's
subresources ``OS::TripleO::Network::{{network.name}}``.

User Experience
^^^^^^^^^^^^^^^

Tooling will be provided for user's to export the network information from
an existing deployment. This tooling will output a network data file in
v2 format, which from then on can be used to manage the network resources
using tripleoclient commands or tripleo-ansible cli playbooks.

The command line tool to manage the network resources will output the
environment file that must be included when deploying the heat stack. (Similar
to the environment file produced when provisioning baremetal nodes without
nova.)

CLI Commands
^^^^^^^^^^^^

Command to export provisioned overcloud network information to network data v2
format.

.. code-block:: shell

    openstack overcloud network export \
      --stack <stack_name> \
      --output <network_data_v2.yaml>

Command to create/update overcloud networks outside of heat.

.. code-block:: shell

    openstack overcloud network provision \
      --networks-file <network_data_v2.yaml> \
      --output <network_environment.yaml>


Main difference between current network data schema and the v2 schema proposed
here:

* Base subnet is moved to the ``subnets`` map, aligning configuration for
  non-routed and routed deploymends (spine-and-leaf, DCN/Edge)
* The ``enabled`` (bool) is no longer used. Disabled networks should be
  excluded from the file, removed or commented.
* The ``compat_name`` option is no longer required. This was used to change
  the name of the heat resource internally. Since the heat resource will be a
  thing of the past with network data v2, we don't need it.
* The keys ``ip_subnet``, ``gateway_ip``, ``allocation_pools``, ``routes``,
  ``ipv6_subnet``, ``gateway_ipv6``, ``ipv6_allocation_pools`` and
  ``routes_ipv6`` are no longer valid at the network level.
* New key ``physical_network``, our current physical_network names for base and
  non-base segments are not quite compatible. Adding logic in code to
  compensate is complex. (This field may come in handy when creating ironic
  ports in metalsmith as well.)
* New keys ``network_type`` and ``segmentation_id`` since we could have users
  that used ``{{network.name}}NetValueSpecs`` to set network_type vlan.

.. note:: The new tooling should validate that non of the keys previously
          valid in network data v1 are used in network data v2.

Example network data v2 file for IPv4
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    - name: Storage
      name_lower: storage                     (optional, default: name.lower())
      admin_state_up: false                   (optional, default: false)
      dns_domain: storage.localdomain.        (optional, default: undef)
      mtu: 1442                               (optional, default: 1500)
      shared: false                           (optional, default: false)
      service_net_map_replace: storage        (optional, default: undef)
      ipv6: true                              (optional, default: false)
      vip: true                               (optional, default: false)
      subnets:
        subnet01:
          ip_subnet: 172.18.1.0/24
          gateway_ip: 172.18.1.254            (optional, default: undef)
          allocation_pools:                   (optional, default: [])
            - start: 172.18.1.10
              end: 172.18.1.250
          enable_dhcp: false                  (optional, default: false)
          routes:                             (optional, default: [])
            - destination: 172.18.0.0/24
              nexthop: 172.18.1.254
          vlan: 21                            (optional, default: undef)
          physical_network: storage_subnet01  (optional, default: {{name.lower}}_{{subnet name}})
          network_type: flat                  (optional, default: flat)
          segmentation_id: 21                 (optional, default: undef)
        subnet02:
          ip_subnet: 172.18.0.0/24
          gateway_ip: 172.18.0.254            (optional, default: undef)
          allocation_pools:                   (optional, default: [])
            - start: 172.18.0.10
              end: 172.18.0.250
          enable_dhcp: false                  (optional, default: false)
          routes:                             (optional, default: [])
            - destination: 172.18.1.0/24
              nexthop: 172.18.0.254
          vlan: 20                            (optional, default: undef)
          physical_network: storage_subnet02  (optional, default: {{name.lower}}_{{subnet name}})
          network_type: flat                  (optional, default: flat)
          segmentation_id: 20                 (optional, default: undef)

Example network data v2 file for IPv6
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    - name: Storage
      name_lower: storage
      admin_state_up: false
      dns_domain: storage.localdomain.
      mtu: 1442
      shared: false
      vip: true
      subnets:
        subnet01:
          ipv6_subnet: 2001:db8:a::/64
          gateway_ipv6: 2001:db8:a::1
          ipv6_allocation_pools:
            - start: 2001:db8:a::0010
              end: 2001:db8:a::fff9
          enable_dhcp: false
          routes_ipv6:
            - destination: 2001:db8:b::/64
              nexthop: 2001:db8:a::1
          ipv6_address_mode: null
          ipv6_ra_mode: null
          vlan: 21
          physical_network: storage_subnet01  (optional, default: {{name.lower}}_{{subnet name}})
          network_type: flat                  (optional, default: flat)
          segmentation_id: 21                 (optional, default: undef)
        subnet02:
          ipv6_subnet: 2001:db8:b::/64
          gateway_ipv6: 2001:db8:b::1
          ipv6_allocation_pools:
            - start: 2001:db8:b::0010
              end: 2001:db8:b::fff9
          enable_dhcp: false
          routes_ipv6:
            - destination: 2001:db8:a::/64
              nexthop: 2001:db8:b::1
          ipv6_address_mode: null
          ipv6_ra_mode: null
          vlan: 20
          physical_network: storage_subnet02  (optional, default: {{name.lower}}_{{subnet name}})
          network_type: flat                  (optional, default: flat)
          segmentation_id: 20                 (optional, default: undef)

Example network data v2 file for dual stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Dual IPv4/IPv6 with two subnets per-segment, one for IPv4 and the other for
IPv6. A single neutron port with an IP address in each subnet can be created.

In this case ``ipv6`` key will control weather services are configured to
bind to IPv6 or IPv4. (default ipv6: false)

.. code-block:: yaml

    - name: Storage
      name_lower: storage
      admin_state_up: false
      dns_domain: storage.localdomain.
      mtu: 1442
      shared: false
      ipv6: true                            (default ipv6: false)
      vip: true
      subnets:
        subnet01:
          ip_subnet: 172.18.1.0/24
          gateway_ip: 172.18.1.254
          allocation_pools:
            - start: 172.18.1.10
              end: 172.18.1.250
          routes:
            - destination: 172.18.0.0/24
              nexthop: 172.18.1.254
          ipv6_subnet: 2001:db8:a::/64
          gateway_ipv6: 2001:db8:a::1
          ipv6_allocation_pools:
            - start: 2001:db8:a::0010
              end: 2001:db8:a::fff9
          routes_ipv6:
            - destination: 2001:db8:b::/64
              nexthop: 2001:db8:a::1
          vlan: 21
        subnet02:
          ip_subnet: 172.18.0.0/24
          gateway_ip: 172.18.0.254
          allocation_pools:
            - start: 172.18.0.10
              end: 172.18.0.250
          routes:
            - destination: 172.18.1.0/24
              nexthop: 172.18.0.254
          ipv6_subnet: 2001:db8:b::/64
          gateway_ipv6: 2001:db8:b::1
          ipv6_allocation_pools:
            - start: 2001:db8:b::0010
              end: 2001:db8:b::fff9
          routes_ipv6:
            - destination: 2001:db8:a::/64
              nexthop: 2001:db8:b::1
          vlan: 20

Alternatives
------------

#. Not changing the network data format

   In this case we need an alternative to provide the values for resource
   properties currently managed using heat parameters, when moving
   management of the network resources outside the heat stack.

#. Only add new keys for properties

   Keep the concept of the ``base`` subnet at level-0, and only add keys
   for properties currently managed using heat parameters.


Security Impact
===============

N/A


Upgrade Impact
==============

When (if) we remove the capability to manage network resources in the
overcloud heat stack, the user must run the export command to generate
a new network data v2 file. Use this file as input to the ``openstack
overcloud network provision`` command, to generate the environment file
required for heat stack without network resources.


Performance Impact
==================

N/A


Documentation Impact
====================

The network data v2 format must be documented. Procedures to use the commands
to export network information from existing deployments as well as
procedures to provision/update/adopt network resources with the non-heat stack
tooling must be provided.

Heat parameters which will be deprecated/removed:

* ``{{network.name}}NetValueSpecs``: Deprecated, Removed.
  This was used to set ``provider:physical_network`` and
  ``provider:network_type``, or actually **any** network property.
* ``{network.name}}NetShared``: Deprecated, replaced by network level
  ``shared`` (bool)
* ``{{network.name}}NetAdminStateUp``: Deprecated, replaced by network
  level ``admin_state_up`` (bool)
* ``{{network.name}}NetEnableDHCP``: Deprecated, replaced by subnet
  level ``enable_dhcp`` (bool)
* ``IPv6AddressMode``: Deprecated, replaced by subnet level
  ``ipv6_address_mode``
* ``IPv6RAMode``: Deprecated, replaced by subnet level ``ipv6_ra_mode``

Once deployed_networks.yaml (https://review.opendev.org/751876) is used the
following parameters are Deprecated, since they will no longer be used:

* {{network.name}}NetCidr
* {{network.name}}SubnetName
* {{network.name}}Network
* {{network.name}}AllocationPools
* {{network.name}}Routes
* {{network.name}}SubnetCidr_{{subnet}}
* {{network.name}}AllocationPools_{{subnet}}
* {{network.name}}Routes_{{subnet}}


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * Harald Jensås


Work Items
----------

* Add tags to resources using heat stack - https://review.opendev.org/750666
* Tools to extract provisioned networks from existing deployment
  https://review.opendev.org/750671, https://review.opendev.org/750672
* New tooling to provision/update/adopt networks
  https://review.opendev.org/751739, https://review.opendev.org/751875
* Deployed networks template in THT - https://review.opendev.org/751876
