..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Enable logging to stdout/journald for rsyslog
=============================================

https://blueprints.launchpad.net/tripleo/+spec/logging-stdout-rsyslog

We can optimize the current logging implementation to take advantage
of metadata that our default logging driver (journald) adds to the
logs.

Problem Description
===================

Currently, we put all the logs of the containers into a directory in
the host (/var/log/containers/). While this approach works, it relies
on mounting directories from the host itself. This makes it harder for
logging forwarders, since we need to configure them to track all those
files. With every service that we add, we end up having to write
configuration for that service for those specific files.

Furthermore, we lose important metadata with this approach. We can
figure out what service wrote what log, but we lose the container name and ID,
which is very useful. These we can easily get just by using the default
docker logging mechanism.

Instead of relying on the host filesystem for our logs, we can adopt a
simpler solution that both preserves important metadata that is
discarded by the current implementation and that will support most
services without requiring per-service configuration.

Proposed Change
===============

Overview
--------

The proposal is to configure containerized services to log to
stdout/stderr as is common practice for containerized applications.
This allows the logs to get picked up by the docker logging driver,
and thus we can use "docker logs" to view the logs of a service as one
would usually expect. It will also help us decouple the
containers from the host, since we will no longer be relying on host
filesystem mounts for log collection.

In the case of services where it's difficult or not possible to log to
stdout or stderr, we will place log files in a docker volume, and this
volume will be shared with a sidecar container that will output the
logs to stdout so they are consumable by the logging drvier. This will
also apply for containers that log only to syslog (such as HAProxy).
We will stop mounting ``/dev/log`` from the host, and instead add a
sidecar container that will output the logs instead.

Additionally, since our default logging driver is journald, we will
get all the container logs accessible via ``journalctl`` and the
journald libraries. So one would be able to do ``journalctl
CONTAINER_NAME=<container name>`` to get the logs of a specific
container on the node. Furthermore, we would get extra metadata
information for each log entry [1]. We would benefit for
getting the container name (as the ``CONTAINER_NAME`` metadata item)
and the container ID (as the ``CONTAINER_ID`` and
``CONTAINER_ID_FULL`` metadata items) from each journald log entry
without requiring extra processing.  Adding extra tags to the
containers is possible [2], and would get reflected via the
``CONTAINER_TAG`` metadata entry. These tags can optionally describe the
application that emitted the logs or describe the platform that it
comes from.

This will also make it easier for us to forward logs, since there will
be a centralized service (journald) on each host from which we can
collect the logs.  When we add a new service, it will be a matter of
following the same logging pattern, and we will automatically be able
to forward those logs without requiring specific configuration to
track a new set of log files.

With this solution in place, we need to also provide tooling to
integrate with centralized logging solutions. This will then cover
integration to the Openshift Logging Stack [3] and ViaQ [4]. We are
proposing the use of rsyslog for message collection, manipulation, and
log forwarding.  This will also be done in a containerized fashion,
where rsyslog will be a "system container" that reads from the host
journal. Rsyslog will perform metadata extraction from log messages
(such as extracting the user, project, and domain from standard oslo
format logs), and will then finally forward the logs to a central
collector.

Pluggable implementation
~~~~~~~~~~~~~~~~~~~~~~~~

The implementation needs to be done in a pluggable manner. This is because
end-users have already created automation based on the assumption that logs
exist in the ``/var/log/<service>`` / ``/var/log/containers/*`` directories
that we have been providing. For this reason, logging to stdout/stderr will be
optional, and we'll keep logging to files in the host as a default for now.
This will then be optionally enabled via an environment file.

Example
~~~~~~~

nova-api container:

In the proposed solution, the standard nova logs will go to the
nova_api container's stdout/stderr. However, since we are also
interested in the apache access logs, we will then create a docker
volume where the access logs will be hosted. A sidecar container will
mount this volume, create a FIFO (named pipe) and output whatever it
gets from that file. Note that this sidecar container will need to be
started before the actual nova_api container.

For each log file generated in the main container, we will create a
sidecar container that outputs that log.  This will make it easier to
associate log messages with the originating service.

Alternatives
------------

Keep logging to files in the hosts' directory.

We can still use the current solution; however, it is not ideal as it
violates container logging best practices, relies heavily on
directories on the host (which we want to avoid) and is inconsistent
in the way we can get logging from services (some in files, some in
syslog).

Other End User Impact
---------------------

Since we're not getting rid of the previous logging solution, users won't be
impacted. They will, however, get another way of getting logs and interacting
with them in the host system, and further create automation from that if
needed.

Performance Impact
------------------

* TODO: Any performance considerations on getting everything to journald?

Implementation
==============

Primary assignees:
  jaosorior
  jbadiapa
  larsks

Work Items
----------

* Allow services to log to stdout/stderr (if possible).

* Implement pluggable logging for each service in t-h-t.

* Add Rsyslog container.

Testing
=======

TODO: Evaluate how can we log to an EFK stack in upstream CI. Do we have one
available?

References
==========

[1] https://docs.docker.com/engine/admin/logging/journald/
[2] https://docs.docker.com/engine/admin/logging/log_tags/
[3] https://docs.openshift.com/container-platform/3.5/install_config/aggregate_logging.html
[4] https://github.com/ViaQ/Main/blob/master/README-install.md
