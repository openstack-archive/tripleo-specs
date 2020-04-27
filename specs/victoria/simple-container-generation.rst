..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


===========================
Simple Container Generation
===========================

Simple container generation is an initiative to reduce complexity in the
TripleO container build, deployment, and distribution process by reducing the
size and scope of the TripleO container build tools.

The primary objective of this initiative is to replace Kolla, and our
associated Kolla customization tools, as the selected container generation
tool-kit. The TripleO community has long desired an easier solution for
deployers and integrators alike and this initiative is making that desire a
reality.

The Simple container generation initiative is wanting to pivot from a
tool-chain mired between a foundational component of Kolla-Ansible and a
general purpose container build system, to a vertically integrated solution
that is only constructing what TripleO needs, in a minimally invasive, and
simple to understand way.

[#f3]_


Problem Description
===================

TripleO currently leverages Kolla to produce container images. These images are
built for Kolla-Ansible using an opinionated build process which has general
purpose features. While our current images work, they're large and not well
suited for the TripleO use-case, especially in distributed data-centers. The
issue of container complexity and size impacts three major groups, deployers,
third party integrators, and maintainers. As the project is aiming to simplify
interactions across the stack, the container life cycle and build process has
been identified as something that needs to evolve. The TripleO project needs
something vertically integrated which produces smaller images, that are easier
to maintain, with far fewer gyrations required to tailor images to our needs.


Proposed Change
===============

Overview
--------

Implement a container file generation role, and a set of statically defined
override variable files which are used to generate our required
container files. [#f2]_

Layering
^^^^^^^^

.. code-block:: text

    tripleo-base+---+
                    |
                    |
                    +---+-openstack-${SERVICE}-1-common-+-->openstack-${SERVICE}-1-a
                        |                               |
                        |                               +-->openstack-${SERVICE}-1-b
                        |                               |
                        |                               +-->openstack-${SERVICE}-1-c
                        +-->openstack-${SERVICE}-2
                        |
                        +-->ancillary-${SERVICE}-1
                        |
                        +-->ancillary-${SERVICE}-2


User Experience
^^^^^^^^^^^^^^^

Building the standard set of images will be done through a simple command line
interface using the TripleO python client.

.. code-block:: shell

    $ openstack tripleo container image build [opts] <args>


This simple sub-command will provide users the ability to construct images as
needed, generate container files, and debug runtime issues.


CLI Options
^^^^^^^^^^^

The python TripleO client options for the new container image build entry point.

===========   ===============================   =================================================================
Option        Default                           Description
===========   ===============================   =================================================================
config-file   $PATH/overcloud_containers.yaml   Configuration file setting the list of containers to build.
exclude       []                                Container type exclude. Can be specified multiple times.
work-dir      /tmp/container-builds             Container builds directory, storing the container files and
                                                logs for each image and its dependencies.
skip-push     False                             Skip pushing images to the registry
skip-build    False                             Only generates container files without producing a local build.
base          centos                            Base image name.
type          binary                            Image type.
tag           latest                            Image tag.
registry      localhost                         Container registry URL.
namespace     tripleomaster                     Container registry namespace.
volume        []                                Container bind mount used when building the image. Should be
                                                specified multiple times if multiple volumes.
===========   ===============================   =================================================================


Container Image Build Tools
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Container images will be built using Buildah_, The required Buildah
functionality will leverage `BuildahBuilder` via `python-tripleoclient`
integration and be exposed though CLI options.

.. _Buildah: https://buildah.io


Image layout
^^^^^^^^^^^^

Each image will have its own YAML file which has access to the following
parameters. Each YAML file will have one required parameter (tcib_from for the
source image to build from) and optional parameters.

=================   =============================  ====================  ========  ===================================================
Option              Default                        Type                  Required  Description
=================   =============================  ====================  ========  ===================================================
tcib_path           `{{ lookup('env', 'HOME') }}`  String                          Path to generated the container file(s) for a given
                                                                                   image.
tcib_args                                          Dict[str, str]                  Single level `key:value` pairs. Implements arg_.
tcib_from           `centos:8`                     Str                   True      Container image to deploy from. Implements from_.
tcib_labels                                        Dict[str, str]                  Single level `key:value` pairs. Implements label_.
tcib_envs                                          Dict[str, str]                  Single level `key:value` pairs. Implements env_.
tcib_onbuilds                                      List[str]                       <item>=String. Implements onbuild_.
tcib_volumes                                       List[str]                       <item>=String. Implements volume_.
tcib_workdir                                       Str                             Implements workdir_.
tcib_adds                                          List[str]                       <item>=String. Implements add_.
tcib_copies                                        List[str]                       <item>=String. Implements copy_.
tcib_exposes                                       List[str]                       <item>=String. Implements expose_.
tcib_user                                          Str                             Implements user_.
tcib_shell                                         Str                             Implements shell_.
tcib_runs                                          List[str]                       <item>=String. Implements run_.
tcib_healthcheck                                   Str                             Implements healthcheck_.
tcib_stopsignal                                    Str                             Implements stopsignal_.
tcib_entrypoint                                    Str                             Implements entrypoint_.
tcib_cmd                                           Str                             Implements cmd_.
tcib_actions                                       List[Dict[str, str]]            Each item is a Single level Dictionary `key:value`
                                                                                   pairs. Allows for arbitrary verbs which maintains
                                                                                   ordering.
tcib_gather_files                                  List[str]                       Each item is a String. Collects files from the
                                                                                   host and stores them in the build directory.
=================   =============================  ====================  ========  ===================================================

.. _arg: https://docs.docker.com/engine/reference/builder/#arg
.. _from: https://docs.docker.com/engine/reference/builder/#from
.. _label: https://docs.docker.com/engine/reference/builder/#label
.. _env: https://docs.docker.com/engine/reference/builder/#env
.. _onbuild: https://docs.docker.com/engine/reference/builder/#onbuild
.. _volume: https://docs.docker.com/engine/reference/builder/#volume
.. _workdir: https://docs.docker.com/engine/reference/builder/#workdir
.. _add: https://docs.docker.com/engine/reference/builder/#add
.. _copy: https://docs.docker.com/engine/reference/builder/#copy
.. _expose: https://docs.docker.com/engine/reference/builder/#expose
.. _user: https://docs.docker.com/engine/reference/builder/#user
.. _shell: https://docs.docker.com/engine/reference/builder/#shell
.. _run: https://docs.docker.com/engine/reference/builder/#run
.. _healthcheck: https://docs.docker.com/engine/reference/builder/#healthcheck
.. _stopsignal: https://docs.docker.com/engine/reference/builder/#stopsignal
.. _entrypoint: https://docs.docker.com/engine/reference/builder/#entrypoint
.. _cmd: https://docs.docker.com/engine/reference/builder/#cmd


  Application packages are sorted within each container configuration file.
  This provides a programmatic interface to derive package sets, allows
  overrides, and is easily visualized. While the package option is not
  processes by the `tripleo_container_image_build` role, it will serve as a
  standard within our templates.

  ================  ====================================================
  Option            Description
  ================  ====================================================
  tcib_packages     Dictionary of packages to install.

                    .. code-block:: yaml

                        common:
                          - openstack-${SERVICE}-common
                        distro-1:
                          common:
                            - openstack-${SERVICE}-proprietary
                          x86_64:
                            - $dep-x86_64
                          power:
                            - $dep-power
                        distro-2:
                          common:
                            - openstack-${SERVICE}
                            - $dep
  ================  ====================================================

  This option is then captured and processed by a simple `RUN` action.

  .. code-block:: yaml

      tcib_actions:
        - run: "dnf install -y {{ tcib_packages['common'] }} {{ tcib_packages[ansible_distribution][ansible_architecture] }}"


Example Container Variable File
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    tcib_from: ubi8
    tcib_path: "{{ lookup('env', 'HOME') }}/example-image"
    tcib_labels:
      maintainer: MaintainerX
    tcib_entrypoint: dumb-init --single-child --
    tcib_stopsignal: SIGTERM
    tcib_envs:
      LANG: en_US.UTF-8
    tcib_runs:
      - mkdir -p /etc/ssh && touch /etc/ssh/ssh_known_host
    tcib_copies:
      - /etc/hosts /opt/hosts
    tcib_gather_files:
      - /etc
    tcib_packages:
      common:
        - curl
      centos:
        x86_64:
          - wget
    tcib_actions:
      - run: "dnf install -y {{ tcib_packages['common'] }} {{ tcib_packages[ansible_distribution][ansible_architecture] }}"
      - copy: /etc/resolv.conf /resolv.conf
      - run: ["/bin/bash", "-c", "echo hello world"]


Container File Structure
^^^^^^^^^^^^^^^^^^^^^^^^

The generated container file(s) will follow a simple directory structure
which provide an easy way to view, and understand, build relationships and
dependencies throughout the stack.

.. code-block:: shell

    tripleo-base/${CONTAINERFILE}
    tripleo-base/ancillary-${SERVICE}-1/${CONTAINERFILE}
    tripleo-base/ancillary-${SERVICE}-2/${CONTAINERFILE}
    tripleo-base/openstack-${SERVICE}-1-common/${CONTAINERFILE}
    tripleo-base/openstack-${SERVICE}-1-common/openstack-${SERVICE}-1-a/${CONTAINERFILE}
    tripleo-base/openstack-${SERVICE}-1-common/openstack-${SERVICE}-1-b/${CONTAINERFILE}
    tripleo-base/openstack-${SERVICE}-1-common/openstack-${SERVICE}-1-c/${CONTAINERFILE}
    tripleo-base/openstack-${SERVICE}-2/${CONTAINERFILE}


Alternatives
------------

* Use Ansible Bender

Ansible Bender was evaluated as a tool which could help to build the container
images. However it has not been productized downstream; which would make it
difficult to consume. It doesn't generate Dockerfiles and there is a strong
dependency on Bender tool; the container image build process would therefore be
more difficult to do in a standalone environment where Bender isn't available.
[#f1]_

* Leave the container image build process untouched.

We could leave the container image generate process untouched. This keeps us a
consumer of Kolla and requires we maintain our complex ancillary tooling to
ensure Kolla containers work for TripleO.


Security Impact
---------------

While security is not a primary virtue in the simple container generation
initiative, security will be improved by moving to simplified containers. If
the simple container generation initiative is ratified, all containers used
within TripleO will be vertically integrated into the stack, making it possible
to easily audit the build tools and all applications, services, and files
installed into our containerized runtimes. With simplification we'll improve
the ease of understanding and transparency which makes our project more
sustainable, thereby more secure. The proposed solution must provide layers
where we know what command has been run exactly; so we can quickly figure out
how an image was built.


Upgrade Impact
--------------

There is no upgrade impact because the new container images will provide
feature parity with the previous ones; they will have the same or similar
injected scripts that are used when the containers start.


Other End User Impact
---------------------

None


Performance Impact
------------------

We should expect better performance out of our containers, as they will be
smaller. While the runtime will act the same, the software delivery will be
faster as the size of each container will smaller, with better constructed
layers. Smaller containers will decrease the mean time to ready which will have
a positive performance impact and generally improve the user experience.


Other Deployer Impact
---------------------

The simplified container generation initiative will massively help third party
integrators. With simplified container build tools we will be able to easily
articulate requirements to folks looking to build on-top of TripleO. Our
tool-chain will be capable of bootstrapping applications where required, and
simple enough to integrate with a wide variety of custom applications
constructed in bespoke formats.


Developer Impact
----------------

In the first phase, there won't be any developer impact because the produced
images will be providing the same base layers as before. For example, they will
contain all the Kolla scripts that are required to merge configuration files or
initialize the container at startup.

These scripts will be injected in the container images for backward
compatibility:

* kolla_extend_start
* set_configs.py
* start.sh
* copy_cacerts.sh
* httpd_setup.sh

In a second phase, we will simplify these scripts to remove what isn't needed
by TripleO. The interface in the composable services will likely evolve over
time. For example kolla_config will become container_config. There is no plan
at this time to rewrite the configuration file merge logic.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * Cloudnull
  * EmilienM


Work Items
----------

First phase
^^^^^^^^^^^

* Ansible role to generate container file(s) - https://review.opendev.org/#/c/722557
* Container images layouts - https://review.opendev.org/#/c/722486
* Deprecate "openstack overcloud container image build"
* Implement "openstack tripleo container image build" which will reuse the
  `BuildahBuilder` and the same logic as the deprecated command but without Kolla.
* Build new images and publish them.
* Switch the upstream CI to use the new images.

Second phase:

* Simplifying the injected scripts to only do what we need in TripleO.
* Rename the configuration interfaces in TripleO Heat Templates.


Dependencies
============

The tooling will be in existing repositories so there is no new dependency. It
will mainly be in tripleo-ansible, tripleo-common, python-tripleoclient and
tripleo-heat-templates. Like before, Buildah will be required to build the
images.


Testing
=======

* The tripleo-build-containers-centos-8 job will be switched to be using
  the new "openstack tripleo container image build" command.

* A molecule job will exercise the container image build process using
  the new role.

* Some end-to-end job will also be investigated to build and deploy
  a container into a running deployment.


Documentation Impact
====================

Much of the documentation impact will be focused on cleanup of the existing
documentation which references Kolla, and the creation of documentation that
highlights the use of the vertically integrated stack.

Since the changes should be transparent for the end-users who just pull images
without rebuilding it, the manuals will still be updated with the new command
and options if anyone wants to build the images themselves.

References
==========

.. [#f1] https://review.opendev.org/#/c/722136/
.. [#f2] https://review.opendev.org/#/c/722557/
.. [#f3] https://blueprints.launchpad.net/tripleo/+spec/simplified-containers
