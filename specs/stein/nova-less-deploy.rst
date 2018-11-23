..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Provision nodes without Nova and Glance
=======================================

https://blueprints.launchpad.net/tripleo/+spec/nova-less-deploy

Currently TripleO undercloud uses Heat, Nova, Glance, Neutron and Ironic for
provisioning bare metal machines. This blueprint proposes excluding Heat, Nova
and Glance from this flow, removing Nova and Glance completely from the
undercloud.

Problem Description
===================

Making TripleO workflows use Ironic directly to provision nodes has quite a few
benefits:

#. First and foremost, getting rid of the horrible "no valid hosts found"
   exception. The scheduling will be much simpler and the errors will be
   clearer.

   .. note::
      This and many other problems with using Nova in the undercloud come from
      the fact that Nova is cloud-oriented software, while the undercloud is
      more of a traditional installer. In the "pet vs cattle" metaphore, Nova
      handles the "cattle" case, while the undercloud is the "pet" case.

#. Also important for the generic provisioner case, we'll be able to get rid of
   Nova and Glance, reducing the memory footprint.

#. We'll get rid of pre-deploy validations that currently try to guess what
   Nova scheduler will expect.

#. We'll be able to combine nodes deployed by Ironic with pre-deployed servers.

#. We'll become in charge of building the configdrive, potentially putting more
   useful things there.

#. Hopefully, scale-up will be less error-prone.

Also in the future we may be able to:

#. Integrate things like building RAID on demand much easier.

#. Use introspection data in scheduling and provisioning decisions.
   Particularly, we can automate handling root device hints.

#. Make Neutron optional and use static DHCP and/or *os-net-config*.

Proposed Change
===============

Overview
--------

This blueprint proposes removal replacing the triad Heat-Nova-Glance with
Ironic driven directly by Mistral. To avoid placing Ironic-specific code into
tripleo-common, a new library metalsmith_ has been developed and accepted into
the Ironic governance.

As part of the implementation, this blueprint proposes completely separting the
bare metal provisioning process from software configuration, including the CLI
level. This has two benefits:

#. Having a clear separation between two error-prone processes simplifies
   debugging for operators.

#. Reusing the existing *deployed-server* workflow simplifies the
   implementation.

In the distant future, the functionality of metalsmith_ may be moved into
Ironic API itself. In this case it will be phased out, while keeping the same
Mistral workflows.

Operator workflow
-----------------

As noted in Overview_, the CLI/GUI workflow will be split into hardware
provisioning and software configuration parts (the former being optional).

#. In addition to existing Heat templates, a new file
   baremetal_deployment.yaml_ will be populated by an operator with the bare
   metal provisioning information.

#. Bare metal deployment will be conducted by a new CLI command or GUI
   operation using the new `deploy_roles workflow`_::

      openstack overcloud node provision \
         -o baremetal_environment.yaml baremetal_deployment.yaml

   This command will take the input from baremetal_deployment.yaml_, provision
   requested bare metal machines and output a Heat environment file
   baremetal_environment.yaml_ to use with the *deployed-server* feature.

#. Finally, the regular deployment is done, including the generated file::

      openstack overcloud deploy \
         <other cli arguments> \
         -e baremetal_environment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-bootstrap-environment-centos.yaml \
         -r /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-server-roles-data.yaml

For simplicity the two commands can be combined::

      openstack overcloud deploy \
         <other cli arguments> \
         -b baremetal_deployment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-environment.yaml \
         -e /usr/share/openstack-tripleo-heat-templates/environments/deployed-server-bootstrap-environment-centos.yaml \
         -r /usr/share/openstack-tripleo-heat-templates/deployed-server/deployed-server-roles-data.yaml

The new argument ``--baremetal-deployment``/``-b`` will accept the
baremetal_deployment.yaml_ and do the deployment automatically.

Breakdown of the changes
------------------------

This section describes the required changes in depth.

Image upload
~~~~~~~~~~~~

As Glance will no longer be used, images will have to be served from other
sources. Ironic supports HTTP and file sources from its images. For the
undercloud case, the file source seems to be the most straightforward, also the
*Edge* case may require using HTTP images.

To make both cases possible, the ``openstack overcloud image upload`` command
will now copy the three overcloud images (``overcloud-full.qcow2``,
``overcloud-full.kernel`` and ``overcloud-full.ramdisk``) to
``/var/lib/ironic/httpboot/overcloud-images``. This will allow referring to
images both via ``file:///var/lib/ironic/httpboot/overcloud.images/...`` and
``http(s)://<UNDERCLOUD HOST>:<IPXE PORT>/overcloud-images/...``.

Finally, a checksum file will be generated from the copied images using::

   cd /var/lib/ironic/httpboot/overcloud-images
   md5sum overcloud-full.* > MD5SUMS

This is required since the checksums will no longer come from Glance.

baremetal_deployment.yaml
~~~~~~~~~~~~~~~~~~~~~~~~~

This file will describe which the bare metal provisioning parameters. It will
provide the information that is currently implicitly deduced from the Heat
templates.

.. note::
   We could continue extracting it from the templates well. However, a separate
   file will avoid a dependency on any Heat-specific logic, potentially
   benefiting standalone installer cases. It also provides the operators with
   more control over the provisioning process.

The format of this file resembles one of the ``roles_data`` file. It describes
the deployment parameters for each role. The file contains a list of roles,
each with a ``name``. Other accepted parameters are:

``count``
   number of machines to deploy for this role. Defaults to 1.
``profile``
   profile (``compute``, ``control``, etc) to use for this role. Roughly
   corresponds to a flavor name for a Nova based deployment. Defaults to no
   profile (any node can be picked).
``hostname_format``
   a template for generating host names. This is similar to
   ``HostnameFormatDefault`` of a ``roles_data`` file and should use
   ``%index%`` to number the nodes. The default is ``%stackname%-<role name in
   lower case>-%index%``.
``instances``
   list of instances in the format accepted by `deploy_instances workflow`_.
   This allows to tune parameters per instance.

Examples
^^^^^^^^

Deploy one compute and one control with any profile:

.. code-block:: yaml

   - name: Compute
   - name: Controller

HA deployment with two computes and profile matching:

.. code-block:: yaml

   - name: Compute
     count: 2
     profile: compute
     hostname_format: compute-%index%.example.com
   - name: Controller
     count: 3
     profile: control
     hostname_format: controller-%index%.example.com

Advanced deployment with custom hostnames and parameters set per instance:

.. code-block:: yaml

   - name: Compute
     profile: compute
     instances:
       - hostname: compute-05.us-west.example.com
         nics:
           - network: ctlplane
             fixed_ip: 10.0.2.5
         traits:
           - HW_CPU_X86_VMX
       - hostname: compute-06.us-west.example.com
         nics:
           - network: ctlplane
             fixed_ip: 10.0.2.5
         traits:
           - HW_CPU_X86_VMX
   - name: Controller
     profile: control
     instances:
       - hostname: controller-1.us-west.example.com
         swap_size_mb: 4096
       - hostname: controller-2.us-west.example.com
         swap_size_mb: 4096
       - hostname: controller-3.us-west.example.com
         swap_size_mb: 4096

deploy_roles workflow
~~~~~~~~~~~~~~~~~~~~~

The workflow ``tripleo.baremetal_deploy.v1.deploy_roles`` will accept the
information from baremetal_deployment.yaml_, convert it into the low-level
format accepted by the `deploy_instances workflow`_ and call the
`deploy_instances workflow`_ with it.

It will accept the following mandatory input:

``roles``
   parsed baremetal_deployment.yaml_ file.

It will accept one optional input:

``plan``
   plan/stack name, used for templating. Defaults to ``overcloud``.

It will return the same output as the `deploy_instances workflow`_ plus:

``environment``
   the content of the generated baremetal_environment.yaml_ file.

Examples
^^^^^^^^

The examples from baremetal_deployment.yaml_ will be converted to:

.. code-block:: yaml

   - hostname: overcloud-compute-0
   - hostname: overcloud-controller-0

.. code-block:: yaml

   - hostname: compute-0.example.com
     profile: compute
   - hostname: compute-1.example.com
     profile: compute
   - hostname: controller-0.example.com
     profile: control
   - hostname: controller-1.example.com
     profile: control
   - hostname: controller-2.example.com
     profile: control

.. code-block:: yaml

   - hostname: compute-05.us-west.example.com
     nics:
       - network: ctlplane
         fixed_ip: 10.0.2.5
     profile: compute
     traits:
       - HW_CPU_X86_VMX
   - hostname: compute-06.us-west.example.com
     nics:
       - network: ctlplane
         fixed_ip: 10.0.2.5
     profile: compute
     traits:
       - HW_CPU_X86_VMX
   - hostname: controller-1.us-west.example.com
     profile: control
     swap_size_mb: 4096
   - hostname: controller-2.us-west.example.com
     profile: control
     swap_size_mb: 4096
   - hostname: controller-3.us-west.example.com
     profile: control
     swap_size_mb: 4096

deploy_instances workflow
~~~~~~~~~~~~~~~~~~~~~~~~~

The workflow ``tripleo.baremetal_deploy.v1.deploy_instances`` is a thin wrapper
around the corresponding metalsmith_ calls.

The following inputs are mandatory:

``instances``
   list of requested instances in the format described in `Instance format`_.
``ssh_keys``
   list of SSH public keys contents to put on the machines.

The following inputs are optional:

``ssh_user_name``
   SSH user name to create, defaults to ``heat-admin`` for compatibility.
``timeout``
   deployment timeout, defaults to 3600 seconds.
``concurrency``
   deployment concurrency - how many nodes to deploy at the same time. Defaults
   to 20, which matches introspection.

Instance format
^^^^^^^^^^^^^^^

The instance record format closely follows one of the `metalsmith ansible
role`_ with only a few TripleO-specific additions and defaults changes.

Either or both of the following fields must be present:

``hostname``
   requested hostname. It is used to identify the deployed instance later on.
   Defaults to ``name``.
``name``
   name of the node to deploy on. If ``hostname`` is not provided, ``name`` is
   also used as the hostname.

The following fields will be supported:

``capabilities``
   requested node capabilities (except for ``profile`` and ``boot_option``).
``conductor_group``
   requested node's conductor group. This is primary for the *Edge* case when
   nodes managed by the same Ironic can be physically separated.
``nics``
   list of requested NICs, see metalsmith_ documentation for details. Defaults
   to ``{"network": "ctlplane"}`` which requests creation of a port on the
   ``ctlplane`` network.
``profile``
   profile to use (e.g. ``compute``, ``control``, etc).
``resource_class``
   requested node's resource class, defaults to ``baremetal``.
``root_size_gb``
   size of the root partition in GiB, defaults to 49.
``swap_size_mb``
   size of the swap partition in MiB, if needed.
``traits``
   list of requested node traits.
``whole_disk_image``
   boolean, whether to treat the image (``overcloud-full.qcow2`` or provided
   through the ``image`` field) as a whole disk image. Defaults to false.

The following fields will be supported, but the defaults should work for all
but the most extreme cases:

``image``
   file or HTTP URL of the root partition or whole disk image.
``image_kernel``
   file or HTTP URL of the kernel image (partition images only).
``image_ramdisk``
   file or HTTP URL of the ramdisk image (partition images only).
``image_checksum``
   checksum of URL of checksum of the root partition or whole disk image.

Certificate authority configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If TLS is used in the undercloud, we need to make the nodes trust
the Certificate Authority (CA) that signed the TLS certificates.
If ``/etc/pki/ca-trust/source/anchors/cm-local-ca.pem`` exists, it will be
included in the generated configdrive, so that the file is copied into the same
location on target systems.

Outputs
^^^^^^^

The workflow will provide the following outputs:

``ctlplane_ips``
   mapping of host names to their respective IP addresses on the ``ctlplane``
   network.
``instances``
   mapping of host names to full instance representations with fields:

   ``node``
      Ironic node representation.
   ``ip_addresses``
      mapping of network names to list of IP addresses on them.
   ``hostname``
      instance hostname.
   ``state``
      `metalsmith instance state`_.
   ``uuid``
      Ironic node uuid.

Also two subdicts of ``instances`` are provided:

``existing_instances``
   only instances that already existed.
``new_instances``
   only instances that were deployed.

.. note::
   Instances are distinguised by their hostnames.

baremetal_environment.yaml
~~~~~~~~~~~~~~~~~~~~~~~~~~

This file will serve as an output of the bare metal provisioning process. It
will be fed into the overcloud deployment command. Its goal is to provide
information for the *deployed-server* workflow.

The file will contain the ``HostnameMap`` generated from role names and
hostnames, e.g.

.. code-block:: yaml

   parameter_defaults:
     HostnameMap:
       overcloud-controller-0: controller-1.us-west.example.com
       overcloud-controller-1: controller-2.us-west.example.com
       overcloud-controller-2: controller-3.us-west.example.com
       overcloud-novacompute-0: compute-05.us-west.example.com
       overcloud-novacompute-1: compute-06.us-west.example.com

undeploy_instances workflow
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The workflow ``tripleo.baremetal_deploy.v1.undeploy_instances`` will take a
list of hostnames and undeploy the corresponding nodes.

Novajoin replacement
--------------------

The *novajoin* service is currently used to enroll nodes into IPA and provide
them with TLS certificates. Unfortunately, it has hard dependencies on Nova,
Glance and Metadata API, even though the information could be provided via
other means. Actually, the metadata API cannot always be provided with Ironic
(notably, it may not be available when using isolated provisioning networks).

A potential solution is to provide the required information via a configdrive,
and make the nodes register themselves instead.

Alternatives
------------

* Do nothing, continue to rely on Nova and work around cases when it does
  match our goals well. See `Problem Description`_ for why it is not desired.

* Avoid metalsmith_, use OpenStack Ansible modules or Bifrost. They currently
  lack features (such as VIF attach/detach API) and do not have any notion of
  scheduling. Implementing sophisticated enough scheduling in pure Ansible
  seems a serious undertaking.

* Avoid Mistral, drive metalsmith_ via Ansible. This is a potential future
  direction of this work, but currently it seems much simpler to call
  metalsmith_ Python API from Mistral actions. We would anyway need Mistral (
  (or Ansible Tower) to drive Ansible, because we need some API level.

* Remove Neutron in the same change. Would reduce footprint even further, but
  some operators may find the presence of an IPAM desirable. Also setting up
  static DHCP would increase the scope of the implementation substantially and
  complicate the upgrade even further.

* Keep Glance but remove Nova. Does not make much sense, since Glance is only a
  requirement because of Nova. Ironic can deploy from HTTP or local file
  locations just as well.

Security Impact
---------------

* Overcloud images will be exposed to unauthenticated users via HTTP. We need
  to communicate it clearly that secrets must not be built into images in plain
  text and should be delivered via *configdrive* instead. If it proves
  a problem, we can limit ourselves to providing images via local files.

  .. note::
   This issue exists today, as images are transferred via insecure medium in
   all supported deploy methods.

* Removing two services from the undercloud will reduce potential attack
  surface and simplify audit.

Upgrade Impact
--------------

The initial version of this feature will be enabled for new deployments only.

The upgrade procedure will happen within a release, not between releases.
It will go roughly as follows:

#. Upgrade to a release where undercloud without Nova and Glance is supported.

#. Make a full backup of the undercloud.

#. Run ``openstack overcloud image upload`` to ensure that the
   ``overcloud-full`` images are available via HTTP(s).

The next steps will probably be automated via an Ansible playbook or a Mistral
workflow:

#. Mark deployed nodes *protected* in Ironic to prevent undeploying them
   by mistake.

#. Run a Heat stack update replacing references to Nova servers with references
   to deployed servers. This will require telling Heat not to remove the
   instances.

#. Mark nodes as managed by *metalsmith* (optional, but simplifies
   troubleshooting).

#. Update node's ``instance_info`` to refer to images over HTTP(s).

   .. note:: This may require temporary moving nodes to maintenance.

#. Run an undercloud update removing Nova and Glance.

Other End User Impact
---------------------

* Nova CLI will no longer be available for troubleshooting. It should not be a
  big problem in reality, as most of the problems it is used for are caused by
  using Nova itself.

  metalsmith_ provides a CLI tool for troubleshooting and advanced users. We
  will document using it for tasks like determining IP addresses of nodes.

* It will no longer be possible to update images via Glance API, e.g. from GUI.
  It should not be a bit issue, as most of users use pre-built images. Advanced
  operators are likely to resort to CLI anyway.

* *No valid host found* error will no longer be seen by operators. metalsmith_
  provides more detailed errors, and is less likely to fail because of its
  scheduling approach working better with the undercloud case.

Performance Impact
------------------

* A substantial speed-up is expected for deployments because of removing
  several layers of indirection. The new deployment process will also fail
  faster if the scheduling request cannot be satisfied.

* Providing images via local files will remove the step of downloading them
  from Glance, providing even more speed-up for larger images.

* An operator will be able to tune concurrency of deployment via CLI arguments
  or GUI parameters, other than ``nova.conf``.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

New features for bare metal provisioning will have to be developed with this
work in mind. It may mean implementing something in metalsmith_ code instead of
relying on Nova servers or flavors, or Glance images.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Dmitry Tantsur, IRC: dtantsur, LP: divius

Work Items
----------

Phase 1 (Stein, technical preview):

#. Update ``openstack overcloud image upload`` to copy images into the HTTP
   location and generate checksums.

#. Implement `deploy_instances workflow`_ and `undeploy_instances workflow`_.

#. Update validations to not fail if Nova and/or Glance are not present.

#. Implement `deploy_roles workflow`_.

#. Provide CLI commands for the created workflows.

#. Provide an experimental OVB CI job exercising the new approach.

Phase 2 (T+, fully supported):

#. Update ``openstack overcloud deploy`` to support the new workflow.

#. Support scaling down.

#. Provide a `Novajoin replacement`_.

#. Provide an upgrade workflow.

#. Consider deprecating provisioning with Nova and Glance.

Dependencies
============

* metalsmith_ library will be used for easier access to Ironic+Neutron API.

Testing
=======

Since testing this feature requires bare metal provisioning, a new OVB job will
be created for it. Initially it will be experimental, and will move to the
check queue before the feature is considered fully supported.

Documentation Impact
====================

Documentation will have to be reworked to explain the new deployment approach.
Troubleshooting documentation will have to be updated.

References
==========

.. _metalsmith: https://docs.openstack.org/metalsmith/latest/
.. _metalsmith ansible role: https://docs.openstack.org/metalsmith/latest/user/ansible.html#instance
.. _metalsmith instance state: https://docs.openstack.org/metalsmith/latest/reference/api/metalsmith.html#metalsmith.Instance.state
