..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Support Keystoneless Undercloud (basic auth or noauth)
======================================================

The goal of this proposal is to introduce the community to the idea of
removing Keystone from TripleO undercloud and run the remaining OpenStack
services either with basic authentication or noauth (i.e. Standalone mode).


Problem Description
===================

With the goal of having a thin undercloud we've been simplifying the
undercloud architecture since a few cycles and have removed a number
of OpenStack services. After moving to use `network_data_v2`_ and
`ephemeral_heat`_ by default, we are left only with neutron, ironic
and ironic-inspector services.

Keystone authentication and authorization does not add lot of value to the
undercloud. We use `admin` and `admin` project for everything. There are
also few service users (one per service) for communication between services.
Most of the overcloud deployment and configuration is done as the os user.
Also, for large deployments we increase token expiration time to a large
value which is orthogonal to keystone security.


Proposed Change
===============

Overview
--------

At present, we have keystone running in the undercloud providing catalog,
authentication/authorization services to the remaining deployed services
neutron, ironic and ironic-inspector. Ephemeral heat uses a fake keystone
client which does not talk to keystone.

All these remaining services are capabale of running standalone using either
`http_basic` or `noauth` auth_strategy and clients using openstacksdk and
keystoneauth can use `HTTPBasicAuth` or `NoAuth` identity plugins with the
standalone services.

The proposal is to deploy these OpenStack services either with basic auth or
noauth and remove keystone from the undercloud by default.

- Deploy ironic/ironic-inspector/neutron with `http_basic` (default) or `noauth`

This would also allow us to remove some additional services like `memcached`
from the undercloud mainly used for authtoken caching.


Alternatives
------------

- Keep keystone in the undercloud as before.


Security Impact
---------------

There should not be any significant security implications by disabling keystone
on the undercloud as there are no multi-tenancy and RABC requirements for
undercloud users/operators. Deploying baremetal and networking services with `http_basic` authentication would protect against any possible intrusion as before.


Upgrade Impact
--------------

There will be no upgrade impact; this change will be transparent to the
end-user.


Other End User Impact
---------------------

None.


Performance Impact
------------------

Disabling authentication and authorization would make the API calls faster and
the overall resource requirements of undercloud would reduce.


Other Deployer Impact
---------------------

None

Developer Impact
----------------

None.


Implementation
==============

- Add THT support for configuring `auth_strategy` for ironic and neutron
  services and manage htpasswd files used for basic authentication by the
  ironic services.

.. code-block:: yaml

        IronicAuthStrategy: http_basic
        NeutronAuthStrategy: http_basic

- Normally, Identity service middleware provides a X-Project-Id header based on
  the authentication token submitted by the service client. However when keystone
  is not available neutron expects `project_id` in the `POST` requests (i.e create
  API). Also, metalsmith communicates with `neutron` to create `ctlplane` ports for
  instances.

  Add a middleware for neutron API `http_basic` pipeline to inject a fake project_id
  in the context.

- Add basic authentication middleware to oslo.middleware and use it for undercloud
  neutron.

- Create/Update clouds.yaml to use `auth_type: http_basic` and use endpoint overrides
  for the public endpoints with `<service_name>_endpoint_override` entries. We
  would leverage the `EndpointMap` and change `extraconfig/post_deploy` to create
  and update clouds.yaml.

.. code-block:: yaml

        clouds:
          undercloud:
            auth:
              password: piJsuvz3lKUtCInsiaQd4GZ1w
              username: admin
            auth_type: http_basic
            baremetal_api_version: '1'
            baremetal_endpoint_override: https://192.168.24.2:13385
            baremetal_introspection_endpoint_override: https://192.168.24.2:13050
            network_api_version: '2'
            network_endpoint_override: https://192.168.24.2:13696

Assignee(s)
-----------

Primary assignee:
  ramishra

Other contributors:


Work Items
----------

- Add basic authentication middleware in oslo.middleware
  https://review.opendev.org/c/openstack/oslo.middleware/+/802234
- Support `auth_strategy` with ironic and neutron services
  https://review.opendev.org/c/openstack/tripleo-heat-templates/+/798241
- Neutron middleware to add fake project_id to noauth pipleline
  https://review.opendev.org/c/openstack/neutron/+/799162
- Configure neutron paste deploy for basic authentication
  https://review.opendev.org/c/openstack/tripleo-heat-templates/+/804598
- Disable keystone by default
  https://review.opendev.org/c/openstack/tripleo-heat-templates/+/794912
- Add option to enable keystone if required
  https://review.opendev.org/c/openstack/python-tripleoclient/+/799409
- Other patches:
  https://review.opendev.org/c/openstack/tripleo-ansible/+/796991
  https://review.opendev.org/c/openstack/tripleo-common/+/796825
  https://review.opendev.org/c/openstack/tripleo-ansible/+/797381
  https://review.opendev.org/c/openstack/tripleo-heat-templates/+/799408


Dependencies
============

Ephemeral heat and network-data-v2 are used as defaults.


Documentation Impact
====================

Update the undercloud installation and upgrade guides.


References
==========

* `network_data_v2`_ specification
* `ephemeral_heat`_ specification

.. _network_data_v2: https://specs.openstack.org/openstack/tripleo-specs/specs/wallaby/triplo-network-data-v2-node-ports.html
.. _ephemeral_heat: https://specs.openstack.org/openstack/tripleo-specs/specs/wallaby/ephemeral-heat-overcloud.html
