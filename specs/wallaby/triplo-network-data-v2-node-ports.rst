..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Network Data v2 - node ports and node network config
====================================================

With "Network Data v2" the goal is to move management of network resources
out of the heat stack. The schema spec [1]_ talked about the
``network_data.yaml`` format and managing networks, segments and subnets. This
spec follows up with node ports for composable networks and moving the node
network configuration action to the baremetal/network configuration workflow.


Problem description
===================

Applying a network change on day 2, currently requires a full stack update
since network resources such as ports are managed by heat. It has also been
problematic to create ports for large scale deployments; neutron on the single
node undercloud gets overwhelmed and it is difficult to throttle port creation
in Heat. As an early indication on the performance of port creation with the
proposed ansible module:

Performance stats: 100 nodes x 3 networks = 300 ports

.. code-block:: text

          4xCPU 1.8 GHz (8GB)             8x CPU 2.6 GHz (12GB)
          -------------------  --------------------------------
  Concurr:                 10          20         10          4
  ........     ..............   .........  .........  .........
  Create       real 5m58.006s   1m48.518s  1m51.998s  1m25.022s
  Delete:      real 4m12.812s   0m47.475s  0m48.956s  1m19.543s
  Re-run:      real 0m19.386s    0m4.389s   0m4.453s   0m4.977s


Proposed Change
===============

Extend the baremetal provisioning workflow that runs before overcloud
deployment to also create ports for composable networks. The baremetal
provisioning step already create ports for the provisioning network. Moving
the management of ports for composable networks to this workflow will
consolidate all port management into one workflow.

Also make baremetal provisioning workflow execute the tripleo-ansible
``tripleo_network_config`` role to configure node networking after
node provisioning.

The deploy workflow would be:

#. Operator defines composable networks in network data YAML file.
#. Operator provisions composable networks by running the
   ``openstack overcloud network provision`` command, providing the network
   data YAML file as input.
#. Operator defines roles and nodes in the baremetal deployment YAML file. This
   YAML also defines the networks for each role.
#. Operator deploys baremetal nodes by running the
   ``openstack overcloud node provision`` command. This step creates ports in
   neutron, and also configures networking; including composable networks; on
   the nodes using ansible role to apply network config with os-net-config
   [2]_.
#. Operator deploys heat stack including the environment files produced by the
   commands executed in the previous steps by running the
   ``openstack overcloud deploy`` command.
#. Operator executes config-download to install and configure openstack on the
   overcloud nodes. *(optional - only if overcloud deploy command executed with
   ``-stack-only``)*


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Harald Jensås <hjensas@redhat.com>

Approver(s)
-----------

Primary approver:
  TODO


Implementation Details
----------------------

The baremetal YAML definition will be extended, adding the ``networks`` and the
``network_config`` keys in role ``defaults`` as well as per-instance to support
``fixed_ip`` addressing, manually pre-created port resource and per-node
network configuration template.

The ``networks`` will replace the current ``nic`` key, until the ``nic`` key is
deprecated either can be used but not both at the same time. Networks in
``networks`` will support a boolean key ``vif`` which indicate if the port
should be attached in Ironic or not. If no network with ``vif: true`` is
specified an implicit one for the control plane will be appended:

.. code-block:: yaml

  - network: ctlplane
    vif: true

For networks with ``vif: true``, ports will be created by metalsmith. For
networks with ``vif: false`` (or ``vif`` not specified) the workflow will
create neutron ports based on the YAML definition.

The neutron ports will initially be tagged with the *stack name* and the
instance *hostname*, these tags are used for idempotency. The ansible module
managing ports will get all ports with the relevant tags and then add/remove
ports based on the expanded roles defined in the Baremetal YAML definition.
(The *hostname* and *stack_name* tags are also added to ports created with heat
in this tripleo-heat-templates change [4]_, to enable *adoption* of neutron
ports created by heat for the upgrade scenario.)

Additionally the ports will be tagged with the ironic node uuid when this is
available. Full set of tags are shown in the example below.

.. code-block:: json

   {
     "port": {
       "name": "controller-1-External",
       "tags": ["tripleo_ironic_uuid=<IRONIC_NODE_UUID>",
                "tripleo_hostname=controller-1",
                "tripleo_stack_name=overcloud"],
     }
   }

.. Note:: In deployments where baremetal nodes have multiple physical NIC's
          multiple networks can have ``vif: true``, so that VIF attach
          in ironic and proper neutron port binding happens. In a scenario
          where neutron on the Undercloud is managing the switch this would
          enable automation of the Top-of-Rack switch configuration.

Mapping of the port data for overcloud nodes will go into a ``NodePortMap``
parameter in tripleo-heat-tempaltes. The map will contain submaps for each
node, keyed by the node name. Initially the ``NodePortMap`` will be consumed by
alternative *fake-port*
``OS::TripleO::{{role.name}}::Ports::{{network.name}}Port`` resource templates.
In the final implementation the environment file created can be extended and
the entire ``OS::TripleO::{{role.name}}`` resource can be replaced with a
template that references parameter in the generated environment directly, i.e a
re-implemented ``puppet/role.role.j2.yaml`` without the server and port
resources. The ``NodePortMap`` will be added to the
*overcloud-baremetal-deployed.yaml* created by the workflow creating the
overcloud node port resources.

Network ports for ``vif: false`` networks, will be managed by a new ansible
module ``tripleo_overcloud_network_ports``, the input for this role will be a
list of instance definitions as generated by the
``tripleo_baremetal_expand_roles`` ansible module. The
``tripleo_baremetal_expand_roles`` ansible module will be extended to add
network/subnet information from the baremetal deployment YAML definition.

The baremetal provision workflow will be extended to write a ansible inventory,
we should try extend tripleo-ansible-inventory so that the baremetal
provisioning workflow can re-use existing code to create the inventory.
The inventory will be used to configure networking on the provisioned nodes
using the **triple-ansible** ``tripleo_network_config`` ansible role.


Already Deployed Servers
~~~~~~~~~~~~~~~~~~~~~~~~

The Baremetal YAML definition will be used to describe the **pre-deployed**
servers baremetal deployment. In this scenario there is no Ironic node to
update, no ironic UUID to add to a port's tags and no ironic node to attach
VIFs to.

All ports, including the ctlplane port will be managed by the
``tripleo_overcloud_network_ports`` ansible module. The Baremetal YAML
definition for a deployment with pre-deployed servers will have to include an
``instance`` entry for each pre-deployed server. This entry will have the
``managed`` key set to ``false``.

It should be possible for an already deployed server to have a management
address that is completely separate from the tripleo managed addreses. The
Baremetal YAML definition can be extended to carry a ``management_ip`` field
for this purpose. In the case no managment address is available the ctlplane
network entry for pre-deployed instances must have ``fixed_ip`` configured.

The deployment workflow will *short circuit* the baremetal provisioning of
``managed: false`` instances. The Baremetal YAML definition can define a
mix of *already deployed server* instances, and instances that should be
provisioned via metalsmith. See :ref:`baremetal_yaml_pre_provsioned`.

YAML Examples
~~~~~~~~~~~~~

Example: Baremetal YAML definition with defaults properties
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  - name: Controller
    count: 1
    hostname_format: controller-%index%
    defaults:
      profile: control
      network_config:
        template: templates/multiple_nics/multiple_nics.j2
        physical_bridge_name: br-ex
        public_interface_name: nic1
        network_deployment_actions: ['CREATE']
        net_config_data_lookup: {}
      networks:
        - network: ctlplane
          vif: true
        - network: external
          subnet: external_subnet
        - network: internal_api
          subnet: internal_api_subnet
        - network: storage
          subnet: storage_subnet
        - network: storage_mgmt
          subnet: storage_mgmt_subnet
        - network: Tenant
          subnet: tenant_subnet
  - name: Compute
    count: 1
    hostname_format: compute-%index%
    defaults:
      profile: compute
      network_config:
        template: templates/multiple_nics/multiple_nics.j2
        physical_bridge_name: br-ex
        public_interface_name: nic1
        network_deployment_actions: ['CREATE']
        net_config_data_lookup: {}
      networks:
        - network: ctlplane
          vif: true
        - network: internal_api
          subnet: internal_api_subnet
        - network: tenant
          subnet: tenant_subnet
        - network: storage
          subnet: storage_subnet

Example: Baremetal YAML definition with per-instance overrides
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  - name: Controller
    count: 1
    hostname_format: controller-%index%
    defaults:
      profile: control
      network_config:
        template: templates/multiple_nics/multiple_nics.j2
        physical_bridge_name: br-ex
        public_interface_name: nic1
        network_deployment_actions: ['CREATE']
        net_config_data_lookup: {}
        bond_interface_ovs_options:
      networks:
        - network: ctlplane
          vif: true
        - network: external
          subnet: external_subnet
        - network: internal_api
          subnet: internal_api_subnet
        - network: storage
          subnet: storage_subnet
        - network: storage_mgmt
          subnet: storage_mgmt_subnet
        - network: tenant
          subnet: tenant_subnet
    instances:
      - hostname: controller-0
        name: node00
        networks:
          - network: ctlplane
            vif: true
          - network: internal_api:
            fixed_ip: 172.21.11.100
      - hostname: controller-1
        name: node01
        networks:
          External:
            port: controller-1-external
      - hostname: controller-2
        name: node02
  - name: ComputeLeaf1
    count: 1
    hostname_format: compute-leaf1-%index%
    defaults:
      profile: compute-leaf1
      networks:
        - network: internal_api
          subnet: internal_api_subnet
        - network: tenant
          subnet: tenant_subnet
        - network: storage
          subnet: storage_subnet
    instances:
      - hostname: compute-leaf1-0
        name: node03
        network_config:
          template: templates/multiple_nics/multiple_nics_dpdk.j2
          physical_bridge_name: br-ex
          public_interface_name: nic1
          network_deployment_actions: ['CREATE']
          net_config_data_lookup: {}
          num_dpdk_interface_rx_queues: 1
        networks:
          - network: ctlplane
            vif: true
          - network: internal_api
            fixed_ip: 172.21.12.105
          - network: tenant
            port: compute-leaf1-0-tenant
          - network: storage
            subnet: storage_subnet


.. _baremetal_yaml_pre_provsioned:

Example: Baremetal YAML for Already Deployed Servers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  - name: Controller
    count: 3
    hostname_format: controller-%index%
    defaults:
      profile: control
      network_config:
        template: templates/multiple_nics/multiple_nics.j2
      networks:
        - network: ctlplane
        - network: external
          subnet: external_subnet
        - network: internal_api
          subnet: internal_api_subnet
        - network: storage
          subnet: storage_subnet
        - network: storage_mgmt
          subnet: storage_mgmt_subnet
        - network: tenant
          subnet: tenant_subnet
      managed: false
    instances:
      - hostname: controller-0
        networks:
          - network: ctlplane
            fixed_ip: 192.168.24.10
      - hostname: controller-1
        networks:
          - network: ctlplane
            fixed_ip: 192.168.24.11
      - hostname: controller-2
        networks:
          - network: ctlplane
            fixed_ip: 192.168.24.12
  - name: Compute
    count: 2
    hostname_format: compute-%index%
    defaults:
      profile: compute
      network_config:
        template: templates/multiple_nics/multiple_nics.j2
      networks:
        - network: ctlplane
        - network: internal_api
          subnet: internal_api_subnet
        - network: tenant
          subnet: tenant_subnet
        - network: storage
          subnet: storage_subnet
    instances:
      - hostname: compute-0
        managed: false
        networks:
          - network: ctlplane
            fixed_ip: 192.168.24.100
      - hostname: compute-1
        managed: false
        networks:
          - network: ctlplane
            fixed_ip: 192.168.24.101

Example: NodeNetworkDataMappings
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  NodePortMap:
    controller-0:
      ctlplane:
        ip_address: 192.168.24.9 (2001:DB8:24::9)
        ip_subnet: 192.168.24.9/24 (2001:DB8:24::9/64)
        ip_address_uri: 192.168.24.9 ([2001:DB8:24::9])
      internal_api:
        ip_address: 172.18.0.9 (2001:DB8:18::9)
        ip_subnet: 172.18.0.9/24 (2001:DB8:18::9/64)
        ip_address_uri: 172.18.0.9 ([2001:DB8:18::9])
      tenant:
        ip_address: 172.19.0.9 (2001:DB8:19::9)
        ip_subnet: 172.19.0.9/24 (2001:DB8:19::9/64)
        ip_address_uri: 172.19.0.9 ([2001:DB8:19::9])
    compute-0:
      ctlplane:
        ip_address: 192.168.24.15 (2001:DB8:24::15)
        ip_subnet: 192.168.24.15/24 (2001:DB8:24::15/64)
        ip_address_uri: 192.168.24.15 ([2001:DB8:24::15])
      internal_api:
        ip_address: 172.18.0.15 (2001:DB8:18::1)
        ip_subnet: 172.18.0.15/24 (2001:DB8:18::1/64)
        ip_address_uri: 172.18.0.15 ([2001:DB8:18::1])
      tenant:
        ip_address: 172.19.0.15 (2001:DB8:19::15)
        ip_subnet: 172.19.0.15/24 (2001:DB8:19::15/64)
        ip_address_uri: 172.19.0.15 ([2001:DB8:19::15])

Example: Ansible inventory
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

  Controller:
    vars:
      role_networks:
        - External
        - InternalApi
        - Tenant
      role_networks_lower:
        External: external
        InternalApi: internal_api
        Tenant: tenant
      networks_all:
        - External
        - InternalApi
        - Tenant
      neutron_physical_bridge_name: br-ex
      neutron_public_interface_name: nic1
      tripleo_network_config_os_net_config_mappings: {}
      network_deployment_actions: ['CREATE', 'UPDATE']
      ctlplane_subnet_cidr: 24
      ctlplane_mtu: 1500
      ctlplane_gateway_ip: 192.168.24.254
      ctlplane_dns_nameservers: []
      dns_search_domains: []
      ctlplane_host_routes: {}
      internal_api_cidr: 24
      internal_api_gateway_ip: 172.18.0.254
      internal_api_host_routes: []
      internal_api_mtu: 1500
      internal_api_vlan_id: 20
      tenant_cidr: 24
      tenant_api_gateway_ip: 172.19.0.254
      tenant_host_routes: []
      tenant_mtu: 1500
    hosts:
      controller-0:
        ansible_host: 192.168.24.9
        ctlplane_ip: 192.168.24.9
        internal_api_ip: 172.18.0.9
        tenant_ip: 172.19.0.9
  Compute:
    vars:
      role_networks:
        - InternalApi
        - Tenant
      role_networks_lower:
        InternalApi: internal_api
        Tenant: tenant
      networks_all:
        - External
        - InternalApi
        - Tenant
      neutron_physical_bridge_name: br-ex
      neutron_public_interface_name: nic1
      tripleo_network_config_os_net_config_mappings: {}
      network_deployment_actions: ['CREATE', 'UPDATE']
      ctlplane_subnet_cidr: 24
      ctlplane_mtu: 1500
      ctlplane_gateway_ip: 192.168.25.254
      ctlplane_dns_nameservers: []
      dns_search_domains: []
      ctlplane_host_routes: {}
      internal_api_cidr: 24
      internal_api_gateway_ip: 172.18.1.254
      internal_api_host_routes: []
      internal_api_mtu: 1500
      internal_api_vlan_id: 20
      tenant_cidr: 24
      tenant_api_gateway_ip: 172.19.1.254
      tenant_host_routes: []
      tenant_mtu: 1500
    hosts:
      compute-0:
        ansible_host: 192.168.25.15
        ctlplane_ip: 192.168.25.15
        internal_ip: 172.18.1.15
        tenant_ip: 172.19.1.15


TODO
----

* Constraint validation, for example ``BondInterfaceOvsOptions`` uses
  ``allowed_pattern: ^((?!balance.tcp).)*$`` to ensure balance-tcp bond mode is
  not used, as it is known to cause packet loss.

Work Items
----------

#. Write ansible inventory after baremetal provisioning

   Create an ansible inventory, similar to the inventory created by config-
   download. The ansible inventory is required to apply network
   configuration to the deployed nodes.

   We should try to extend tripleo-ansible-inventory so that the baremetal
   provisioning workflow can re-use existing code to create the inventory.

   It is likely that it makes sense for the workflow to also run the
   tripleo-ansible role tripleo_create_admin to create the *tripleo-admin*
   ansible user.

#. Extend baremetal provisioning workflow to create neutron ports and
   update the ironic node ``extra`` field with the ``tripleo_networks`` map.

#. The baremetal provisioning workflow needs a *pre-deployed-server* option
   that cause it to not deploy baremetal nodes, only create network ports.
   When this option is used the baremetal deployment YAML file will also
   describe the already provisioned nodes.

#. Apply and validate network configuration using the **triple-ansible**
   ``tripleo_network_config`` ansible role. This step will be integrated in
   the provisioning command.

#. Disable and remove management of composable network ports in
   tripleo-heat-templates.

#. Change the Undercloud and Standalone deploy to apply network configuration
   prior to the creating the ephemeral heat stack using the
   ``tripleo_network_config`` ansible role.

Testing
=======

Multinode OVB CI job's with network-isolation will be updated to test the new
workflow.

Upgrade Impact
==============

During upgrade switching to use network ports managed outside of the heat stack
the ``PortDeletionPolicy`` must be set to ``retain`` during the update/upgrade
*prepare* step, so that the existing neutron ports (which will be adopted by
the pre-heat port management workflow) are not deleted when running the update/
upgrade *converge* step.

Moving node network configuration out of tripleo-heat-templates will require
manual (or scripted) migration of settings controlled by heat template
parameters to the input file used for baremetal/network provisioning. At least
the following parameters are affected:

* NeutronPhysicalBridge
* NeutronPublicInterface
* NetConfigDataLookup
* NetworkDeploymentActions

Parameters that will be deprecated:

* NetworkConfigWithAnsible
* {{role.name}}NetworkConfigTemplate
* NetworkDeploymentActions
* {{role.name}}NetworkDeploymentActions
* BondInterfaceOvsOptions
* NumDpdkInterfaceRxQueues
* {{role.name}}LocalMtu
* NetConfigDataLookup
* DnsServers
* DnsSearchDomains
* ControlPlaneSubnetCidr
* HypervisorNeutronPublicInterface
* HypervisorNeutronPhysicalBridge

The environment files used to select one of the pre-defined nic config
templates will no longer work. The template to use must be set in the YAML
defining the baremetal/network deployment. This affect the following
environment files:

* environments/net-2-linux-bonds-with-vlans.j2.yaml
* environments/net-bond-with-vlans.j2.yaml
* environments/net-bond-with-vlans-no-external.j2.yaml
* environments/net-dpdkbond-with-vlans.j2.yaml
* environments/net-multiple-nics.j2.yaml
* environments/net-multiple-nics-vlans.j2.yaml
* environments/net-noop.j2.yaml
* environments/net-single-nic-linux-bridge-with-vlans.j2.yaml
* environments/net-single-nic-with-vlans.j2.yaml
* environments/net-single-nic-with-vlans-no-external.j2.yaml

Documentation Impact
====================

The documentation effort is **heavy** and will need to be incrementally
updated. As a minumum, a separate page explaining the new process must be
created.

The TripleO docs will need updates in many sections, including:

* `TripleO OpenStack Deployment
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/install_overcloud.html>`_
* `Provisioning Baremetal Before Overcloud Deploy
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/provisioning/baremetal_provision.html#>`_
* `Deploying with Custom Networks
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/custom_networks.html>`_
* `Configuring Network Isolation
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/network_isolation.html>`_
* `Deploying Overcloud with L3 routed networking
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/routed_spine_leaf_network.html>`_


Alternatives
============

#. **Not changing how ports are created**

   In this case we keep creating the ports with heat, the do nothing
   alternative.

#. **Create a completely separate workflow for composable network ports**

   A separate workflow that can run before/after node provisioning. It can read
   the same YAML format as baremetal provisioning, or it can have it's own YAML
   format.

   The problem with this approach is that we loose the possibility to store
   relations between neutron-port and baremetal node in a database. As in, we'd
   need our own database (a file) maintaining the relationships.

   .. Note:: We need to implement this workflow anyway for a pre-deployed
             server scenario, but instead of a completely separate workflow
             the baremetal deploy workflow can take an option to not
             provision nodes.

#. **Create ports in ironic and bind neutron ports**

   Instead of creating ports unknown to ironic, create ports for the ironic
   nodes in the baremetal service.

   The issue is that ironic does not have a concept of virtual port's, so we
   would have to either add this support in ironic, switch TripleO to use
   neutron trunk ports or create *fake* ironic ports that don't actually
   reflect NICs on the baremetal node. (This abandoned ironic spec [3]_ discuss
   one approach for virtual port support, but it was abandoned in favor of
   neutron trunk ports.)

   With each PTG there is a re-occurring suggestion to replace neutron with a
   more light weight IPAM solution. However, the effort to actually integrate
   it properly with ironic and neutron for composable networks probably isn't
   time well spent.


References
==========

.. [1] `Review: Spec for network data v2 format <https://review.opendev.org/752437>`_.
.. [2] `os-net-config <https://opendev.org/openstack/os-net-config>`_.
.. [3] `Abandoned spec for VLAN Aware Baremetal Instances <https://review.opendev.org/277853>`_.
.. [4] `Review: Add hostname and stack_name tags to ports <https://review.opendev.org/761845>`_.
