..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
PKI management of the overcloud using Certmonger
================================================

There is currently support for enabling SSL for the public endpoints of the
OpenStack services. However, certain use cases require the availability of SSL
everywhere. This spec proposes an approach to enable it.

Problem Description
===================

Even though there is support for deploying both the overcloud and the
undercloud with TLS/SSL support for the public endpoints, there are deployments
that demand the usage of encrypted communications through all the interfaces.

The current approach for deploying SSL in TripleO is to inject the needed
keys/certificates through Heat environment files; this requires the
pre-creation of those. While this approach works for the public-facing
services, as we attempt to secure the communication between different
services, and in different levels of the infrastructure, the amount of keys
and certificates grows. So, getting the deployer to generate all the
certificates and manage them will be quite cumbersome.

On the other hand, TripleO is not meant to handle the PKI of the cloud. And
being the case that we will at some point need to enable the deployer to be
able to renew, revoke and keep track of the certificates and keys deployed in
the cloud, we are in need of a system with such capabilities.

Instead of brewing an OpenStack-specific solution ourselves. I propose the
usage of already existing systems that will make this a lot easier.

Proposed Change
===============

Overview
--------

The proposal is to start using certmonger[1] in the nodes of the overcloud to
interact with a CA for managing the certificates that are being used. With this
tool, we can request the fetching of the needed certificates for interfaces
such as the internal OpenStack endpoints, the database cluster and the message
broker for the cloud. Those certificates will in turn have automatic tracking,
and for cases where there is a certificate to identify the node, it could
even automatically request a renewal of the certificate when needed.

Certmonger is already available in several distributions (both Red Hat or
Debian based) and has the capability of interacting with several CAs, so if the
operator already has a working one, they could use that. On the other hand,
certmonger has the mechanism for registering new CAs, and executing scripts
(which are customizable) to communicate with those CAs. Those scripts are
language independent. But for means of the open source community, a solution
such as FreeIPA[2] or Dogtag[3] could be used to act as a CA and handle the
certificates and keys for us. Note that it's possible to write a plugin for
certmonger to communicate with Barbican or another CA, if that's what we would
like to go for.

In the FreeIPA case, this will require a full FreeIPA system running either on
another node in the cluster or in the undercloud in a container[4].

For cases where the services are terminated by HAProxy, and the overcloud being
in an HA-deployment, the controller nodes will need to share a certificate that
HAProxy will present when accessed. In this case, the workflow will be as
following:

#. Register the undercloud as a FreeIPA client. This configures the kerberos
   environment and provides access control to the undercloud node.
#. Get keytab (credentials) corresponding to the undercloud in order to access
   FreeIPA, and be able to register nodes.
#. Create a HAProxy service
#. Create a certificate/key for that service
#. Store the key in FreeIPA's Vault.
#. Create each of the controllers to be deployed as hosts in FreeIPA (Please
   see note about this)
#. On each controller node get the certificate from service entry.
#. Fetch the key from the FreeIPA vault.
#. Set certmonger to keep track of the resulting certificates and
   keys.

.. note::

    While the process of creating each node beforehand could sound cumbersome,
    this can be automated to increase usability. The proposed approach is to
    have a nova micro-service that automatically registers the nodes from the
    overcloud when they are created [5]. This hook will not only register the
    node in the system, but will also inject an OTP which the node can use to
    fetch the required credentials and get its corresponding certificate and
    key. The aforementioned OTP is only used for enrollment. Once enrollment
    has already taken place, certmonger can already be used to fetch
    certificates from FreeIPA.

    However, even if this micro-service is not in place, we could pass the OTP
    via the TripleO Heat Templates (in the overcloud deployment). So it is
    possible to have the controllers fetching their keytab and subsequently
    request their certificates even if we don't have auto-enrollment in place.

.. note::

    Barbican could also be used instead of FreeIPA's Vault. With the upside of
    it being an already accepted OpenStack service. However, Barbican will also
    need to have a backend, which might be Dogtag in our case, since having an
    HSM for the CI will probably not be an option.

Now, for services such as the message broker, where an individual certificate
is required per-host, the process is much simpler, since the nodes will have
already been registered in FreeIPA and will be able to fetch their credentials.
Now we can just let certmonger do the work and request, and subsequently track
the appropriate certificates.

Once the certificates and keys are present in the nodes, then we can let the
subsequent steps of the overcloud deployment process take place; So the
services will be configured to use those certificates and enable TLS where the
deployer specifies it.

Alternatives
------------

The alternative is to take the same approach as we did for the public
endpoints. Which is to simply inject the certificates and keys to the nodes.
That would have the downside that the certificates and keys will be pasted in
heat environment files. This will be problematic for services such as RabbitMQ,
where we are giving a list of nodes for communication, because to enable SSL in
it, we need to have a certificate per-node serving as a message broker.
In this case two approaches could be taken:

* We will need to copy and paste each certificate and key that is needed for
  each of the nodes. With the downside being how much text needs to be copied,
  and the difficulty of keeping track of the certificates. On the other hand,
  each time a node is removed or added, we need to make sure we remember to add
  a certificate and a key for it in the environment file. So this becomes a
  scaling and a usability issue too.

* We could also give in an intermediate certificate, and let TripleO create the
  certificates and keys per-service. However, even if this fixes the usability
  issue, we still cannot keep track of the specific certificates and keys that
  are being deployed in the cloud.

Security Impact
---------------

This approach enables better security for the overcloud, as it not only eases
us to enable TLS everywhere (if desired) but it also helps us keep track and
manage our PKI. On the other hand, it enables other means of security, such as
mutual authentication. In the case of FreeIPA, we could let the nodes have
client certificates, and so they would be able to authenticate to the services
(as is possible with tools such as HAProxy or Galera/MySQL). However, this can
come as subsequent work of this.

Other End User Impact
---------------------

For doing this, the user will need to pass extra parameters to the overcloud
deployment, such as the CA information. In the case of FreeIPA, we will need to
pass the host and port, the kerberos realm, the kerberos principal of the
undercloud and the location of the keytab (the credentials) for the undercloud.

However, this will be reflected in the documentation.

Performance Impact
------------------

Having SSL everywhere will degrade the performance of the overcloud overall, as
there will be some overhead in each call. However, this is a known issue and
this is why SSL everywhere is optional. It should only be enabled for deployers
that really need it.

The usage of an external CA or FreeIPA shouldn't impact the overcloud
performance, as the operations that it will be doing are not recurrent
operations (issuing, revoking or renewing certificates).

Other Deployer Impact
---------------------

If a deployer wants to enable SSL everywhere, they will need to have a working
CA for this to work. Or if they don't they could install FreeIPA in a node.

Developer Impact
----------------

Discuss things that will affect other developers working on OpenStack.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jaosorior


Work Items
----------

* Enable certmonger and the FreeIPA client tools in the overcloud image
  elements.

* Include the host auto-join hook for nova in the undercloud installation.

* Create nested templates that will be used in the existing places for the
  NodeTLSData and NodeTLSCAData. These templates will do the certmonger
  certificate fetching and tracking.

* Configure the OpenStack internal endpoints to use TLS and make this optional
  through a heat environment.

* Configure the Galera/MySQL cluster to use TLS and make this optional through
  a heat environment.

* Configure RabbitMQ to use TLS (which means having a certificate for each
  node) and make this optional through a heat environment

* Create a CI gate for SSL everywhere. This will include a FreeIPA installation
  and it will enable SSL for all the services, ending in the running of a
  pingtest. For the FreeIPA preparations, a script running before the overcloud
  deployment will add the undercloud as a client, configure the appropriate
  permissions for it and deploy a keytab so that it can use the nova hook.
  Subsequently it will create a service for the OpenStack internal endpoints,
  and the database, which it will use to create the needed certificates and
  keys.


Dependencies
============

* This requires the following bug to be fixed in Nova:
  https://bugs.launchpad.net/nova/+bug/1518321

* Also requires the packaging of the nova hook.


Testing
=======

We will need to create a new gate in CI to test this.


Documentation Impact
====================

The documentation on how to use an external CA and how to install and use
FreeIPA with TripleO needs to be created.


References
==========

[1] https://fedorahosted.org/certmonger/
[2] http://www.freeipa.org/page/Main_Page
[3] http://pki.fedoraproject.org/wiki/PKI_Main_Page
[4] http://www.freeipa.org/page/Docker
[5] https://github.com/richm/rdo-vm-factory/blob/use-centos/rdo-ipa-nova/novahooks.py
