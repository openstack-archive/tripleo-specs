..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================================
TripleO - Pattern to safely spawn a container from a container
==============================================================

This spec describes a pattern which can be used as an alternative to
what TripleO does today to allow certain containers (Neutron, etc.) to
spawn side processes which require special privs like network
namespaces. Specifically it avoids exposing the docker socket or
using Podman nsenter hacks that have recently entered the codebase in Stein.

Problem Description
===================

In Queens TripleO implemented a containerized architecture with the goal of
containerizing all OpenStack services. This architecture was a success but
a few applications had regressions when compared with their baremetal deployed
equivalent. One of these applications was Neutron, which requires the ability
to spawn long lived "side" processes that are launched directly from the
Neutron agents themselves. In the original Queens architecture Neutron
launched these side processes inside of the agent container itself which
caused a service disruption if the neutron agents themselves were restarted.
This was previously not the case on baremetal as these processes would continue
running across an agent restart/upgrade.

The work around in Rocky was to add "wrapper" scripts for Neutron agents and
to expose the docker socket to each agent container. These wrappers scripts
were bind mounted into the containers so that they overwrote the normal location
of the side process. Using this crude mechanism binaries like 'dnsmasq' and
'haproxy' would instead launch a shell script instead of the normal binary and
these custom shell scripts relied on the an exposed docker socket from the
host to be able to launch a side container with the same arguments supplied
to the script.

This mechanism functionally solved the issues with our containerization but
exposed some security problems in that we were now exposing the ability to
launch any container to these Neutron agent containers (privileged containers
with access to a docker socket).

In Stein things changed with our desire to support Podman. Unlike Docker
Podman does not include a daemon on the host. All Podman commands are executed
via a CLI which runs the command on the host directly. We landed
patches which required Podman commands to use nsenter to enter the hosts
namespace and run the commands there directly. Again this mechanism requires
extra privileges to be granted to the Neutron agent containers in order for
them to be able to launch these commands. Furthermore the mechanism is
a bit cryptic to support and debug in the field.

Proposed Change
===============

Overview
--------

Use systemd on the host to launch the side process containers directly with
support for network namespaces that Neutron agents require. The benefit of
this approach is that we no longer have to give the Neutron containers privs
to launch containers which they shouldn't require.

The pattern could work like this:

#. A systemd.path file monitors a know location on the host for changes.
   Example (neutron-dhcp-dnsmasq.path):

.. code-block:: yaml

  [Path]
  PathModified=/var/lib/neutron/neutron-dnsmasq-processes-timestamp
  PathChanged=/var/lib/neutron/neutron-dnsmasq-processes-timestamp

  [Install]
  WantedBy=multi-user.target

#. When systemd.path notices a change it fires the service for this
   path file:
   Example (neutron-dhcp-dnsmasq.service):

.. code-block:: yaml

  [Unit]
  Description=neutron dhcp dnsmasq sync service

  [Service]
  Type=oneshot
  ExecStart=/usr/local/bin/neutron-dhcp-dnsmasq-process-sync
  User=root

#. We use the same "wrapper scripts" used today to write two files. The
   first file is a dump of CLI arguments used to launch the process
   on the host. This file can optionally include extra data like
   network namespaces which are required for some neutron side processes.
   The second file is a timestamp which is monitored by systemd.path
   on the host for changes and is used as a signal that it needs to
   process the first file with arguments.

# When a change is detected the systemd.service above executes a script on the
  host to cleanly launch containerized side processes. When the script finishes
  launching processes it truncates the file to start with a clean slate.

# Both the wrapper scripts and the host scripts use flock to eliminate race
  conditions which could cause issues in relaunching or missed containers.

Alternatives
------------

With Podman an API like varlink would be an option however it would likely
still required exposure to a socket on the host which would involve
extra privileges like what we have today. This would avoid the nsenter hacks
however.

An architecture like Kubernetes would give us an API which could be used
to launch containers directly via the COE.

Additionally an external process manager in Neutron that is "containers aware"
could be written to improve either of the above options.  The current python
in Neutron was writtin primarily for launching processes on baremetal with
assumptions that some of the processes it launches are meant to live across
a contain restart. Implementing a class that can launch side processes via a
clean interface rather than overwriting binaries would be desirable.
Classes which supported launching containers via Kubernetes and or Systemd
via the host directly could be supported.

Security Impact
---------------

This mechanism should allow us to remove some of the container privileges for
neutron agents which in the past were used to execute containers. It is
a more restrictive crude interface that allows the containers only to launch
a specific type of process rather than any container it chooses.

Upgrade Impact
--------------

The side process containers should be the same regardless of how they are
launched so the upgrade should be minimal.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dan-prince

Other contributors:
  emilienm

Work Items
----------

# Ansible playbook to create systemd files, wrappers

# TripleO Heat template updates to use the new playbooks

# Remove/deprecate the old docker.socket and nsenter code from puppet-tripleo
