..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============
TripleO Ceph
============

https://blueprints.launchpad.net/tripleo/+spec/tripleo-ceph

A light Ansible framework for TripleO integration with Ceph clusters
deployed with cephadm_ and managed with Ceph orchestrator_.


Problem Description
===================

Starting in the Octopus release, Ceph has its own day1 tool called
cephadm_ and it's own day2 tool called orchestrator_ which will
replace ceph-ansible_. What should TripleO's Ceph integration
do about this? We currently provide the following user experience:

 Describe an OpenStack deployment, which includes Ceph, and TripleO
 will "make it so"

The above has been true for TripleO since Kilo and should
continue. TripleO should also continue hyper-converged support
(collocation of OpenStack and Ceph containers). There is sufficient
value in both of these (one tool and hyper-convergence) to justify
this project. At the same time we want to deploy Ceph in a way
consistent with the way the Ceph project is moving and decouple the
complexity of day2 management of Ceph from TripleO.


Proposed Change
===============

Overview
--------

Modify tripleo-ansible, tripleo-heat-templates, and
python-tripleoclient in support of the following goals:

- Provide Ansible roles which deploy Ceph by calling cephadm_ and Ceph
  orchestrator
- Focus on the day1 problem for Ceph RBD, RGW, CephFS, and Dashboard
  deployment by leveraging `cephadm bootstrap --apply-spec` as
  described in Ceph issue 44873_
- By default, day2 Ceph operations should be done directly with Ceph
  orchestrator_ or Ceph Dashboard and not by running `openstack
  overcloud deploy`
- TripleO stack updates do not trigger the new Ansible roles
  introduced by this spec.
- Provide an opinionated Ceph installation based on parameters from
  TripleO (including hardware details from Ironic)
- Configure cephx keyrings and pools for OpenStack on a deployed Ceph
  cluster
- Support collocation (hyperconvergence) of OpenStack/Ceph containers
  on same host
  - cephadm_ reconciliation loop must not break OpenStack configuration
  - TripleO configuration updates must not break Ceph configuration
- Provide Ceph integration but maximize orthogonality between
  OpenStack and Ceph

The implementation of the TripleO CephClient service during the W
cycle is covered in a different spec in review 757644_. This work will
be merged before the work described in this spec as it will be
compatible with the current Ceph deployment methods. It will also be
compatible with the future deployment methods described in this spec.

Integration Points
------------------

The default deployment method of OpenStack/Ceph for TripleO Victoria
is the following 2-step-process:

1. Deploy nodes with metalsmith_
2. Deploy OpenStack and Ceph with `openstack overcloud deploy`

The Ceph portion of item 2 uses external_deploy_steps_tasks to call
ceph-ansible by using the tripleo-ansible roles: tripleo_ceph_common,
tripleo_ceph_uuid, tripleo_ceph_work_dir, tripleo_ceph_run_ansible.

The ultimate goal for this spec is to support the following
4-step-process:

1. Deploy the hardware with metalsmith_
2. Configure networking (including storage networks)
3. Deploy Ceph with the roles and interface provided by tripleo-ansible/python-tripleoclient
4. Deploy OpenStack with `openstack overcloud deploy`

Item 2 above depends on the spec for network data v2 format described
in review 752437_ and a subsequent network-related feature which moves
port management out of Heat, and supports applying network
configuration prior to Heat stack deployment described in review
760536_.

Item 3 above is the focus of this spec but it is not necessarily
the only integration point. If it is not possible to configure the
storage networks prior to deploying OpenStack, then the new method
of Ceph deployment will still happen via external_deploy_steps_tasks
as it currently does in Victoria via the 2-step-process. Another way
to say this is that Ceph may be deployed *during* the overcloud
deployment in the 2-step process or Ceph may be deployed *before* the
overcloud during the 4-step process; in either case we will change how
Ceph is deployed.

The benefit of deploying Ceph before deploying the overcloud is that
the complexity of the Ceph deployment is decoupled from the complexity
of the OpenStack deployment. Even if Ceph is deployed before the
overcloud, its deployment remains a part of TripleO the same way that
the bare metal deployment remains a part of TripleO; even though a
separate tool, e.g. metalsmith_ or cephadm_ is used to deploy the
resources which are not deployed when `openstack overcloud deploy`
is run.

Additional details on how Ceph is deployed before vs during the
overcloud deployment are covered in the implementation section.

Alternatives
------------

We could ask deployers to do this:

- Deploy hardware and configure networking
- Use cephadm_ and orchestrator_ directly to configure that hardware
  with Ceph and create OpenStack pools accessible by CephX clients
- Use TripleO to configure OpenStack

We have completed a POC of the above using Ussuri and config-download
tags to only run certain steps but would prefer to offer an option to
automate the Ceph deployment. The TripleO project has already ensured
that the move from one to three is automated and requires only two
commands because the tripleo python client now has an option to call
metalsmith_. The alternative is to not automate step two, but that is
user unfriendly.

Another alternative is to continue using ceph-ansible_ as we do today.
However, even though ceph-ansible_ can deploy Octopus today and will
continue to support deployment of Luminous and Nautilus, the project
has a cephadm-adopt_ playbook for converting Ceph clusters that it has
deployed to mangement by cephadm_ orchestrator_ so seems to be moving
away from true Octopus support. ceph-ansible_ has lot of code and day2
support; porting ceph-ansible itself to cephadm_ or orchestrator_ is
more work than completing this project with a smaller scope and looser
coupling.

Security Impact
---------------

The cephadm_ tool is imperative and requires SSH access to the Ceph
cluster nodes in order to execute remote commands and deploy the
specified services. This command will need to be installed on one of
the overcloud nodes which will host the composable CephMon service.
From the cephadm_ point of view, that node will be a bootstrap node
on which the Ceph cluster is created.

For this reason the Ceph cluster nodes must be SSH accessible and
provide a user with root privileges to perform some tasks. For
example, the standard way to add a new host when using cephadm_ is to
run the following:

- `ssh-copy-id -f -i /etc/ceph/ceph.pub root@*<new-host>*`
- `ceph orch host add *<new-host>*`

The TripleO deployment flow, and in particular config-download,
already provides the key elements to properly configure and run
the two actions described above, hence the impact from a security
point of view is unchanged compared to the previous deployment model.

We will create a user like ceph-admin using the same process
config-download uses to create the tripleo-admin user and then
cephadm_ will use this user when it runs commands to add other
hosts.

Upgrade Impact
--------------

Ceph Nautilus clusters are still managed by ceph-ansible, and cephadm_
can be enabled, as the new, default backend, once the Octopus release
is reached. Therefore, starting from Nautilus, two main steps are
identified in the upgrade process:

- Upgrade the cluster using ceph-ansible_ `rolling_update`:
  ceph-ansible_ should provide, as already done in the past, a rolling
  update playbook that can be executed to upgrade all the services to
  the Octopus release
- Migrate the existing cluster to cephadm/orchestrator: when all the
  services are updated to Octopus cephadm-adopt_ will be executed as
  an additional step

New Ceph Octopus deployed clusters will use cephadm_ and ceph
orchestrator_ by default, and the future upgrade path will be provided
by cephadm_upgrade_, which will be able to run, stop and resume all
the Ceph upgrade phases. At that point day2 ceph operations will need
to be carried out directly with ceph orchestrator. Thus, it will no
longer be necessary to include the
`tripleo-heat-templates/environments/ceph-ansible/*` files in the
`openstack overcloud deploy` command with the exception of the Ceph
client configuration as described in review 757644_, which will have a
new environment file.

.. note::

    The Upgrade process for future releases can be subject of slight
    modifications according to the OpenStack requirements.


Other End User Impact
---------------------

The main benefit from the operator perspective is the ability to take
advantage of the clear separation between the deployment phase and
day2 operations as well as the separation between the Ceph deployment
and the OpenStack deployment. At the same time TripleO can still
address all the deployment phase operations with a single tool but
leave and rely on orchestrator_ for what concerns day2 tasks.

Many common tasks can now be performed the same way regardless of if
the Ceph cluster is internal (deployed by) or external to TripleO.
The operator can use the cephadm_ and orchestrator_ tools which will
be accessible from one of the Ceph cluster monitor nodes.

For instance, since cephadm_ maintains the status of the cluster, the
operator is now able to perform the following tasks without interacting
with TripleO at all:

1. Monitor replacement
2. OSD replacement (if a hardware change is necessary then Ironic
   might be involved)

.. note::

    Even though cephadm_ standalone, when combined with Ceph
    orchestrator_, should support all the commands required to the
    carry out day2 operations, our plan is for tripleo-ceph to
    continue to manage and orchestrate other actions that can
    be taken by an operator when TripleO should be involved. E.g.
    a CephStorage node is added as a scale-up operation, then
    the tripleo-ceph Ansible roles should make calls to add the OSDs.

Performance Impact
------------------

Stack updates will not trigger Ceph tools so "OpenStack only" changes
won't be delayed by Ceph operations. Ceph client configuration will
take less time though this benefit is covered in review 757644_.

Other Deployer Impact
---------------------

Like ceph-ansible, cephadm_ is distributed as an RPM and can be
installed from Ceph repositories. However, since the deployment
approach is changed and cephadm_ requires a Ceph monitor node to
bootstrap a minimal cluster, we would like to install the cephadm_
RPM on the overcloud image. As of today this RPM is approximately 46K
and we expect this to simplify the installation process. When cephadm_
bootstraps the first Ceph monitor (on the first Controller node by
default) it will download the necessary Ceph containers. To contrast
this proposal with the current Ceph integration, ceph-ansible_ needs
to be installed on the undercloud and it then manages the download of
Ceph containers to overcloud nodes. In the case of both cephadm_ and
ceph-ansible, no other package changes are needed for the overcloud
nodes as both tools run Ceph in containers.

This change affects all TripleO users who deploy an Overcloud which
interfaces with Ceph. Any TripleO users who does not interface with
Ceph will not be directly impacted by this project.

TripleO users who currently use
`environments/ceph-ansible/ceph-ansible.yaml` in order to have their
overcloud deploy an internal Ceph cluster will need to migrate to the
new method when deploying W. This file and others will deprecated as
described in more detail below.

The proposed changes do not take immediate effect after they are
merged because both the ceph-ansible_ and cephadm_ interfaces will
exist intree concurrently.

Developer Impact
----------------

How Ceph is deployed could change for anyone maintaining TripleO code
for OpenStack services which use Ceph. In theory there should be no
change as the CephClient service will still configure the Ceph
configuration and Ceph key files in the same locations. Those
developers will just need to switch to the new interfaces when they
are stable.

Implementation
==============

How configuration data is passed to the new tooling when Ceph is
deployed *before* or *during* the overcloud deployment, as described
in the Integration Points section of the beginning of this spec, will
be covered in more detail in this section.

Deprecations
------------

Files in `tripleo-heat-templates/environments/ceph-ansible/*` and
`tripleo-heat-templates/deployment/ceph-ansible/*` will be deprecated
in W and removed in X. They will be obsoleted by the new THT
parameters covered in the next section with the exception of
`ceph-ansible/ceph-ansible-external.yaml` which will be replaced by
`environments/ceph-client.yaml` as described in review 757644_.

The following tripleo-ansible roles will be deprecated at the start
of W: tripleo_ceph_common, tripleo_ceph_uuid, tripleo_ceph_work_dir,
and tripleo_ceph_run_ansible. The ceph_client role will not be
deprecated but it will be re-implemented as described in review
757644_. New roles will be introduced to tripleo-ansible to replace
them.

Until the project described here is complete during X we will
continue to maintain the deprecated ceph-ansible_ roles and
Heat templates for the duration of W and so it is likely that during
one release we will have intree support both ceph-ansible_ and
cephadm_.

New THT Templates
-----------------

Not all THT configuration for Ceph can be removed. The firewall is
still configured based on THT as descrbed in the next section and THT
also controls which composable service is deployed and where. The
following new files will be created in
`tripleo-heat-templates/environments/`:

- cephadm.yaml: triggers new cephadm Ansible roles until `openstack
  overcloud ceph ...` makes it unnecessary. Contains the paths to the
  files described in the Ceph End State Definition YAML Input section.
- ceph-rbd.yaml: RBD firewall ports, pools and cephx key defaults
- ceph-rgw.yaml: RGW firewall ports, pools and cephx key defaults
- ceph-mds.yaml: MDS firewall ports, pools and cephx key defaults
- ceph-dashboard.yaml: defaults for Ceph Dashboard firewall ports

All of the above (except cephadm.yaml) will result in the appropriate
firewall ports being opened as well as a new idempotent Ansible role
connecting to the Ceph cluster in order to create the Ceph pools and
cephx keys to access those pools. Which ports, pools and keys are
created will depend on which files are included. E.g. if the deployer
ran `openstack overcloud deploy ... -e ceph-rbd.yaml -e cep-rgw.yaml`
then the ports, pools and cephx keys would be configured for Nova,
Cinder, and Glance to use Ceph RBD and RGW would be configured with
Keystone, but no firewall ports, pools and keys for the MDS service
would be created and the firewall would not be opened for the Ceph
dashboard.

None of the above files, except cephadm.yaml, will result in Ceph
itself being deployed and none of the parameters needed to deploy Ceph
itself will be in the above files. E.g. PG numbers and OSD devices
will not be defined in THT anymore. Instead the parameters which are
needed to deploy Ceph itself will be in tripleo_ceph_config.yaml as
described in the Ceph End State Definition YAML Input section and
cephadm.yaml will only contain references to those files.

The cephx keys and pools, created as described above, will result in
output data which looks like the following::

  pools:
  - volumes
  - vms
  - images
  - backups
  openstack_keys:
  - caps:
    mgr: allow *
    mon: profile rbd
    osd: 'osd: profile rbd pool=volumes, profile rbd pool=backups,
         profile rbd pool=vms, profile rbd pool=images'
    key: AQCwmeRcAAAAABAA6SQU/bGqFjlfLro5KxrB1Q==
    mode: '0600'
    name: client.openstack

The above can be written to a file, e.g. ceph_client.yaml, and passed
as input to the the new ceph client role described in review 757644_
(along with the ceph_data.yaml file produced as output as described in
Ceph End State Definition YAML Output).

In DCN deployments this type of information is extracted from the Heat
stack with `overcloud export ceph`. When the new method of deployment
is used this information can come directly from each genereated yaml
file (e.g. ceph_data.yaml and ceph_client.yaml) per Ceph cluster.

Firewall
--------

Today the firewall is not configured by ceph-ansible_ and it won't be
configured by cephadm_ as its `--skip-firewalld` will be used. We
expect the default overcloud to not have firewall rules until
`openstack overcloud deploy` introduces them. The THT parameters
described in the previous section will have the same firewall ports as
the ones they will deprecate (`environments/ceph-ansible/*`) so that
the appropriate ports per service and based on composable roles will
be opened in the firewall as they are today.

OSD Devices
-----------

The current defaults will always be wrong for someone because the
`devices` list of available disks will always vary based on hardware.
The new default will use all available devices when creating OSDs by
running `ceph orch apply osd --all-available-devices`. It will still
be possible to override this default though the ceph-ansible_ syntax of
the `devices` list will be deprecated. In its place the OSD Service
Specification defined by cephadm_ drivegroups will be used and the tool
will apply it by running `ceph orch apply osd -i osd_spec.yml`. More
information on the `osd_spec.yaml` is covered in the Ceph End State
Definition YAML Input section.

Ceph Placement Group Parameters
-------------------------------

The new tool will deploy Ceph with the pg autotuner feature enabled.
Parameters to set the placement groups will be deprecated. Those who
wish to disable the pg autotuner may do so using Ceph CLI tools after
Ceph is deployed.

Ceph End State Definition YAML Input
------------------------------------

Regardless of if Ceph is deployed *before* or *during* overcloud
deployment, a new playbook which deploys Ceph using cephadm_ will be
created and it will accept the following files as input:

- deployed-metal.yaml: this file is generated by running a command
  like `openstack overcloud node provision ... --output
  deployed-metal.yaml` when using metalsmith_.

- (Optional) "deployed-network-env": the file that is generated by
  `openstack network provision` as described in review 752437_. This
  file is used when deploying Ceph before the overcloud to identify
  the storage networks. This will not be necessary when deploying Ceph
  during overcloud deployment so it is optional and the storage
  network will be identified instead as it is today.

- (Optional) Any valid cephadm_ config.yml spec file as described in
  Ceph issue 44205_ may be directly passed to the cephadm_ execution
  and where applicable will override all relevant settings in the file
  described at the end of this list.

- (Optional) Any valid drivegroup_ YAML file (e.g. osd_spec.yml) may
  be passed and the tooling will apply it with `ceph orch apply osd -i
  osd_spec.yml`. This setting will override all relevant settings in
  the file described at the end of this list.

- tripleo_ceph_config.yaml: This file will contain configuration data
  compatible with nearly all Ceph options supported today by TripleO
  Heat Templates with the exception of the firewall, ceph pools and
  cephx keys. A template of this file will be provided in as a default
  in one of the new tripleo-ansible roles (e.g. tripleo_cephadm_common)

Another source of data which is input into the new playbook is the
inventory which is covered next section.

Ansible Inventory and Ansible User
----------------------------------

The current Ceph implementation uses the Ansible user tripleo-admin.
That user and the corresponding SSH keys are created by the
tripleo-ansible role tripleo_create_admin. This role uses the
heat-admin account which is the default account if `openstack
overcloud node provision` is not passed the `--overcloud-ssh-user`
option. The current implementation also uses the inventory generated
by tripleo-ansible-inventory. These resources will not be available
if Ceph is deployed *before* the overcloud and there's no reason they
are needed if Ceph is deployed *during* the overcloud deployment.

Regardless if Ceph is deployed *before* or *during* overcloud, prior
to deploying Ceph, `openstack overcloud admin authorize` should be run
and it should pass options to enable a ceph-admin user which can be
used by cephadm_ and to allow SSH access for the ansible roles
described in this spec.

A new command, `openstack overcloud ceph inventory` will be
implemented which creates an Ansible inventory for the new playbook
and roles described in this spec. This command will require the
following input:

- deployed-metal.yaml: this file is generated by running a command
  like `openstack overcloud node provision ... --output
  deployed-metal.yaml` when using metalsmith_.

- (Optional) roles.yaml: If this file is not passed then
  /usr/share/openstack-tripleo-heat-templates/roles_data.yaml will be
  used in its place. If the roles in deployed-metal.yaml do not have a
  definition found in roles.yaml, then an error is thrown that a role
  being used is undefined. By using this file, the TripleO composable
  roles will continue to work as they to today. The services matching
  "OS::TripleO::Services::Ceph*" will correspond to a new Ansible
  inventory group and the hosts in that group will correspond to the
  hosts found in deployed-metal.yaml.

- (Options) `-u --ssh-user <USER>`: this is not a file but an option
  which defaults to "ceph-admin". This represents the user which was
  created created on all overcloud nodes by `openstack overcloud admin
  authorize`.

- (Options) `-i --inventory <FILE>`: this is not a file but an option
  which defaults to "/home/stack/inventory.yaml". This represents the
  inventory which will be created.

If Ceph is deployed before the overcloud, users will need to run this
command to generate an Ansible inventory file. They will also need to
pass the path to the generated inventory file to `openstack overcloud
ceph provision` as input.

If Ceph is deployed *during* overcloud deployment, users do not need
to know about this command as external_deploy_steps_tasks will run
this command directly to generate the inventory before running the new
tripleo ceph playbook with this inventory.

Ceph End State Definition YAML Output
-------------------------------------

The new playbook will write output data to one yaml file which
contains information about the Ceph cluster and may be used as
input to other processes.

In the case that Ceph is deployed before the overcloud, if `openstack
overcloud ceph provision --output ceph_data.yaml` were run, then
`ceph_data.yaml` would then be passed to `openstack overcloud deploy
... -e ceph_data.yaml`. The `ceph_data.yaml` file will contain
key/value pairs such as the Ceph FSID, Name, and the Ceph monitor IPs.

In the case that Ceph is deployed with the overcloud, if
external_deploy_steps_tasks calls the new playbook, then the same file
will be written to it's default location (/home/stack/ceph_data.yaml)
and the new client role will directly read the parameters from this file.

An example of what this file, e.g. `ceph_data.yaml`, looks like is::

  cluster: ceph
  fsid: af25554b-42f6-4d2b-9b9b-d08a1132d3e899
  ceph_mon_ips:
  - 172.18.0.5
  - 172.18.0.6
  - 172.18.0.7

In DCN deployments this type of information is extracted from the Heat
stack with `overcloud export ceph`. When the new method of deployment
is used this information can come directly from the `ceph_data.yaml`
file per Ceph cluster. This file will be passed as input to the new
ceph client role described in review 757644_.

Requirements for deploying Ceph during Overcloud deployment
-----------------------------------------------------------

If Ceph is deployed *during* the overcloud deployment, the following
should be the case:

- The external_deploy_steps_tasks playbook will execute the new
  Ansible roles after `openstack overcloud deploy` is executed.
- If `openstack overcloud node  provision .. --output
  deployed-metal.yaml` were run, then `deployed-metal.yaml` would be
  input to `openstack overcloud deploy`. This is the current behavior
  we have in V.
- Node scale up operations for day2 Ceph should be done by running
  `openstack overcloud node provision` and then `openstack overcloud
  deploy`. This will include reasserting the configuration of
  OpenStack services unless those operations are specifically set to
  "noop".
- Creates its own Ansible inventory and user
- The path to the "Ceph End State Definition YAML Input" is referenced
  via a THT parameter so that when external_deploy_steps_tasks runs it
  will pass this file to the new playbook.

Requirements for deploying Ceph before Overcloud deployment
-----------------------------------------------------------

If Ceph is deployed *before* the overcloud deployment, the following
should be the case:

- The new Ansible roles will be triggered when the user runs a command
  like `openstack overcloud ceph ...`; this command is meant
  to be run after running `openstack overcloud node provision` to
  trigger metalsmith_  but before running `openstack overcloud deploy`.
- If `openstack overcloud node  provision .. --output
  deployed-metal.yaml` were run, then `deployed-metal.yaml` would be
  input to `openstack overcloud ceph provision`.
- Node scale up operations for day2 Ceph should be done by running
  `openstack overcloud node provision`, `openstack overcloud network
  provision`, and `openstack overcloud admin authorize` to enable a
  ceph-admin user. However it isn't necessary to run `openstack
  overcloud ceph ...` because the operator should connect to the Ceph
  cluster itself to add the extra resources, e.g. use a cephadm shell
  to add the new hardware as OSDs or other Ceph resource. If the
  operation includes adding hyperconverged node with both Ceph and
  OpenStack services then the third step will be to run `openstack
  overcloud deploy`.
- Requires the user to create an inventory (and user) before running
  using new Ceph deployment tools.
- "Ceph End State Definition YAML Input" is directly passed.

Container Registry Support
--------------------------

It is already supported to host a container registry on the
undercloud. This registry contains Ceph and OpenStack containers
and it may be populated before deployment or during deployment.
When deploying ceph before overcloud deployment it will need to be
populated before deployment. The new integration described in this
spec will direct cephadm_ to pull the Ceph containers from the same
source identified by `ContainerCephDaemonImage`. For example::

  ContainerCephDaemonImage: undercloud.ctlplane.mydomain.tld:8787/ceph-ci/daemon:v4.0.13-stable-4.0-nautilus-centos-7-x86_64

Network Requirements for Ceph to be deployed before the Overcloud
-----------------------------------------------------------------

The deployment will be completed by running the following commands:

- `openstack overcloud node provision ...`
- `openstack overcloud network provision ...` (see review 751875_)
- `openstack overcloud ceph ...` (triggers cephadm/orchestrator)
- `openstack overcloud deploy ...`

In the past stack updates did everything, but the split for
metalsmith_ established a new pattern. As per review 752437_ and a
follow up spec to move port management out of Heat, and apply network
configuration prior to the Heat stack deployment, it will eventually
be possible for the network to be configured before `openstack
overcloud deploy` is run. This creates an opening for the larger goal
of this spec which is a looser coupling between Ceph and OpenStack
deployment while retaining full integration. After the storage and
storage management networks are configured, then Ceph can be deployed
before any OpenStack services are configured. This should be possible
regardless of if the same node hosts both Ceph and OpenStack
containers.

Development work on for deploying Ceph before overcloud deployment
can begin before the work described in reviews 752437_ and 760536_
is completed by either of the following methods:

Option 1:
- `openstack overcloud deploy --skip-tags step2,step3,step4,step5`
- use tripleo-ceph development code to stand up Ceph
- `openstack overcloud deploy --tags step2,step3,step4,step5`

The last step will also configure the ceph clients. This sequence has
been verified to work in a proof of concept of this proposal.

Option 2:
- Create the storage and storage management networks from the undercloud (using review 751875_)
- Create the Ironic ports for each node as per review 760536_
- Use instances Nics Properties to pass a list of dicts to provision the node not just on the ctlplane network but also the storage and storage-management networks when the node is provisioned with metalsmith_
- Metalsmith/Ironic should attach the VIFs so that the nodes are connected to the Storage and Storage Management networks so that Ceph can then be deployed.

PID1 services used by Ceph
--------------------------

During the W cycle we will not be able to fully deploy an HA Dashboard
and HA RGW service before the overcloud is deployed. Thus, we will
deploy these services as we do today; by using a ceph tool, though
we'll use cephadm_ in place of ceph-ansible_, and then complete the
configuration of these services during overcloud deployment. Though
the work to deploy the service itself will be done before overcloud
deployment, the service won't be accessible in HA until after the
overcloud deployment.

Why can't we fully deploy the HA RGW service before the overcloud?
Though cephadm_ can deploy an HA RGW service without TripleO its
implementation uses keepalived which cannot be collocated with
pacemaker, which is required on controller nodes. Thus, during the
W cycle we will keep using the RGW service with haproxy and revisit
making it a separate deployment with collaboration with the PID1 team
in a future cycle.

Why can't we fully deploy the HA Dashboard service before the
overcloud? cephadm_ does not currently have a builtin HA model for
its dashboard and the HA Dashboard is only available today when it
is deployed by TripleO (unless it's configured manually).

Ceph services which need VIPs (Dashbard and RGW) need to know what the
VIPs will be in advance but the VIPs do not need to be pingable before
those Ceph services are deployed. Instead we will be able to know what
the VIPs are before deploying Ceph per the work related to reviews
751875_ and 760536_. We will pass these VIPs as input to cephadm_.

For example, if we know the Dashboard VIP in advance, we can run the
following::

  ceph --cluster {{ cluster }} dashboard set-grafana-api-url {{ dashboard_protocol }}://{{ VIP }}:{{ grafana_port }}"

The new automation could then save the VIP parameter in the ceph mgr
global config. A deployer could then and wait for haproxy to be
available from the overcloud deploy so that an HA dashbard similar to
the one Victoria deploys is available.

It would be simpler if we could address the above issues before
overcloud deployment but doing so is out of the scope of this spec.
However, we can aim to offer the dashboard in HA with the new tooling
around the time of the X cycle and we hope to do so through
collaboration with the Ceph orchestrator community.

TripleO today also supports deploying the Ceph dashboard on any
composed network. If the work included in review 760536_ allows us to
compose and deploy the overcloud networks in advance, then we plan to
pass parameters to cephadm to continue support of the dashboard on its
own private network.

TLS-Everywhere
--------------

If Ceph is provisioned before the overcloud, then we will not have
the certificates and keys generated by certmonger via TripleO's
tls-everywhere framework. We expect cephadm to be able to deploy the
Ceph Dashboard (with Grafana), RGW (with HA via haproxy) with TLS
enabled. For the sake of orthogonality we could require that the
certificates and keys for RGW and Dashboard be generated outside of
TripleO so that these services could be fully deployed without the
overcloud. However, because we still need to use PID1 services as
described in the previous section, we will continue to use TripleO's
TLS-e framework.

Assignee(s)
-----------

- fmount
- fultonj
- gfidente
- jmolmo

Work Items
----------

- Create a set of roles matching tripleo_ansible/roles/tripleo_cephadm_*
  which can coexist with the current tripleo_ceph_common,
  tripleo_ceph_uuid, tripleo_ceph_work_dir, tripleo_ceph_run_ansible,
  roles.
- Patch the python tripleo client to support the new command options
- Create a new external_deploy_steps_tasks interface for deploying
  Ceph using the new method during overcloud deployment
- Update THT scenario001/004 to use new method of ceph deployment

Proposed Schedule
-----------------

- OpenStack W: merge tripleo-ansible/roles/ceph_client descrbed in
  review 757644_ early as it will work with ceph-ansible_ internal
  ceph deployments too. Create tripleo-ansible/roles/cephadm_* roles
  and tripleo client work to deploy Octopus as experimental and then
  default (only if stable). If new tripleo-ceph is not yet stable,
  then Wallaby will release with Nautilus support as deployed by
  ceph-ansible_ just like Victoria. Either way Nautilus support via
  current THT and tripleo-ansible triggering ceph-ansible_ will be
  deprecated.

- OpenStack X: tripleo-ansible/roles/cephadm_* become the default,
  tripleo-ansible/roles/ceph_* are removed except the new ceph_client,
  tripleo-heat-templates/environments/ceph-ansible/* removed. Migrate
  to Ceph Pacific which GAs upstream in March 2021.

Dependencies
============

- The spec for tripleo-ceph-client described in review 757644_
- The spec for network data v2 format described in review 752437_
- The spec for node ports described in review 760536_

The last two items above are not required if we deploy Ceph during
overcloud deployment.

Testing
=======

This project will be tested against at least two different scenarios.
This will ensure enough coverage on different use cases and cluster
configurations, which is pretty similar to the status of the job
definition currently present in the TripleO CI.
The defined scenarios will test different features that can be enabled
at day1.
As part of the implementation plan, the definition of the
tripleo-heat-templates environment CI files, which contain the testing job
parameters, is one of the goals of this project, and we should make sure
to have:

- a basic scenario that covers the ceph cluster deployment using cephadm_;
  we will gate the tripleo-ceph project against this scenario, as well
  as the related tripleo heat templates deployment flow;

- a more advanced use case with the purpose of testing the configuration
  that can be applied to the ceph cluster and are orchestrated by the
  tripleo-ceph project.

The two items described above are pretty similar to the test suite that
today is maintained in the TripleO CI, and they can be implemented
reworking the existing scenarios, adding the proper support to the
cephadm_ deployment model.
A WIP patch can be created and submitted with the purpose of testing
and gating the tripleo-ceph project, and, when it becomes stable
enough, the scenario001 will be able to be officially merged.
The same approach can be applied to the existing scenario004, which
can be seen as an improvement of the first testing job.
This is mostly used to test the Rados Gateway service deployment and
the manila pools and key configuration.
An important aspect of the job definition process is related to
standalone vs multinode.
As seen in the past, multinode can help catching issues that are not
visible in a standalone environment, but of course the job
configuration can be improved in the next cycles, and we can start
with standalone testing, which is what is present today in CI.
Maintaining the CI jobs green will be always one of the goals of the
ceph integration project, providing a smooth path and a good experience
moving from ceph-ansible_ to cephadm_, continuously improving the testing
area to ensure enough coverage of the implemented features.

Documentation Impact
====================

tripleo-docs will be updated to cover Ceph integration with the new tool.


.. Indirect Hyperlink Targets

.. _cephadm: https://docs.ceph.com/en/latest/cephadm/
.. _orchestrator: https://docs.ceph.com/en/latest/mgr/orchestrator/
.. _ceph-ansible: https://github.com/ceph/ceph-ansible
.. _metalsmith: https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/provisioning/baremetal_provision.html
.. _cephadm-adopt: https://github.com/ceph/ceph-ansible/blob/master/infrastructure-playbooks/cephadm-adopt.yml
.. _drivegroup: https://docs.ceph.com/en/latest/cephadm/drivegroups
.. _cephadm_upgrade: https://docs.ceph.com/docs/master/cephadm/upgrade
.. _44205: https://tracker.ceph.com/issues/44205
.. _44873: https://tracker.ceph.com/issues/44873
.. _757644: https://review.opendev.org/#/c/757644
.. _752437: https://review.opendev.org/#/c/752437
.. _751875: https://review.opendev.org/#/c/751875
.. _757644: https://review.opendev.org/#/c/757644
.. _760536: https://review.opendev.org/#/c/760536
