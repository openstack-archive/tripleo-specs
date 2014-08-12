..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Enable Neutron DVR on overcloud in TripleO
==========================================

https://blueprints.launchpad.net/tripleo/+spec/support-neutron-dvr

Neutron distributed virtual routing should be able to be configured in TripleO.


Problem Description
===================

To be able to enable distributed virtual routing in Neutron there needs to be
several changes to the current TripleO overcloud deployment.  The overcloud
compute node(s) are constructed with the ``neutron-openvswitch-agent`` image
element, which provides the ``neutron-openvswitch-agent`` on the compute node.
In order to support distributed virtual routing, the compute node(s) must also
have the ``neutron-metadata-agent`` and ``neutron-l3-agent`` installed. The
installation of the ``neutron-l3-agent`` and ``neutron-dhcp-agent`` will need
also to be decoupled.

Additionally, for distributed virtual routing to be enabled, the
``neutron.conf``, ``l3_agent.ini`` and ``ml2_conf.ini`` all need to have
additional settings.

Proposed Change
===============

In the tripleo-image-elements, move the current ``neutron-network-node`` element
to an element named ``neutron-router``, which will be responsible for doing the
installation and configuration work required to install the ``neutron-l3-agent``
and the ``neutron-metadata-agent``. This ``neutron-router`` element will list
the ``neutron-openvswitch-agent`` in its element-deps.  The ``neutron-network
-node`` element will then become simply a 'wrapper' whose sole purpose is to list
the dependencies required for a network node (neutron, ``neutron-dhcp-agent``,
``neutron-router``, os-refresh-config).

Additionally, in the tripleo-image-elements/neutron element, the
``neutron.conf``, ``l3_agent.ini`` and ``plugins/ml2/ml2_conf.ini`` will be
modified to add the configuration variables required in each to support
distributed virtual routing (the required configuration variables are listed at
https://wiki.openstack.org/wiki/Neutron/DVR/HowTo#Configuration).

In the tripleo-heat-templates, the ``nova-compute-config.yaml``
``nova-compute-instance.yaml`` and ``overcloud-source.yaml`` files will be
modified to provide the correct settings for the new distributed virtual routing
variables.  The enablement of distributed virtual routing will be determined by
a 'NeutronDVR' variable which will be 'False' by default (distributed virtual
routing not enabled) for backward compatibility, but can be set to 'True' if
distributed virtual routing is desired.

Lastly, the tripleo-incubator script ``devtest_overcloud.sh`` will be modified
to: a) build the overcloud-compute disk-image with ``neutron-router`` rather
than with ``neutron-openvswitch-agent``, and b) configure the appropriate
parameter values to be passed in to the heat stack create for the overcloud so
that distributed routing is either enabled or disabled.

Alternatives
------------

We could choose to make no change to the ``neutron-router`` image-element and
it can be included as well in the list of elements arguments to the disk image
build for compute nodes.  This has the undesired effect of also
including/configuring and starting the ``neutron-dhcp-agent`` on each compute
node.  Alternatively, it is possible to keep the ``neutron-network-node``
element as it is and create a ``neutron-router`` element which is a copy of
most of the element contents of the ``neutron-network-node`` element but without
the dependency on the ``neutron-dhcp-agent`` element.  This approach would
introduce a significant amount of code duplication.

Security Impact
---------------

Although TripleO installation does not use FWaaS, enablement of DVR currently
is known to break FWaaS.
See https://blueprints.launchpad.net/neutron/+spec/neutron-dvr-fwaas

Other End User Impact
---------------------

The user will have the ability to set an environment variable during install
which will determine whether distributed virtual routing is enabled or not.

Performance Impact
------------------

None identified

Other Deployer Impact
---------------------

The option to enable or disable distributed virtual routing at install time will
be added.  By default distributed virtual routing will be disabled.

Developer Impact
----------------

None identified

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Erik Colnick (erikcolnick on Launchpad)
Other contributors:
  None

Work Items
----------

 * Create ``neutron-router`` element in tripleo-image-elements and move related
   contents from ``neutron-network-node`` element.  Remove the
   ``neutron-dhcp-agent`` dependency from the element-deps of the
   ``neutron-router`` element.

 * Add the ``neutron-router`` element as a dependency in the
   ``neutron-network-node`` ``element-deps`` file.  The ``element-deps``
   file becomes the only content in the ``neutron-network-node`` element.

 * Add the configuration values indicated in
   https://wiki.openstack.org/wiki/Neutron/DVR/HowTo#Configuration to the
   ``neutron.conf``, ``l3_agent.ini`` and ``ml2_conf.ini`` files in the
   ``neutron`` image element.

 * Add the necessary reference variables to the ``nova-compute-config.yaml`` and
   ``nova-compute-instance.yaml`` tripleo-heat-templates files in order to be
   able to set the new variables in the config files (from above item).  Add
   definitions and default values in ``overcloud-source.yaml``.

 * Modify tripleo-incubator ``devtest_overcloud.sh`` script to set the
   appropriate environment variables which will drive the configuration of
   neutron on the overcloud to either enable distributed virtual routers or
   disable distributed virtual routers (with disable as the default).

Dependencies
============

None

Testing
=======

Existing TripleO CI will help ensure that as this is implemented, the current
feature set is not impacted and that the default behavior of disabled
distributed virtual routers is maintained.

Additional CI tests which test the installation with distributed virtual
routers should be added as this implementation is completed.

Documentation Impact
====================

Documentation of the new configuration option will be needed.

References
==========

