..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Network configuration
=====================

Network configuration for the TripleO GUI

Problem Description
===================

Currently, it's not possible to make advanced network configurations using the
TripleO GUI.

Proposed Change
===============

Overview
--------

In the GUI, we will provide a wizard to guide the user through configuring the
networks of their deployment.  The user will be able to assign networks to
roles, and configure additional network parameters.  We will use the
``network_data.yaml`` in the `TripleO Heat Templates`_.   The idea is to expose
the data in ``network_data.yaml`` via the web interface.

In addition to the wizard, we will implement a dynamic network topology diagram
to visually present the configured networks.  This will enable the Deployer to
quickly validate their work.  The diagram will rely on ``network_data.yaml``
and ``roles_data.yaml`` for the actual configuration.

For details, please see the `wireframes`_.

.. _wireframes: https://openstack.invisionapp.com/share/UM87J4NBQ#/screens
.. _TripleO Heat Templates: https://review.openstack.org/#/c/409921/

Alternatives
------------

As an alternative, heat templates can be edited manually to allow customization
before uploading.

Security Impact
---------------

The Deployer could accidentally misconfigure the network topology, and thereby
cause data to be exposed.

Other End User Impact
---------------------

Performance Impact
------------------

The addition of the configuration wizard and the network topology diagram should
have no performance impact on the amount of time needed to run a deployment.

Other Deployer Impact
---------------------

Developer Impact
----------------

As with any new substantial feature, the impact on the developer is cognitive.
We will have to gain a detail understanding of network configuration in
``network_data.yaml``.  Also, testing will add overhead on our efforts.

Implementation
==============

We can proceed with implementation immediately.

Assignee(s)
-----------

Primary assignee:
  hpokorny

Work Items
----------

* Network configuration wizard
  - Reading data from the backend
  - Saving changes
  - UI based on wireframes
* Network topology diagram
  - Investigate suitable javascript libraries
  - UI based on wireframes

Dependencies
============

* The presence of ``roles_data.yaml`` and ``network_data.yaml`` in the plan
* A javascript library for drawing the diagram

Testing
=======

Testing shouldn't pose any real challenges with the exception of the network
topology diagram rendering.  At best, this is currently unknown as it depends on
the chosen javascript library.  Verifying that the correct diagram is displayed
using automated testing might be non-trivial.

Documentation Impact
====================

We should document the new settings introduced by the wizard.  The documentation
should be transferable between the heat template project, and TripleO UI.

References
==========
