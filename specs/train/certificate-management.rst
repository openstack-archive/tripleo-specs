..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Move certificate management in tripleo-heat-templates
=====================================================

Launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/ansible-certmonger

Problem Description
===================

There are multiple issues with the current way certificates are managed with
Puppet and Certmonger, especially in a containerized environment:

* Multiple containers are using the same certificate
* There isn't any easy way to find out which container needs to be restarted
  upon certificate renewal
* Shared certificates are bad

The main issue now is the use of "pkill", especially for httpd services. Since
Certmonger has no knowledge of what container has an httpd service running,
it uses a wide fly swatter in the hope all related services will effectively
be reloaded with the new certificate.

The usage of "pkill" by Certmonger is prevented on a SELinux enforcing host.

Proposed Change
===============

Introduction
------------

While the use of certmonger isn't in question, the way we're using it is.

The goal of this document is to describe how we could change that usage,
allowing to provide a better security, while allowing Certmonger to restart
only the needed containers in an easy fashion.

Implement certmonger in Ansible
-------------------------------

A first step will be to implement a certmonger "thing" in Ansible. There are
two ways to do that:

* Reusable role
* Native Ansible module

While the first one is faster to implement, the second would be better, since
it will allow to provide a clean way to manage the certificates.

Move certificate management to tripleo-heat-templates
-----------------------------------------------------

Once we have a way to manage Certmonger within Ansible, we will be able to move
calls directly in relevant tripleo-heat-templates files, allowing to generate
per-container certificate.

Doing so will also allow Certmonger to know exactly which container to
restart upon certificate renewal, using a simple "container_cli kill" command.

Alternatives
============

One alternative is proposed

Maintain a list
---------------

We could maintain the code as-is, and just add a list for the containers
needing a restart/reload. Certmonger would loop on that list, and do its
job upon certificate renewal.

This isn't a good solution, since the list will eventually lack updates, and
this will create new issues instead of solving the current ones.

Also, it doesn't allow to get per-container certificate, which is bad.

Proposed roadmap
================

In Stein:

* Create "tripleo-certmonger" Ansible reusable role in tripleo-common

In Train:

* Move certificate management/generation within tripleo-heat-templates.
* Evaluate the benefices of moving to a proper Ansible module for Certmonger.
* If evaluation is good and we have time, implement it and update current code.

In "U" release:

* Check if anything relies on puppet-certmonger, and if not, drop this module.

Security Impact
===============

We will provide a better security level by avoiding shared x509 keypairs.

Upgrade Impact
==============

Every container using the shared certificate will be restarted in order to
load the new, dedicated one.

We will have to ensure the nova metadata are properly updated in order to
let novajoin create the services in FreeIPA, allowing to request per-service
certificates.

Tests should also be made regarding novajoin update/upgrade in order to ensure
all is working as expected.

If the containers are already using dedicated certificates, no other impact is
expected.

End User Impact
===============

During the upgrade, a standard short downtime is to be expected, unless
the deployment is done using HA.

Performance Impact
==================

No major performance impact is expected.

Deployer Impact
===============

No major deployer impact is expected.

Developer Impact
================

People adding new services requiring a certificate will need to call the
Certmonger module/role in the new tripleo-heat-templates file.

They will also need to ensure new metadata is properly generated in order to
let novajoin create the related service in FreeIPA.

Implementation
==============

Contributors
------------

* Cédric Jeanneret
* Grzegorz Grasza
* Nathan Kinder

Work Items
----------

* Implement reusable role for Certmonger
* Move certificate management to tripleo-heat-templates
* Remove certmonger parts from Puppet
* Update/create needed documentations about the certificate management

Later:
* Implement a proper Ansible Module
* Update the role in order to wrap module calls


Dependencies
============

None - currently, no Certmonger module for Ansible exists.

Testing
=======

We have to ensure the dedicated certificate is generated with the right
content, and ensure it's served by the right container.

We can do that using openssl CLI, maybe adding a new check in the CI via
a new role in tripleo-quickstart-extras.

This is also deeply linked to novajoin, thus we have to ensure it works as
expected.

Documentation Impact
====================

We will need to document how the certificate are managed.

References
==========

* `Example of existing certificate management in Ansible <https://github.com/ansible/ansible/tree/devel/lib/ansible/modules/crypto>`_
* `Skeleton certmonger_getcert <https://github.com/nkinder/ansible/commit/c2f74d07e6b71055fad2207ed26ae82bb8beffc3>`_
* `Existing reusable roles in TripleO <https://github.com/openstack/tripleo-common/tree/master/roles>`_
