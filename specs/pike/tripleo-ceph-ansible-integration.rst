..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
 Enable TripleO to Deploy Ceph via Ceph Ansible
===============================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-ceph-ansible

Enable TripleO to deploy Ceph via Ceph Ansible using a new Mistral
workflow. This will make the Ceph installation less tightly coupled
with TripleO but the existing operator interfaces to deploy Ceph with
TripleO will still be supported until the end of the Queens release.

Problem Description
===================

The Ceph community maintains ceph-ansible to deploy and manage Ceph.
Members of the TripleO community maintain similar tools too. This is
a proposal to have TripleO trigger the Ceph community's tools via
Mistral as an alternative method to deploy and manage Ceph.

Benefits of using another project to deploy and manage Ceph
===========================================================

Avoid duplication of effort
---------------------------

If there is a feature or bug fix in the Ceph community's tools not in
the tools used by TripleO, then members of the TripleO community could
allow deployers to use those features directly instead of writing
their own implementation. If this proposal is successful, then it
might result in not maintaining two code bases, (along with the bug
fixes and testing included) in the future. For example, if
ceph-ansible fixed a bug to correctly handle alternative system paths
to block devices, e.g. /dev/disk/by-path/ in lieu of /dev/sdb, then
the same bug would not need to be fixed in puppet-ceph. This detail
would also be nicely abstracted from a deployer because this spec
proposes maintaining parity with TripleO Heat Templates. Thus, the
deployer would not need to change the `ceph::profile::params::osds`
parameter as the same list of OSDs would work.

In taking this approach it's possible for there to be cases where
TripleO's deployment architecture may have unique features that don't
exist within ceph-ansible. In these cases, efforts may need to be
taken so ensure such a features remian in parity with this approach.
In no way, does this proposal enable a TripleO deployer to bypass
TripleO and use ceph-ansible directly. Also, because Ceph is not an
OpenStack service itself but a service that TripleO uses, this
approach remains consistent with the TripleO mission.


Consistency between OpenStack and non-OpenStack Ceph deployments
----------------------------------------------------------------

A deployer may seek assistance from the Ceph community with a Ceph
deployment and this process will be simplified if both deployments
were done using the same tool.

Enable Decoupling of Ceph management from TripleO
-------------------------------------------------

The complexity of Ceph management can be moved to a different tool
and abstracted, where appropriate, from TripleO making the Ceph
management aspect of TripleO less complex. Combining this with
containerized Ceph would offer flexible deployment options. This
is a deployer benefit that is difficult to deliver today.

Features in the Ceph community's tools not in TripleO's tools
-------------------------------------------------------------

The Ceph community tool, ceph-ansible [1]_, offers benefits to
OpenStack users not found in TripleO's tool chain, including playbooks
to deploy Ceph in containers and migrate a non-containerized
deployment to a containerized deployment without downtime. Also,
making the Ceph deployment in TripleO less tightly coupled, by moving
it into a new Mistral workflow, would make it easier in a future
release to add a business logic layer through a tool like Tendrl [2]_,
to offer additional Ceph policy based configurations and possibly a
graphical tool to see the status of the Ceph cluster. However, the
scope of this proposal for Pike does not include Tendrl and instead
takes the first step towards deploying Ceph via a Mistral workflow by
triggering ceph-ansible directly. After the Pike cycle is complete
triggering Mistral may be considered in a future spec.

Proposed Change
===============

Overview
--------

The ceph-ansible [1]_ project provides a set of playbooks to deploy
and manage Ceph. A proof of concept [3]_ has been written which uses
two custom Mistral actions from the experimental
mistral-ansible-actions project [4]_ to have a Mistral workflow on the
undercloud trigger ceph-ansible to produce a working hyperconverged
overcloud.

The deployer experience to stand up Ceph with TripleO at the end of
this cycle should be the following:

#. The deployer chooses to deploy a role containing any of the
   Ceph server services: CephMon, CephOSD, CephRbdMirror, CephRgw,
   or CephMds.
#. The deployer provides the same Ceph parameters they provide today
   in a Heat env file, e.g. a list of OSDs.
#. The deployer starts the deploy and gets an overcloud with Ceph

Thus, the deployment experience remains the same for the deployer but
behind the scenes a Mistral workflow is started which triggers
ceph-ansible. The details of the Mistral workflow to accomplish this
follows.

TripleO Ceph Deployment via Mistral
-----------------------------------

TripleO's workflow to deploy a Ceph cluster would be changed so that
there are two ways to deploy a Ceph cluster; the way currently
supported by TripleO and the way described in this proposal.

The workflow described here assumes the following:

#. A deployer chooses to deploy Ceph server services from the
   following list of five services found in THT's roles_data.yaml:
   CephMon, CephOSD, CephRbdMirror, CephRgw, or CephMds.
#. The deployer chooses to include new Heat environment files which
   will be in THT when this spec is implemented. The new Heat
   environment file will change the implementation of any of the five
   services from the previous step. Using storage-environment.yaml,
   which defaults to Ceph deployed by puppet-ceph, will still trigger
   the Ceph deployment by puppet-ceph. However, if the new Heat
   environment files are included instead of storage-environment.yaml,
   then the implementation of the service will be done by ceph-ansible
   instead; which already configures these services for hosts under
   the following roles in the Ansible inventory: mons, osds, mdss,
   rgws, or rbdmirrors.
#. The undercloud has a directory called /usr/share/ceph-ansible
   which contains the ceph-ansible playbooks described in this spec.
   It will be present because its install will contain the
   installation of the ceph-ansible package.
#. The Mistral on the Undercloud will contain to custom actions called
   `ansible` and `ansible-playbook` (or similar) and will also contain
   the workflow for each task below and can be observed by running
   `openstack workflow list`. Assume this is the case because the
   tripleo-common package will be modified to ship these actions and
   they will be available after undercloud installation.
#. Heat will ship a new CustomResource type like
   OS::Mistral::WorflowExecution [6]_, which will execute custom
   Mistral workflows.

The standard TripleO workflow, as executed by a deployer, will create
a custom Heat resource which starts an independent Mistral workflow to
interact with ceph-ansible. An example of such a Heat resource would be
OS::Mistral::WorflowExecution [6]_.

Each independent Mistral workflow may be implemented directly in
tripleo-common/workbooks. A separate Mistral workbook will be created
for each goal described below:

* Initial deployment of OpenStack and Ceph
* Adding additional Ceph OSDs to existing OpenStack and Ceph clusters

The initial goal for the Pike cycle will be to maintain feature parity
with what is possible today in TripleO and puppet-ceph but with
containerized Ceph. Additional Mistral workflows may be written, time
permitting or in a future cycle to add new features to TripleO's Ceph
deployment which leverage ceph-ansible playbooks to shrink the Ceph
Cluster and safely remove an OSD or to perform maintenance on the
cluster by using Ceph's 'noout' flag so that the maintenance does not
result in more data migration than necessary.

Initial deployment of OpenStack and Ceph
----------------------------------------

The sequence of events for this new Mistral workflow and Ceph-Ansible
to be triggered during initial deployment with TripleO follows:

#. Define the Overcloud on the Undercloud in Heat. This includes the
   Heat parameters that are related to storage which will later be
   passed to ceph-ansible via a Mistral workflow.
#. Run `openstack overcloud deploy` with standard Ceph options but
   including a new Heat environment file to make the implementation
   of the service deployment use ceph-ansible.
#. The undercloud assembles and uploads the deployment plan to the
   undercloud Swift and Mistral environment.
#. Mistral starts the workflow to deploy the Overcloud and interfaces
   with Heat accordingly.
#. A point in the deployment is reached where the Overcloud nodes are
   imaged, booted, and networked. At that point the undercloud has
   access to the provisioning or management IPs of the Overcloud
   nodes.
#. A new Heat Resource is created which starts a Mistral workflow to
   Deploy Ceph on the systems with the any of the five Ceph server
   services, including CephMon, CephOSD, CephRbdMirror, CephRgw, or
   CephMds [6]_.
#. The servers which host Ceph services have their relevant firewall
   ports opened according to the needs of their service, e.g. the Ceph
   monitor firewalls are configured to accept connections on TCP
   port 6789. [7]_.
#. The Heat resource is passed the same parameters normally found in
   the tripleo-heat-templates environments/storage-environment.yaml
   but instead through a new Heat environment file. Additional files
   may be passed to include overrides, e.g. the list of OSD disks.
#. The Heat resource passes its parameters to the Mistral workflow as
   parameters. This will include information about which hosts should
   have which of the five Ceph server services.
#. The Mistral workflow translates these parameters so that they match
   the parameters that ceph-ansible expects, e.g.
   ceph::profile::params::osds would become devices though they'd have
   the same content, which would be a list of block devices. The
   translation entails building an argument list that may be passed
   to the playbook by calling `ansible-playbook --extra-vars`.
   Typically ceph-ansible uses modified files in the group_vars
   directory but in this case, no files are modified and instead the
   parameters are passed programmatically. Thus, the playbooks in
   /usr/share/ceph-ansible may be run unaltered and that will be the
   default directory. However, it will be possible to pass an
   alternative location for the /usr/share/ceph-ansible playbook as
   an argument. No playbooks are run yet at this stage.
#. The Mistral environment is updated to generate a new SSH key-pair
   for ceph-ansible and the Overcloud nodes using the same process
   that is used to create the SSH keys for TripleO validations and
   install the public key on Overcloud nodes. After this environment
   update it will be possible to run `mistral environment-get
   ssh_keys_ceph` on the undercloud and see the public and private
   keys in JSON.
#. The Mistral Action Plugin `ansible-playbook` is called and passed
   the list of parameters as described earlier. The dynamic ansible
   inventory used by tripleo-validations is used with the `-i`
   option. In order for ceph-ansible to work as usual there must be a
   group called `[mons]` and `[osds]` in the inventory. In addition to
   optional groups for `[mdss]`, `[rgws]`,  or `[rbdmirrors]`.
   Modifications to the tripleo-validations project's
   tripleo-ansible-inventory script may be made to support this, or a
   derivative work of the same as shipped by TripleO common. The SSH
   private key for the heat-admin user and the provisioning or
   management IPs of the Overcloud nodes are what Ansible will use.
#. The mistral workflow computes the number of forks in Ansible
   according to the number of machines that are going to be
   bootstrapped and will pass this number with `ansible-playbook
   --forks`.
#. Mistral verifies that the Ansible ping module can execute `ansible
   $group -m ping` for any group in mons, osds, mdss, rgws, or
   rbdmirrors, that was requested by the deployer. For example, if the
   deployer only specified the CephMon and CephOSD service, then
   Mistral will only run `ansible mons -m ping` and `ansible osds -m
   ping`. The Ansible ping module will SSH into each host as the
   heat-admin user with key which was generated as described
   previously. If this fails, then the deployment fails.
#. Mistral starts the Ceph install using the `ansible-playbook`
   action.
#. The Mistral workflow creates a Zaqar queue to send progress
   information back to the client (CLI or web UI).
#. The workflow posts messages to the "tripleo" Zaqar queue or the
   queue name provided to the original deploy workflow.
#. If there is a problem during the status of the deploy may be seen
   by `openstack workflow execution list | grep ceph` and in the logs
   at /var/log/mistral/{engine.log,executor.log}. Running `openstack
   stack resource list` would show the custom Heat resource that
   started the Mistral workflow, but `openstack workflow execution
   list` and `openstack workflow task list` would contain more details
   about what steps completed within the Mistral workflow.
#. The Ceph deployment is done in containers in a way which must
   prevent any configuration file conflict for any composed service,
   e.g. if a Nova compute container (as deployed by TripleO) and a
   Ceph OSD container are on the same node, then they must have
   different ceph.conf files, even if those files have the same
   content. Though, ceph-ansible will manage ceph.conf for Ceph
   services and puppet-ceph will still manage ceph.conf for OpenStack
   services, neither tool will both try to manage the same ceph.conf
   because it will be in a different location on the container host
   and bind mounted to /etc/ceph/ceph.conf within different
   containers.
#. After the Mistral workflow is completed successfully, the custom
   Heat resource is considered successfully created. If the Mistral
   workflow does not complete successfully, then the Heat resource
   is not considered successfully created. TripleO should handle this
   the same way that it handles any Heat resource that failed to be
   created. For example, because the workflow is idempotent, if the
   resource creation fails because the wrong parameter was passed or
   becasue of a temporary network issue, the deployer could simply run
   a stack-update the Mistral worklow would run again and if the
   issues which caused the first run to fail were resolved, the
   deployment should succeed. Similarly if a user updates a parameter,
   e.g. a new disk is added to `ceph::profile::params::osds`, then the
   workflow will run again without breaking the state of the running
   Ceph cluster but it will configure the new disk.
#. After the dependency of the previous step is satisfied, the TripleO
   Ceph external Heat resource is created to configure the appropriate
   Overcloud nodes as Ceph clients.
#. For the CephRGW service, hieradata will be emitted so that it may
   be used for the haproxy listener setup and keystone users setup.
#. The Overcloud deployment continues as if it was using an external
   Ceph cluster.

Adding additional Ceph OSD Nodes to existing OpenStack and Ceph clusters
------------------------------------------------------------------------

The process to add an additional Ceph OSD node is similar to the
process to deploy the OSDs along with the Overcloud:

#. Introspect the new hardware to host the OSDs.
#. In the Heat environment file containing the node counts, increment
   the CephStorageCount.
#. Run `openstack overcloud deploy` with standard Ceph options and the
   environment file which specifies the implementation of the Ceph
   deployment via ceph-ansible.
#. The undercloud updates the deployment plan.
#. Mistral starts the workflow to update the Overcloud and interfaces
   with Heat accordingly.
#. A point in the deployment is reached where the new Overcloud nodes
   are imaged, booted, and networked. At that point the undercloud has
   access to the provisioning or management IPs of the Overcloud
   nodes.
#. A new Heat Resource is created which starts a Mistral workflow to
   add new Ceph OSDs.
#. TCP ports 6800:7300 are opened on the OSD host [7]_.
#. The Mistral environment already has an SSH key-pair as described in
   the initial deployment scenario. The same process that is used to
   install the public SSH key on Overcloud nodes for TripleO
   validations is used to install the SSH keys for ceph-ansible.
#. If necessary, the Mistral workflow updates the number of forks in
   Ansible according to the new number of machines that are going to
   be bootstrapped.
#. The dynamic Ansible inventory will contain the new node.
#. Mistral confirms that Ansible can execute `ansible osds -m ping`.
   This causes Ansible to SSH as the heat-admin user into all of the
   CephOsdAnsible nodes, including the new nodes. If this fails, then
   the update fails.
#. Mistral uses the Ceph variables found in Heat as described in the
   initial deployment scenario.
#. Mistral runs the osd-configure.yaml playbook from ceph-ansible to
   add the extra Ceph OSD server.
#. The OSDs on the server are each deployed in their own containers
   and `docker ps` will list each OSD container.
#. After the Mistral workflow is completed, the Custom Heat resource
   is considered to be updated.
#. No changes are necessary for the TripleO Ceph external Heat
   resource since the Overcloud Ceph clients only need information
   about new OSDs from the Ceph monitors.
#. The Overcloud deployment continues as if it was using an external
   Ceph cluster.

Containerization of configuration files
---------------------------------------

As described in the Containerize TripleO spec, configuration files
for the containerized service will be generated by Puppet and then
passed to the containerized service using a configuration volume [8]_.
A similar containerization feature is already supported by
ceph-ansible, which uses the following sequence to generate the
ceph.conf configuration file.

* Ansible generates a ceph.conf on a monitor node
* Ansible runs the monitor container and bindmount /etc/ceph
* No modification is being done in the ceph.conf
* Ansible copies the ceph.conf to the Ansible server
* Ansible copies the ceph.conf and keys to the appropriate machine
* Ansible runs the OSD container and bindmount /etc/ceph
* No modification is being done in the ceph.conf

These similar processes are compatible, even in the case of container
hosts which run more than one OpenStack service but which each need
their own copy of the configuration file per container. For example,
consider a containerzation node which hosts both Nova compute and Ceph
OSD services. In this scenario, the Nova compute service would be a
Ceph client and puppet-ceph would generate its ceph.conf and the Ceph
OSD service would be a Ceph server and ceph-ansible would generate its
ceph.conf. It is necessary for Puppet to configure the Ceph client
because Puppet configures the other OpenStack related configuration
files as is already provided by TripleO. Both generated ceph.conf
files would need to be stored in a separate directory on the
containerization hosts to avoid conflicts and the directories could be
mapped to specific containers. For example, host0 could have the
following versions of foo.conf for two different containers::

     host0:/container1/etc/foo.conf  <--- generated by conf tool 1
     host0:/container2/etc/foo.conf  <--- generated by conf tool 2

When each container is started on the host, the different
configuration files could then be mapped to the different containers::

     docker run containter1 ... /container1/etc/foo.conf:/etc/foo.conf
     docker run containter2 ... /container2/etc/foo.conf:/etc/foo.conf

In the above scenario, it is necessary for both configuration files
to be generated from the same parameters. I.e. both Puppet and Ansible
will use the same values from the Heat environment file, but will
generate the configuration files differently. After the configuration
programs have run it won't matter that Puppet idempotently updated
lines of the ceph.conf and that Ansible used a Jina2 template. What
will matter is that both configuration files have the same value,
e.g. the same FSID.

Configuration files generated as described in the Containerize TripleO
spec will not store those configuration files on the container
host's /etc directory before passing it to the container guest with a
bind mount. By default, ceph-ansible generates the initial ceph.conf
on the container host's /etc directory before it uses a bind mount to
pass it through to the container. In order to be consistent with the
Containerize TripleO spec, ceph-ansible will get a new feature for
deploying Ceph in containers so that it will not generate the
ceph.conf on the container host's /etc directory. The same option will
need to apply when generating Ceph key rings; which will be stored in
/etc/ceph in the container, but not on the container host.

Because Mistral on the undercloud runs the ansible playbooks, the
user "mistral" on the undercloud will be the one that SSH's into the
overcloud nodes to run ansible playbooks. Care will need to be taken
to ensure that user doesn't make changes which are out of scope.

Alternatives
------------

From a high level, this proposal is an alternative to the current
method of deploying Ceph with TripleO and offers the benefits listed
in the problem description.

From a lower level, how this proposal is implemented as described in
the Workflow section should be considered.

#. In a split-stack scenario, after the hardware has been provisioned
   by the first Heat stack and before the configuration Heat stack is
   created, a Mistral workflow like the one in the POC [3]_ could be
   run to configured Ceph on the Ceph nodes. This scenario would be
   more similar to the one where TripleO is deployed using the TripleO
   Heat Templates environment file puppet-ceph-external.yaml. This
   could be an alternative to a new OS::Mistral::WorflowExecution Heat
   resource [6]_.
#. Trigger the ceph-ansible deployment before the OpenStack deployment
   In the initial workflow section, it is proposed that "A new
   Heat Resource is created which starts a Mistral workflow to Deploy
   Ceph". This may be difficult because, in general, composable services
   currently define snippets of puppet data which is then later combined
   to define the deployment steps, and there is not yet a way to support
   running an arbitrary Mistral workflow at a given step of a deployment.
   Thus, the Mistral workflow could be started first and then it could
   wait for what is described in step 6 of the overview section.

Security Impact
---------------

* A new SSH key pair will be created on the undercloud and will be
  accessible in the Mistral environment via a command like
  `mistral environment-get ssh_keys_ceph`. The public key of this
  pair will be installed in the heat-admin user's authorized_keys
  file on all Overcloud nodes which will be Ceph Monitors or OSDs.
  This process will follow the same pattern used to create the SSH
  keys used for TripleO validations so nothing new would happen in
  that respect; just another instance on the same type of process.
* An additional tool would do configuration on the Overcloud, though
  the impact of this should be isolated via Containers.
* Regardless of how Ceph services are configured, they require changes
  to the firewall. This spec will implement parity in fire-walling for
  Ceph services [7]_.

Other End User Impact
---------------------

None.

Performance Impact
------------------

The following applies to the undercloud:

* Mistral will need to run an additional workflow
* Heat's role in deploying Ceph would be lessened so the Heat stack
  would be smaller.

Other Deployer Impact
---------------------

Ceph will be deployed using a method that is proven but who's
integration is new to TripleO.

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fultonj

Other contributors:
  gfidente
  leseb
  colonwq
  d0ugal (to review Mistral workflows/actions)

Work Items
----------

* Prototype a Mistral workflow to independently install Ceph on
  Overcloud nodes [3]_. [done]
* Prototype a Heat Resource to start an independent Mistral Workflow
  [6]_. [done]
* Expand mistral-ansible-actions with necessary options (fultonj)
* Parametize mistral workflow (fultonj)
* Update and have merged Heat CustomResource [6]_ (gfidente)
* Have ceph-ansible create openstack pools and keys for containerized
  deployments: https://github.com/ceph/ceph-ansible/issues/1321 (leseb)
* get ceph-ansible packaged in ceph.com and push to centos cbs
  (fultonj / leseb)
* Make undercloud install produce /usr/share/ceph-ansible by modifying
  RDO's instack RPM's spec file to add a dependency (fultonj)
* Submit mistral workflow and ansible-mistral-actions to
  tripleo-common (fultonj)
* Prototype new service plugin interface that defines per-service
  workflows (gfidente / shardy / fultonj)
* Submit new services into tht/roles_data.yaml so users can use it.
  This should include a change to the tripleo-heat-templates
  ci/environments/scenario001-multinode.yaml to include the new
  service, e.g. CephMonAnsible so that CI is tested. This may not
  work unless it all co-exists in a single overcloud deploy.
  If it works, we use it to get started. The initial plan is for
  scenario004 to keep using puppet-ceph.
* Implement the deleting the Ceph Cluster scenario
* Implement the adding additional Ceph OSDs to existing OpenStack and
  Ceph clusters scenario
* Implement the removing Ceph OSD nodes scenario
* Implement the performing maintenance on Ceph OSD nodes (optional)

Dependencies
============

Containerization of the Ceph services provided by ceph-ansible is
used to ensure the configuration tools aren't competing. This
will need to be compatible with the Containerize TripleO spec
[9]_.

Testing
=======

A change to tripleo-heat-templates' scenario001-multinode.yaml will be
submitted which includes deployment of the new services CephMonAnsible
and CephOsdAnsible (note that these role names will be changed when
fully working). This testing scenario may not work unless all of the
services may co-exist; however, preliminary testing indicates that
this will work. Initially scenario004 will not be modified and will be
kept using puppet-ceph. We may start by changing ovb-nonha scenario
first as we believe this may be faster. When the CI move to
tripleo-quickstart happens and there is a containers only scenario we
will want to add a hyperconverged containerized deployment too.

Documentation Impact
====================

A new TripleO Backend Configuration document "Deploying Ceph with
ceph-ansible" would be required.

References
==========

.. [1] `ceph-ansible <https://github.com/ceph/ceph-ansible>`_
.. [2] `Tendrl <https://github.com/Tendrl/documentation>`_
.. [3] `POC tripleo-ceph-ansible <https://github.com/fultonj/tripleo-ceph-ansible>`_
.. [4] `Experimental mistral-ansible-actions project <https://github.com/d0ugal/mistral-ansible-actions>`_
.. [5] `Example Custom TripleO role without OpenStack services for configuration by independent Mistral workflow <https://github.com/fultonj/oooq/commit/2e2635f8cae347013737a89341b2cca24b68c28c>`_
.. [6] `Proposed new Heat resource OS::Mistral::WorflowExecution <https://review.openstack.org/#/c/420664>`_
.. [7] `These firewall changes must be managed in a way that does not conflict with TripleO's mechanism for managing host firewall rules and should be done before the Ceph servers are deployed. We are working on a solution to this problem.`
.. [8] `Configuration files generated by Puppet and passed to a containerized service via a config volume <https://review.openstack.org/#/c/416421/29/docker/docker-puppet.py>`_
.. [9] `Spec to Containerize TripleO <https://review.openstack.org/#/c/223182>`_
