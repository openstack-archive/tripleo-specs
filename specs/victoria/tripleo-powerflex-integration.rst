..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================================
Enable TripleO to deploy Dell EMC PowerFlex software defined storage via Ansible
================================================================================

Problem description
===================

There is currently no automated way to deploy VxFlexOS from within TripleO.
Goal is to provide an ease of use at the time of deployment as well as during
lifecycle operations.

Proposed changes
================

Overview
--------
VxFlexOS has been rebranded to PowerFlex.

The deployer experience to stand up PowerFlex with TripleO should be the
following:

The deployer chooses to deploy a role containing any of the PowerFlex services:
PowerflexMDM, PowerflexLIA, PowerflexSDS and PowerflexSDC.

At least three new Overcloud roles should be defined such as:
- Controller with PowerFlex
- Compute with PowerFlex
- Storage with PowerFlex

Custom roles definition are used to define which service will run on which
type of nodes. We'll use this custom roles_data.yaml to deploy the overcloud.

PowerFlex support for HCI, which combines compute and storage into a single
node, has been considered but will not be part of the first drop.

The deployer provides the PowerFlex parameters as offered today in a Heat env
file.

The deployer starts the deployment and gets an overcloud with PowerFlex and
appropriate services deployed on each node per its role.
Current code is available here. Still WIP.

https://github.com/dell/tripleo-powerflex

The following files are created in
/usr/share/openstack-tripleo-heat-templates/deployment/powerflex-ansible :
- powerflex-base.yaml
- powerflex-lia.yaml
- powerflex-mdm.yaml
- powerflex-sdc.yaml
- powerflex-sds.yaml
All of these files are responsible of the configuration of each sevice. Each
service is based upon the powerflex-base.yaml template which calls the Ansible
playbook and triggers the deployment.

The directory /usr/share/powerflex-ansible holds the Ansible playbook which
installs and configure PowerFlex.

A new tripleo-ansible role is created in /usr/share/ansible/roles called
tripleo-powerflex-run-ansible which prepares the variables and triggers the
execution of the PowerFlex Ansible playbook.

An environment name powerflex-ansible.yaml file is created in
/usr/share/openstack-tripleo-heat-emplates/environments/powerflex-ansible
and defines the resource registry mapping and additional parameters required by
the PowerFlex Ansible playbook.

Ports which have to be opened are managed by TripleO.

PowerFlex deployment with TripleO Ansible
-----------------------------------------
Proposal to create a TripleO Ansible playbook to deploy a PowerFlex system.

We refer to a PowerFlex system as a set of services deployed on nodes on a
per-role basis.

The playbook described here assumes the following:

A deployer chooses to deploy PowerFlex and includes the following Overcloud
roles which installs the PowerFlex services based upon the mapping found in
THT's roles_data.yaml:

| Role       | Associated PowerFlex service             |
| ---------- | ---------------------------------------- |
| Controller | PowerflexMDM, PowerflexLIA, PowerflexSDC |
| Compute    | PowerflexLIA, PowerflexSDC               |
| Storage    | PowerflexLIA, PowerflexSDS               |

The deployer chooses to include new Heat environment files which will be in THT
when this spec is implemented. An environment file will change the
implementation of any of the four services from the previous step.

A new Ansible playbook is called during the deployment which triggers the
execution of the appropriate PowerFlex Ansible playbook.

This can be identified as an cascading-ansible deployment.

A separate Ansible playbook will be created for each goal described below:

- Initial deployment of OpenStack and PowerFlex
- Update and upgrade PowerFlex SW
- Scaling up or down DayN operations

This proposal only refers to a single PowerFlex system deployment.

RPMS/Kernel dependencies
------------------------

Virt-Customize will be used to inject the rpms into the overcloud-full-image for
new installations.


Version dependencies
--------------------

Version control is handled outside current proposal. The staging area has the
PowerFlex packages specific to the OS version of overcloud image.

Ansible playbook
=================

Initial deployment of OpenStack and PowerFlex
---------------------------------------------

The sequence of events for this new Ansible playbook to be triggered during
initial deployment with TripleO follows:

1. Define the Overcloud on the Undercloud in Heat. This includes the Heat
parameters that are related to PowerFlex which will later be passed to
powerflex-ansible via TripleO Ansible playbook.

2. Run `openstack overcloud deploy` with default PowerFlex options and include
a new Heat environment file to make the implementation of the service
deployment use powerflex-ansible.

3. The undercloud assembles and uploads the deployment plan to the undercloud
Swift.

4. TripleO starts to deploy the Overcloud and interfaces with Heat accordingly.

5. A point in the deployment is reached where the Overcloud nodes are imaged,
booted, and networked. At that point the undercloud has access to the
provisioning or management IPs of the Overcloud nodes.

6. The TripleO Ansible playbook responsible to Deploy PowerFlex with any of
the four PowerFlex services, including PowerflexMDM, PowerflexLIA, PowerflexSDS
and PowerflexSDC.

7. The servers which host PowerFlex services have their relevant firewall ports
opened according to the needs of their service, e.g. the PowerflexMDM are
configured to accept traffic on TCP port 9011 and 6611.

8. A new Heat environment file which defines additional parameters that we want
to override is passed to the TripleO Ansible playbook.

9. The TripleO Ansible playbook translates these parameters so that they match
the parameters that powerflex-ansible expects. The translation entails building
an argument list that may be passed to the playbook by calling
`ansible-playbook --extra-vars`. An alternative location for the
/usr/share/powerflex-ansible playbook is possible via an argument. No
playbooks are run yet at this stage.

10. The TripleO Ansible playbook is called and passed the list
of parameters as described earlier. A dynamic Ansible inventory is used with the
`-i` option. In order for powerflex-ansible to work there must be a group called
`[mdms]`, '[tbs]', '[sdss]' and '[sdcs]' in the inventory.

11. The TripleO Ansible playbook starts the PowerFlex install using the
powerflex-ansible set of playbooks

Update/Upgrade PowerFlex SW
---------------------------

TBD

Scaling up/down
---------------

This implementation supports the add or remove of SDS and/or SDC at any moment
(Day+N operations) using the same deployment method.

1. The deployer chooses which type of node he wants to add or remove from the
Powerflex system.

2. The deployer launches an update on the Overcloud which will bring up or down
the nodes to add/remove.

3. The nodes will be added or removed from the Overcloud.

4. The SDS and SDC SW will be added or removed from the PowerFlex system.

5. Storage capacity will be updated consequently.
For Scaling down operation, it will succeed only if:
- the minimum of 3 SDS nodes remains
- the free storage capacity available is enough for rebalancing the data

PowerFlex services breakdown
============================

The PowerFlex system is broken down into multiple components, each of these have
to be installed on specific node types.

Non HCI model
-------------

- Controllers will host the PowerflexLIA, PowerflexMDM and PowerflexSDC (Glance)
  components. A minimum of 3 MDMs is required.

- Computes will host the PowerflexLIA and PowerflexSDC as they will be
  responsible for accessing volumes. There is no minimum.

- Storage will host the PowerflexLIA and PowerflexSDS as disks will be presented
  as backend.  A minimum of 3 SDS is required. A minimum of 1 disk per SDS is
  also required to connect the SDS.

HCI model
---------

- Controllers will host the PowerflexLIA, PowerflexMDM and PowerflexSDC (Glance)
  components. A minimum of 3 MDMs is required.

- Compute HCI will host the PowerflexLIA and PowerflexSDC as they will be
  responsible for accessing volumes and the PowerflexSDS as disks will be
  presented as backend.  A minimum of 3 SDS is required. A minimum of 1 disk per
  SDS is also required to connect the SDS.

Security impact
===============

- A new SSH key pair will be created on the undercloud.
  The public key of this pair will be installed in the heat-admin user's
  authorized_keys file on all Overcloud nodes which will be MDMs, SDSs, or SDCs.
  This process will follow the same pattern used to create the SSH keys used for
  TripleO validations so nothing new would happen in that respect; just another
  instance on the same type of process.

- Additional firewall configuration need to include all TCP/UDP ports needed by
  Powerflex services according to the following:
  | Overcloud role | PowerFlex Service | Ports                  |
  | -------------- | ----------------- | ---------------------- |
  | Controller     | LIA, SDC, SDS     | 9099, 7072, 6611, 9011 |
  | Compute        | LIA, SDC          | 9099                   |
  | Storage        | LIA, SDS          | 9099, 7072             |

- Kernel modules package like scini.ko will be installed depending of the
  version of the operating system of the overcloud node.

- Question:  Will there be any SELinux change needed for IP ports that vxflexOS
  is using?

Performance Impact
==================
The following applies to the undercloud:

- TripleO Ansible will need to run an additional playbook

