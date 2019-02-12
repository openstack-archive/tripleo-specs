..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Major Upgrades Including Operating System Upgrade
=================================================

https://blueprints.launchpad.net/tripleo/+spec/upgrades-with-os

.. note::
   Abbreviation "OS" in this spec stands for "operating system", not
   "OpenStack".

So far all our update and upgrade workflows included doing minor
operating system updates (essentially a ``yum update``) on the
machines managed by TripleO. This will need to change as we can't stay
on a single OS release indefinitely -- we'll need to perform a major
OS upgrade. The intention is for the TripleO tooling to help with the
OS upgrade significantly, rather than leaving this task entirely to
the operator.


Problem Description
===================

We need to upgrade undercloud and overcloud machines to a new release
of the operating system.

We would like to provide an upgrade procedure both for environments
where Nova and Ironic are managing the overcloud servers, and
"Deployed Server" environments where we don't have control over
provisioning.

Further constraints are imposed by Pacemaker clusters: Pacemaker is
non-containerized, so it is upgraded via packages together with the
OS. While Pacemaker would be capable of a rolling upgrade, Corosync
also changes major version, and starts to rely on knet for the link
protocol layer, which is incompatible with previous version of
Corosync. This introduces additional complexity: we can't do OS
upgrade in a rolling fashion naively on machines which belong to the
Pacemaker cluster (controllers).


Proposed Change - High Level View
=================================

The Pacemaker constraints will be addressed by performing a one-by-one
(though not rolling) controller upgrade -- temporarily switching to a
single-controller cluster on the new OS, and gradually upgrading the
rest. This will also require implementation of persistent OpenStack
data transfer from older to newer OS releases (to preserve uptime and
for easier recoverability in case of failure).

We will also need to ensure that at least 2 ceph-mon services run at
all times, so ceph-mon services will keep running even after we switch
off Pacemaker and OpenStack on the 2 older controllers.

We should scope two upgrade approaches: full reprovisioning, and
in-place upgrade via an upgrade tool. Each come with different
benefits and drawbacks. The proposed CLI workflows should ideally be
generic enough to allow picking the final preferred approach of
overcloud upgrade late in the release cycle.

While the overcloud approach is still wide open, undercloud seems to
favor an in-place upgrade due to not having a natural place to persist
the data during reprovisioning (e.g. we can't assume overcloud
contains Swift services), but that could be overcome by making the
procedure somewhat more manual and shifting some tasks onto the
operator.

The most viable way of achieving an in-place (no reprovisioning)
operating system upgrade currently seems to be `Leapp`_, "an app
modernization framework", which should include in-place upgrade
capabilites.

Points in favor of in-place upgrade:

* While some data will need to be persisted and restored regardless of
  approach taken (to allow safe one-by-one upgrade), reprovisioning
  may also require managing data which would otherwise persist on its
  own during an in-place upgrade.

* In-place upgrade allows using the same approach for Nova+Ironic and
  Deployed Server environments. If we go with reprovisioning, on
  Deployed Server environments the operator will have to reprovision
  using their own tooling.

* Environments with a single controller will need different DB
  mangling procedure. Instead of ``system_upgrade_transfer_data`` step
  below, their DB data will be included into the persist/restore
  operations when reprovisioning the controller.

Points in favor of reprovisioning:

* Not having to integrate with external in-place upgrade tool. E.g. in
  case of CentOS, there's currently not much info available about
  in-place upgrade capabilities.

* Allows to make changes which wouldn't otherwise be possible,
  e.g. changing a filesystem.

* Reprovisioning brings nodes to a clean state. Machines which are
  continuously upgraded without reprovisioining can potentially
  accumulate unwanted artifacts, resulting in increased number of
  problems/bugs which only appear after an upgrade, but not on fresh
  deployments.


Proposed Change - Operator Workflow View
========================================

The following is an example of expected upgrade workflow in a
deployment with roles: **ControllerOpenstack, Database, Messaging,
Networker, Compute, CephStorage**. It's formulated in a
documentation-like manner so that we can best imagine how this is
going to work from operator's point of view.


Upgrading the Undercloud
------------------------

The in-place undercloud upgrade using Leapp will likely consist of the
following steps. First, prepare for OS upgrade via Leapp, downloading
the necessary packages::

  leapp upgrade

Then reboot, which will upgrade the OS::

  reboot

Then run the undercloud upgrade, which will bring back the undercloud
services (using the newer OpenStack release)::

  openstack tripleo container image prepare default \
      --output-env-file containers-prepare-parameter.yaml
  openstack undercloud upgrade

If we wanted or needed to upgrade the undercloud via reprovisioning,
we would use a `backup and restore`_ procedure as currently
documented, with restore perhaps being utilized just partially.


Upgrading the Overcloud
-----------------------

#. **Update the Heat stack**, generate Heat outputs for building
   upgrade playbooks::

     openstack overcloud upgrade prepare <DEPLOY ARGS>

   Notes:

   * Among the ``<DEPLOY ARGS>`` should be
     ``containers-prepare-parameter.yaml`` bringing in the containers
     of newer OpenStack release.

#. **Prepare an OS upgrade on one machine from each of the
   "schema-/cluster-sensitive" roles**::

     openstack overcloud upgrade run \
         --tags system_upgrade_prepare \
         --limit controller-openstack-0,database-0,messaging-0

   Notes:

   * This stops all services on the nodes selected.

   * For external installers like Ceph, we'll have a similar
     external-upgrade command, which can e.g. remove the nodes from
     the Ceph cluster::

       openstack overcloud external-upgrade run \
           --tags system_upgrade_prepare \
           -e system_upgrade_nodes=controller-openstack-0,database-0,messaging-0

   * If we use in-place upgrade:

     * This will run the ``leapp upgrade`` command. It should use
       newer OS and newer OpenStack repos to download packages, and
       leave the node ready to reboot into the upgrade process.

     * Caution: Any reboot after this is done on a particular node
       will cause that node to automatically upgrade to newer OS.

   * If we reprovision:

     * This should persist node's important data to the
       undercloud. (Only node-specific data. It would not include
       e.g. MariaDB database content, which would later be transferred
       from one of the other controllers instead.)

     * Services can export their ``upgrade_tasks`` to do the
       persistence, we should provide an Ansible module or role to
       make it DRY.

#. **Upload new overcloud base image**::

     openstack overcloud image upload --update-existing \
         --image-path /home/stack/new-images

   Notes:

   * For Nova+Ironic environments only. After this step any new or
     reprovisioned nodes will receive the new OS.

#. **Run an OS upgrade on one node from each of the
   "schema-/cluster-sensitive" roles** or **reprovision those nodes**.

   Only if we do reprovisioning::

     openstack server rebuild controller-openstack-0
     openstack server rebuild database-0
     openstack server rebuild messaging-0

     openstack overcloud admin authorize \
         --overcloud-ssh-user <user> \
         --overcloud-ssh-key <path-to-key> \
         --overcloud-ssh-network <ssh-network> \
         --limit controller-openstack-0,database-0,messaging-0

   Both reprovisioning and in-place::

     openstack overcloud upgrade run \
         --tags system_upgrade_run \
         --limit controller-openstack-0,database-0,messaging-0

   Notes:

   * This step either performs a reboot of the nodes and lets Leapp
     upgrade them to newer OS, or reimages the nodes with a fresh new
     OS image. After they come up, they'll have newer OS but no
     services running. The nodes can be checked before continuing.

   * In case of reprovisioning:

     * The ``overcloud admin authorize`` will ensure existence of
       ``tripleo-admin`` user and authorize Mistral's ssh keys for
       connection to the newly provisioned nodes. The
       ``--overcloud-ssh-*`` work the same as for ``overcloud
       deploy``.

     * The ``--tags system_upgrade_run`` is still necessary because it
       will restore the node-specific data from the undercloud.

     * Services can export their ``upgrade_tasks`` to do the
       restoration, we should provide an Ansible module or role to
       make it DRY.

   * Ceph-mon count is reduced by 1 (from 3 to 2 in most
     environments).

   * Caution: This will have bad consequences if run by accident on
     unintended nodes, e.g. on all nodes in a single role. If
     possible, it should refuse to run if --limit is not specified. If
     possible further, it should refuse to run if a full role is
     included, rather than individual nodes.

#. **Stop services on older OS and transfer data to newer OS**::

     openstack overcloud external-upgrade run \
         --tags system_upgrade_transfer_data \
         --limit ControllerOpenstack,Database,Messaging

   Notes:

   * **This is where control plane downtime starts.**

   * Here we should:

     * Detect which nodes are on older OS and which are on newer OS.

     * Fail if we don't find *at least one* older OS and *exactly
       one* newer OS node in each role.

     * On older OS nodes, stop all services except ceph-mon. (On newer
       node, no services are running yet.)

     * Transfer data from *an* older OS node (simply the first one in
       the list we detect, or do we need to be more specific?) to
       *the* newer OS node in a role. This is probably only going to
       do anything on the Database role which includes DBs, and will
       be a no-op for others.

     * Services can export their ``external_upgrade_tasks`` for the
       persist/restore operations, we'll provide an Ansible module or
       role to make it DRY. The transfer will likely go via undercloud
       initially, but it would be nice to make it direct in order to
       speed it up.

#. **Run the usual upgrade tasks on the newer OS nodes**::

     openstack overcloud upgrade run \
         --limit controller-openstack-0,database-0,messaging-0

   Notes:

   * **Control plane downtime stops at the end of this step.** This
     means the control plane downtime spans two commands. We should
     *not* make it one command because the commands use different
     parts of upgrade framework underneath, and the separation will
     mean easier re-running of individual parts, should they fail.

   * Here we start pcmk cluster and all services on the newer OS
     nodes, using the data previously transferred from the older OS
     nodes.

   * Likely we won't need any special per-service upgrade tasks,
     unless we discover we need some data conversions or
     adjustments. The node will be with all services stopped after
     upgrade to newer OS, so likely we'll be effectively "setting up a
     fresh cloud on pre-existing data".

   * Caution: At this point the newer OS nodes became the authority on
     data state. Do not re-run the previous data transfer step after
     services have started on newer OS nodes.

   * (Currently ``upgrade run`` has ``--nodes`` and ``--roles`` which
     both function the same, as Ansible ``--limit``. Notably, nothing
     stops you from passing role names to ``--nodes`` and vice
     versa. Maybe it's time to retire those two and implement
     ``--limit`` to match the concept from Ansible closely.)

#. **Perform any service-specific && node-specific external upgrades,
   most importantly Ceph**::

     openstack overcloud external-upgrade run \
         --tags system_upgrade_run \
         -e system_upgrade_nodes=controller-openstack-0,database-0,messaging-0

   Notes:

   * Ceph-ansible here runs on a single node and spawns a new version
     of ceph-mon. Per-node run capability will need to be added to
     ceph-ansible.

   * Ceph-mon count is restored here (in most environments, it means
     going from 2 to 3).

#. **Upgrade the remaining control plane nodes**. Perform all the
   previous control plane upgrade steps for the remaining controllers
   too. Two important notes here:

   * **Do not run the ``system_upgrade_transfer_data`` step anymore.**
     The remaining controllers are expected to join the cluster and
     sync the database data from the primary controller via DB
     replication mechanism, no explicit data transfer should be
     necessary.

   * To have the necessary number of ceph-mons running at any given
     time (often that means 2 out of 3), the controllers (ceph-mon
     nodes) should be upgraded one-by-one.

   After this step is finished, all of the nodes which are sensitive
   to Pacemaker version or DB schema version should be upgraded to
   newer OS, newer OpenStack, and newer ceph-mons.

#. **Upgrade the rest of the overcloud nodes** (Compute, Networker,
   CephStorage), **either one-by-one or in batches**, depending on
   uptime requirements of particular nodes. E.g. for computes this
   would mean evacuating and then also running::

     openstack overcloud upgrade run \
         --tags system_upgrade_prepare \
         --limit novacompute-0

     openstack overcloud upgrade run \
         --tags system_upgrade_run \
         --limit novacompute-0

     openstack overcloud upgrade run \
         --limit novacompute-0


   Notes:

   * Ceph OSDs can be removed by the ``external-upgrade run --tags
     system_upgrade_prepare`` step before reprovisioning, and after
     ``upgrade run`` command, ceph-ansible can recreate the OSD via
     the ``external-upgrade run --tags system_upgrade_run`` step,
     always limited to the OSD being upgraded::

       # Remove OSD
       openstack overcloud external-upgrade run \
           --tags system_upgrade_prepare \
           -e system_upgrade_nodes=novacompute-0

       # <<Here the node is reprovisioned and upgraded>>

       # Re-deploy OSD
       openstack overcloud external-upgrade run \
           --tags system_upgrade_run \
           -e system_upgrade_nodes=novacompute-0

#. **Perform online upgrade** (online data migrations) after all nodes
   have been upgraded::

     openstack overcloud external-upgrade run \
         --tags online_upgrade

#. **Perfrom upgrade converge** to re-assert the overcloud state::

     openstack overcloud upgrade converge <DEPLOY ARGS>

#. **Clean up upgrade data persisted on undercloud**::

     openstack overcloud external-upgrade run \
         --tags system_upgrade_cleanup


Additional notes on data persist/restore
----------------------------------------

* There are two different use cases:

  * Persistence for things that need to survive reprovisioning (for
    each node)

  * Transfer of DB data from node to node (just once to bootstrap the
    first new OS node in a role)

* The `synchronize Ansible module`_ shipped with Ansible seems
  fitting, we could wrap it in a role to handle common logic, and
  execute the role via ``include_role`` from
  ``upgrade_tasks``.

* We would persist the temporary data on the undercloud under a
  directory accessible only by the user which runs the upgrade
  playbooks (``mistral`` user). The root dir could be
  ``/var/lib/tripleo-upgrade`` and underneath would be subdirs for
  individual nodes, and one more subdir level for services.

  * (Undercloud's Swift also comes to mind as a potential place for
    storage. However, it would probably add more complexity than
    benefit.)

* **The data persist/restore operations within the upgrade do not
  supplement or replace backup/restore procedures which should be
  performed by the operator, especially before upgrading.** The
  automated data persistence is solely for upgrade purposes, not for
  disaster recovery.


Alternatives
------------

* **Parallel cloud migration.** We could declare the in-place upgrade
  of operating system + OpenStack as too risky and complex and time
  consuming, and recommend standing up a new cloud and transferring
  content to it. However, this brings its own set of challenges.

  This option is already available for anyone whose environment is
  constrained such that normal upgrade procedure is not realistic,
  e.g. in case of extreme uptime requirements or extreme risk-aversion
  environments.

  Implementing parallel cloud migration is probably best handled on a
  per-environment basis, and TripleO doesn't provide any automation in
  this area.

* **Upgrading the operating system separately from OpenStack.** This
  would simplify things on several fronts, but separating the
  operating system upgrade while preserving uptime (i.e. upgrading the
  OS in a rolling fashion node-by-node) currently seems not realistic
  due to:

  * The pacemaker cluster (corosync) limitations mentioned earlier. We
    would have to containerize Pacemaker (even if just ad-hoc
    non-productized image).

  * Either we'd have to make OpenStack (and dependencies) compatible
    with OS releases in a way we currently do not intend, or at least
    ensure such compatibility when running containerized. E.g. for
    data transfer, we could then probably use Galera native
    replication.

  * OS release differences might be too large. E.g. in case of
    differing container runtimes, we might have to make t-h-t be able
    to deploy on two runtimes within one deployment.

* **Upgrading all control plane nodes at the same time as we've been
  doing so far.** This is not entirely impossible, but rebooting all
  controllers at the same time to do the upgrade could mean total
  ceph-mon unavailability. Also given that the upgraded nodes are
  unreachable via ssh for some time, should something go wrong and the
  nodes got stuck in that state, it could be difficult to recover back
  into a working cloud.

  This is probably not realistic, mainly due to concerns around Ceph
  mon availability and risk of bricking the cloud.


Security Impact
---------------

* How we transfer data from older OS machines to newer OS machines is
  a potential security concern.

* The same security concern applies for per-node data persist/restore
  procedure in case we go with reprovisioning.

* The stored data may include overcloud node's secrets and should be
  cleaned up from the undercloud when no longer needed.

* In case of using the `synchronize Ansible module`_: it uses rsync
  over ssh, and we would store any data on undercloud in a directory
  only accessible by the same user which runs the upgrade playbooks
  (``mistral``). This undercloud user has full control over overcloud
  already, via ssh keys authorized for all management operations, so
  this should not constitute a significant expansion of ``mistral``
  user's knowledge/capabilities.


Upgrade Impact
--------------

* The upgrade procedure is riskier and more complex.

  * More things can potentially go wrong.

  * It will take more time to complete, both manually and
    automatically.

* Given that we upgrade one of the controllers while the other two are
  still running, the control plane services downtime could be slightly
  shorter than before.

* When control plane services are stopped on older OS machines and
  running on newer OS machine, we create a window without high
  availability.

* Upgrade framework might need some tweaks but on high level it seems
  we'll be able to fit the workflow into it.

* All the upgrade steps should be idempotent, rerunnable and
  recoverable as much as we can make them so.


Other End User Impact
---------------------

* Floating IP availability could be affected. Neutron upgrade
  procedure typically doesn't immediately restart sidecar containers
  of L3 agent. Restarting will be a must if we upgrade the OS.


Performance Impact
------------------

* When control plane services are stopped on older OS machines and
  running on newer OS machine, only one controller is available to
  serve all control plane requests.

* Depending on role/service composition of the overcloud, the reduced
  throughput could also affect tenant traffic, not just control plane
  APIs.


Other Deployer Impact
---------------------

* Automating such procedure introduces some code which had better not
  be executed by accident. The external upgrade tasks which are tagged
  ``system_upgrade_*`` should also be tagged ``never``, so that they
  only run when explicitly requested.

* For the data transfer step specifically, we may also introduce a
  safety "flag file" on the target overcloud node, which would prevent
  re-running of the data transfer until the file is manually removed.


Developer Impact
----------------

Developers who work on specific composable services in TripleO will
need to get familiar with the new upgrade workflow.


Main Risks
----------

* Leapp has been somewhat explored but its viability/readiness for our
  purpose is still not 100% certain.

* CI testing will be difficult, if we go with Leapp it might be
  impossible (more below).

* Time required to implement everything may not fit within the release
  cycle.

* We have some idea how to do the data persist/restore/transfer parts,
  but some prototyping needs to be done there to gain confidence.

* We don't know exactly what data needs to be persisted during
  reprovisioning.


Implementation
==============

Assignee(s)
-----------

Primary assignees::
  | jistr, chem, jfrancoa

Other contributors::
  | fultonj for Ceph


Work Items
----------

With aditional info in format: (how much do we know about this task,
estimate of implementation difficulty).

* (semi-known, est. as medium) Change tripleo-heat-templates +
  puppet-tripleo to be able to set up a cluster on just one controller
  (with newer OS) while the Heat stack knows about all
  controllers. This is currently not possible.

* (semi-known, est. as medium) Amend upgrade_tasks to work for
  Rocky->Stein with OS upgrade.

* ``system_upgrade_transfer_data``:

  * (unknown, est. as easy) Detect upgraded vs. unupgraded machines to
    transfer data to/from.

  * (known, est. as easy) Stop all services on the unupgraded machines
    transfer data to/from. (Needs to be done via external upgrade
    tasks which is new, but likely not much different from what we've
    been doing.)

  * (semi-known, est. as medium/hard) Implement an Ansible role for
    transferring data from one node to another via undercloud.

  * (unknown, est. as medium) Figure out which data needs transferring
    from old controller to new, implement it using the above Ansible
    role -- we expect only MariaDB to require this, any special
    services should probably be tackled by service squads.

* (semi-known, est. as medium/hard) Implement Ceph specifics, mainly
  how to upgrade one node (mon, OSD, ...) at a time.

* (unknown, either easy or hacky or impossible :) ) Implement
  ``--limit`` for ``external-upgrade run``. (As external upgrade runs
  on undercloud by default, we'll need to use ``delegate_to`` or
  nested Ansible for overcloud nodes. I'm not sure how well --limit
  will play with this.)

* (known, est. as easy) Change update/upgrade CLI from ``--nodes``
  and ``--roles`` to ``--limit``.

* (semi-known, est. as easy/medium) Add ``-e`` variable pass-through
  support to ``external-upgrade run``.

* (unknown, unknown) Test as much as we can in CI -- integrate with
  tripleo-upgrade and OOOQ.

* For reprovisioning:

  * (semi-known, est. as medium) Implement ``openstack overcloud admin
    authorize``. Should take ``--stack``, ``--limit``,
    ``--overcloud-ssh-*`` params.

  * (semi-known, est. as medium/hard) Implement an Ansible role for
    temporarily persisting overcloud nodes' data on the undercloud and
    restoring it.

  * (known, est. as easy) Implement ``external-upgrade run --tags
    system_upgrade_cleanup``.

  * (unknown, est. as hard in total, but should probably be tackled by
    service squads) Figure out which data needs persisting for
    particular services and implement the persistence using the above
    Ansible role.

* For in-place:

  * (semi-known, est. as easy) Calls to Leapp in
    ``system_upgrade_prepare``, ``system_upgrade_run``.

  * (semi-known, est. as medium) Implement a Leapp actor to set up or
    use the repositories we need.

Dependencies
============

* For in-place: Leapp tool being ready to upgrade the OS.

* Changes to ceph-ansible might be necessary to make it possible to
  run it on a single node (for upgrading mons and OSDs node-by-node).


Testing
=======

Testing is one of the main estimated pain areas. This is a traditional
problem with upgrades, but it's even more pronounced for OS upgrades.

* Since we do all the OpenStack infra cloud testing of TripleO on
  CentOS 7 currently, it would make sense to test an upgrade to
  CentOS 8. However, CentOS 8 is nonexistent at the time of writing.

* It is unclear when Leapp will be ready for testing an upgrade from
  CentOS 7, and it's probably the only thing we'd be able to execute
  in CI. The ``openstack server rebuild`` alternative is probably not
  easily executable in CI, at least not in OpenStack infra clouds. We
  might be able to emulate reprovisioning by wiping data.

* Even if we find a way to execute the upgrade in CI, it might still
  take too long to make the testing plausible for validating patches.


Documentation Impact
====================

Upgrade docs will need to be amended, the above spec is written mainly
from the perspective of expected operator workflow, so it should be a
good starting point.


References
==========

* `Leapp`_

* `Leapp actors`_

* `Leapp architecture`_

* `Stein PTG etherpad`_

* `backup and restore`_

* `synchronize Ansible module`_

.. _Leapp: https://leapp-to.github.io/
.. _Leapp actors: https://leapp-to.github.io/actors
.. _Leapp architecture: https://leapp-to.github.io/architecture
.. _Stein PTG etherpad: https://etherpad.openstack.org/p/tripleo-ptg-stein
.. _backup and restore: http://tripleo.org/install/controlplane_backup_restore/00_index.html
.. _synchronize Ansible module: https://docs.ansible.com/ansible/latest/modules/synchronize_module.html
