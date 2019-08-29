..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Replace Mistral with Ansible
============================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/tripleo-mistral-to-ansible

The goal of this proposal is to replace Mistral in TripleO with Ansible
playbooks.


Problem Description
===================

Mistral was originally added to take the place of an “API” and provide common
logic for tripleoclient and TripleO UI. After the TripleO UI was removed, the
only consumer of Mistral is tripleoclient. This means that Mistral now adds
unnecessary complexity.


Proposed Change
===============

Overview
--------

Remove Mistral from the TripleO undercloud and convert all Mistral workbooks,
workflows and actions to Ansible playbooks within tripleo-ansible. tripleoclient
will then be updated to execute the Ansible playbooks rather than the Mistral
workflows.

Alternatives
------------

The only other alternative candidate is to keep using Mistral and accept the
complexity and reinvest in the project.

Security Impact
---------------

* As the code will be re-writing Mistral workflows that currently deal with
  passwords, tokens and secrets we will need to be careful. However the logic
  should be largely the same.

* With the eventual removal of Mistral and Zaqar two complex systems can be
  removed which will reduce the surface area for security issues.

* The new Ansible playbooks will only use the undercloud OpenStack APIs,
  therefore they shouldn't create a new attack vector.


Upgrade Impact
--------------

* Upgrades will need to remove Mistral services and make sure the Ansible
  playbooks are in place.

* Older versions of tripleoclient will no longer work with the undercloud as
  they will expect Mistral to be present.

* Most of the data in Mistral in ephemeral, but some longer term data is stored
  in Mistral environments. This data will likely be moved to Swift.


Other End User Impact
---------------------

The output of CLI commands will change format. For example, the Mistral
workflow ID will no longer be included and other Ansible specific output may be
included. Where possible we will favour streaming Ansible output to the user,
making tripleoclient very light and transparent

Some CLI commands, such as introspection will need to fundamentally change their
output. Currently they send real time updates and progress to the client with
Zaqar. Despite moving the execution locally, we are unable to easily get
messages from a Ansible playbook while it is running. This means the user may
need to wait a long time before they get any feedback.


Performance Impact
------------------

There is no expected performance impact as the workflow should be largely the
same. However, the Ansible playbooks will be executed where the user runs the
CLI rather than by the Mistral server. This could then be slower or faster
depending on the resources available to the machine and the network connection
to the undercloud.

The undercloud itself should have more resources available since it wont be
running Mistral or Zaqar.


Other Deployer Impact
---------------------

If anyone is using the Mistral workflows directly, they will stop working. We
currently don't know of any users doing this and it was never documented.


Developer Impact
----------------

Developers will need to contribute to Ansible playbooks instead of Mistral
workflows. As the pool of developers that know Ansible is larger than those
that know Mistral this should make development easier. Ansible contributions
will likely expect unit/functional tests.


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  d0ugal

Other contributors:
  apetrich
  ekultails
  sshnaidm
  cloudnull

Work Items
----------

- Migrate each Mistral workflows to Ansible playbooks.

- Migrate or replace custom Mistral actions to Ansible native components.

- Remove Mistral and Zaqar.

- Update documentation specific to Mistral.

- Extend our auto-documentation plugin to support playbooks within
  tripleo-ansible. This will allow us to generate API documentation for all
  playbooks committed to tripleo-ansible, which will include our new `cli`
  prefixed playbooks.


Dependencies
============

None


Testing
=======

Since this change will largely be a re-working of existing code the changes
will be tested by the existing CI coverage. This should be improved and
expanded as is needed.


Documentation Impact
====================

Any references to Mistral will need to be updated to point to the new ansible
playbook.


References
==========

* https://review.opendev.org/#/q/topic:mistral-removal+OR+topic:mistral_to_ansible

* https://bugs.launchpad.net/tripleo/+bugs?field.tag=mistral-removal

* http://lists.openstack.org/pipermail/openstack-discuss/2019-October/010384.html
