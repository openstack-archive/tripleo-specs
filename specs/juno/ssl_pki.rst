..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======
SSL PKI
=======

https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-ssl-pki

Each of our clouds require multiple ssl certificates to operate. We need to
support generating these certificates in devtest in a manner which will
closely resemble the needs of an actual deployment. We also need to support
interfacing with the PKI (Public Key Infrastructure) of existing organizations.
This spec outlines the ways we will address these needs.

Problem Description
===================

We have a handful of services which require SSL certificates:

 * Keystone
 * Public APIs
 * Galera replication
 * RabbitMQ replication

Developers need to have these certificates generated automatically for them,
while organizations will likely want to make use of their existing PKI. We
have not made clear at what level we will manage these certificates and/or
their CA(s) and at what level the user will be responsible for them. This is
further complicated by the Public API's likely having a different CA than the
internal-only facing services.

Proposed Change
===============

Each of these services will accept their SSL certificate, key, and CA via
environment JSON (heat templates for over/undercloud, config.json for seed).

At the most granular level, a user can specify these values by editing the
over/undercloud-env.json or config.json files. If a certificate and key is
specified for a service then we will not attempt to automatically generate one
for that service. If only a certificate or key is specified it is considered
an error.

If no certificate and key is specified for a service, we will attempt to
generate a certificate and key, and sign the certificate with a self-signed
CA we generate. Both the undercloud and seed will share a self-signed CA in
this scenario, and each overcloud will have a separate self-signed CA. We will
also add this self-signed CA to the chain of trust for hosts which use services
of the cloud being created.

The use of a custom CA for signing the automatically generated certificates
will be solved in a future iteration.

Alternatives
------------

None presented thus far.

Security Impact
---------------

This change has high security impact as it affects our PKI. We currently do not
have any SSL support, and implementing this should therefore improve our
security. We should ensure all key files we create in this change have file
permissions of 0600 and that the directories they reside in have permissions
of 0700.

There are many security implications for SSL key generation (including entropy
availability) and we defer to the OpenStack Security Guide[1] for this.

Other End User Impact
---------------------

Users can interact with this feature by editing the under/overcloud-env.json
files and the seed config.json file. Additionally, the current properties which
are used for specifying the keystone CA and certificate will be changed to
support a more general naming scheme.

Performance Impact
------------------

We will be performing key generation which can require a reasonable amount of
resources, including entropy sources.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

More SSL keys will be generated for developers. Debugging via monitoring
network traffic can also be more difficult once SSL is adopted. Production
environments will also require SSL unwrapping to debug network traffic, so this
will allow us to closer emulate production (developers can now spot missing SSL
wrapping).

Implementation
==============

The code behind generate-keystone-pki in os-cloud-config will be generalized
to support creation of a CA and certificates separately, and support creation
of multiple certificates using a single CA. A new script will be created
named 'generate-ssl-cert' which accepts a heat environment JSON file and a
service name. This will add ssl.certificate and ssl.certificate_key properties
under the servicename property (an example is below). If no ssl.ca_certificate
and ssl.ca_certificate_key properties are defined then this script will perform
generation of the self-signed certificate.

Example heat environment output::

  {
    "ssl": {
      "ca_certificate": "<PEM Data>",
      "ca_key": "<PEM Data>"
    },
    "horizon" {
      "ssl": {
        "ca_certificate": "<PEM Data>",
        "ca_certificate_key": "<PEM Data>"
      },
    ...
    },
  ...
  }

Assignee(s)
-----------

Primary assignee:
  greghaynes

Work Items
----------

 * Generalize CA/certificate creation in os-cloud-config.
 * Add detection logic for certificate key pairs in -env.json files to devtest
 * Make devtest scripts call CA/cert creation scripts if no cert is found
   for a service

Dependencies
============

The services listed above are not all set up to use SSL certificates yet. This
is required before we can add detection logic for user specified certificates
for all services.

Testing
=======

Tests for new functionality will be made to os-cloud-config. The default
behavior for devtest is designed to closely mimic a production setup, allowing
us to best make use of our CI.

Documentation Impact
====================

We will need to document the new interfaces described in 'Other End User
Impact'.

References
==========

1. Openstack Security Guide: http://docs.openstack.org/security-guide/content/
