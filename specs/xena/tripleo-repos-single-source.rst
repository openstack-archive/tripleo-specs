..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
TripleO Repos Single Source
===========================

This proposal lays out the plan to use tripleo-repos as a single source
to install and configure non-base OS repos for TripleO - including
setting the required DLRN hashes.

https://blueprints.launchpad.net/tripleo/+spec/tripleo-repos-single-source

Problem Description
===================

Reviewing the code base, there are multiple places where repos are
specified. For example,in the release files we are setting the
configuration that is applied by `repo setup role`_.
Some of the other repo/version configurations are included in:

* `tripleo repos`_
* `repo setup role`_
* `release config files`_
* `container tooling (base tcib file)`_
* `tripleo-ansible`_
* `rdo config`_ (example)
* `tripleo-heat-templates`_
* `tripleo-operator-ansible`_

.. _`tripleo repos`: https://opendev.org/openstack/tripleo-repos
.. _`repo setup role`: https://opendev.org/openstack/tripleo-quickstart/src/commit/d14d81204036a02562c3f4efd7acb3b38cb6ae95/roles/repo-setup/templates/repo_setup.sh.j2#L72
.. _`release config files`: https://opendev.org/openstack/tripleo-quickstart/src/commit/d14d81204036a02562c3f4efd7acb3b38cb6ae95/config/release/tripleo-ci/CentOS-8/master.yml#L93
.. _`container tooling (base tcib file)`: https://opendev.org/openstack/tripleo-common/src/commit/d3286377132ee6b0689a39e52858c07954711d13/container-images/tcib/base/base.yaml#L59
.. _`tripleo-ansible`: https://opendev.org/openstack/tripleo-ansible/src/commit/509e630baa92673e72e641635d5742da01b4dc3b/tripleo_ansible/roles/tripleo_podman/vars/redhat-8.2.yml
.. _`rdo config`: https://review.rdoproject.org/r/31439
.. _`tripleo-heat-templates`: https://opendev.org/openstack/tripleo-heat-templates/src/commit/125f45820255efe370af1024080bafc695892faa/environments/lifecycle/undercloud-upgrade-prepare.yaml
.. _`tripleo-operator-ansible`: https://opendev.org/openstack/tripleo-operator-ansible/src/commit/14a601a47be217386df83512fae3a2e5aa5444a3/roles/tripleo_container_image_build/molecule/default/converge.yml#L172


The process of setting repo versions requires getting and
transforming DLRN hashes, for example resolving 'current-tripleo'
to a particular DLRN build ID and specifying the correct proxies.
Currently a large portion of this work is done in the release files
resulting in sections of complicated and fragile Bash scripts -
duplicated across numerous release files.

This duplication, coupled with the various locations in use
for setting repo configurations, modules and supported versions
is confusing and error prone.

There should be one source of truth for which repos are installed
within a tripleo deployment and how they are installed.
Single-sourcing all these functions will avoid the current
problems of duplication, over-writing settings and version confusion.

Proposed Change
===============

Overview
--------

This proposal puts forward using tripleo-repos as the 'source of truth'
for setting repo configurations, modules and supported versions -
including setting the DLRN hashes required to specify exact repo
versions to install for upstream development/CI workflows.

Having a single source of truth for repo config, modules, etc. will make
development and testing more consistent, reliable and easier to debug.

The intent is to use the existing tripleo-repos repo for this work and
not to create a new repo. It is as yet to be determined if we will add
a v2/versioned api or how we will handle the integration with the
existing functionality there.

We aim to modularize the design and implementation of the proposed tripleo-repos
work. Two sub systems in particular have been identified that can be
implemented independently of, and ultimately to be consumed by, tripleo-repos;
the resolution of delorean build hashes from known tags (i.e. resolving
'current-tripleo' to a particular DLRN build ID) and the configuration of dnf
repos and modules will be implemented as independent python modules, with
their own unit tests, clis, ansible modules etc.

Integration Points
------------------

The new work in tripleo-repos will have to support with all
the cases currently in use and will have to integrate with:

* DLRN Repos
* release files
* container and overcloud image builds
* rdo config
* yum/dnf repos and modules
* Ansible (Ansible module)
* promotion pipeline - ensuring the correct DLRN hashes

Incorporating the DLRN hash functionality makes the tool
more complex. Unit tests will be required to guard
against frequent breakages. This is one of the reasons that we decided to split
this DLRN hash resolution into its own dedicated python module
'tripleo-get-hash' for which we can have independent unit tests.

The scope of the new tripleo-repos tool will be limited to upstream
development/CI workflows.

Alternatives
------------

Functionality to set repos, modules and versions is already available today.
It would be possible to leave the status quo or:

* Use rdo config to set one version per release - however, this would not
  address the issue of changing DLRN hashes
* Create an rpm that lays down /etc/tripleo-release where container-tools could
  be meta data in with that, similar to /etc/os-release

Security Impact
---------------

No security impact is anticipated. The work is currently in the tripleo
open-source repos and will remain there - just in a consolidated
place and format.

Upgrade Impact
--------------

Currently there will be no upgrade impact. The new CLI will support
all release versions under support and in use. At a later date,
when the old CLI is deprecated there may be some update
implications.

However,there may be work to make the emit_releases_file
https://opendev.org/openstack/tripleo-ci/src/branch/master/scripts/emit_releases_file/emit_releases_file.py
functionality compatible with the new CLI.

Other End User Impact
---------------------

Work done on the new project branch will offer a different version of CLI, v2.
End users would be able to select which version of the CLI to use - until
the old CLI is deprecated.


Performance Impact
------------------

No performance impact is expected. Possible performance improvements could
result from ensuring that proxy handling (release file, mirrors, rdoproject)
is done correctly and consistently.

Other Deployer Impact
---------------------


Developer Impact
----------------

See ```Other End User Impact``` section.

Implementation
==============

The functionality added to tripleo-repos will be writen as a Python module
with a CLI and will be able to perform the following primary functions:

* Single source the installation of all TripleO related repos
* Include the functionality current available in the repo-setup role
  including creating repos from templates and files
* Perform proxy handling such as is done in the release files
  (mirrors, using rdoproject for DLRN repos)
* Get and transform human-readable DLRN hashes - to be implemented as an
  independent module.
* Support setting yum modules such as container-tools - to be implemented
  as an independent module.
* Support enabling and disabling repos and setting their priorities

The repo-setup role shall remain but it will invoke tripleo-repos.
All options required to be passed to tripleo-repos should be in the
release file.

Work done on the new project branch will offer a different version of CLI, v2.
Unit tests will be added on this branch to test the new CLI directly.
CI would be flipped to run in the new branch when approved by TripleO teams.
All current unit tests should pass with the new code.

An Ansible module will be added to call the tripleo-repos
options from Ansible directly without requiring the end
user to invoke the Python CLI from within Ansible.

The aim is for tripleo-repos to be the single source for all repo related
configuration. In particular the goal is to serve the following 3 personas:

* Upstream/OpenStack CI jobs
* Downstream/OSP/RHEL jobs
* Customer installations

The configuration required to serve each of these use cases is slightly
different. In upstream CI jobs we need to configure the latest current-tripleo
promoted content repos. In downstream/OSP jobs we need to use rhos-release
and in customer installations we need to use subscription manager.

Because of these differing requirements we are leaning towards storing the
configuration for each in their intended locations, with the upstream config
being the 'base' and the downstream config building ontop of that (the
implication is that some form of inheritance will be used to avoid duplication).
This was discussed during the `Xena PTG session`_

.. _`Xena PTG session`: https://etherpad.opendev.org/p/ci-tripleo-repos

Assignee(s)
-----------

* sshnaidm (DF and CI)
* marios (CI and W-release PTL)
* weshay
* chandankumar
* ysandeep
* arxcruz
* rlandy
* other DF members (cloudnull)

Work Items
----------

Proposed Schedule
=================

Investigative work will be begin in the W-release cycle on a project branch
in tripleo-repos. The spec will be put forward for approval in the X-release
cycle and impactful and integration work will be visible once the spec
is approved.

Dependencies
============

This work has a dependency on the `DLRN API`_ and on yum/dnf.

.. _`DLRN API`: https://dlrn.readthedocs.io/en/latest/api.html

Testing
=======

Specific unit tests will be added with the python-based code built.
All current CI tests will run through this work and will
test it on all releases and in various aspects such as:

* container build
* overcloud image build
* TripleO deployments (standalone, multinode, scenarios, OVB)
* updates and upgrades

CLI Design
==========

Here is an abstract sketch of the intended cli design for the
new tripleo-repos.

It covers most of the needs discussed at multiple places.

Scenario 1
----------

The goal is to construct a repo with the correct hash for an integration
or a component pipeline.

For this scenario:

* Any combination of `hash, distro, commit, release, promotion, url` parameters can passed
* Use the `tripleo-get-hash`_ module to determine the DLRN build ID
* Use the calculated DLRN build ID to create and add a repo

.. _`tripleo-get-hash`: https://opendev.org/openstack/tripleo-repos/src/branch/master/tripleo-get-hash


Scenario 2
----------

The goal is to construct any type of yum/dnf repo.

For this scenario:

* Construct and add a yum/dnf repo using a combination of the following parameters
* filename - filename for saving the resulting repo (mandatory)
* reponame - name of repository (mandatory)
* baseurl - base URL of the repository (mandatory)
* down_url - URL to download repo file from (mandatory/multually exclusive to baseurl)
* priority - priority of resulting repo (optional)
* enabled - 0/1 whether the repo is enabled or not (default: 1 - enabled)
* gpgcheck - whether to check GPG keys for repo (default: 0 - don't check)
* module_hotfixes - whether to make all RPMs from the repository available (default: 0)
* sslverify - whether to use a cert to use repo metadata (default: 1)
* type - type of the repo(default: generic, others: custom and file)


Scenario 3
----------

The goal is to enable or disable specific dnf module and also install or
remove a specific package.

For this scenario:

* Specify
* module name
* which version to disable
* which version to enable
* which specific package from the module to install (optional)


Scenario 4
----------

The goal is to enable or disable some repos,
remove any associated repo files no longer needed,
and then perform a system update.

For this scenario:

* Specify
* repo names to be disabled
* repo names to be enabled
* the files to be removed
* whether to perform the system update


Documentation Impact
====================

tripleo-docs will be updated to point to the new supported
repo/modules/versions setting workflow in tripleo-repos.

References to old sources of settings such as tripleo-ansible,
release files in tripleo-quickstart and the repo-setup role
will have to be removed and replaced to point to the new
workflow.
