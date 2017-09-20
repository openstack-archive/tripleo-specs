..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Best practices for logging of containerized services
====================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/containerized-services-logs

Containerized services shall persist its logs. There are many ways to address
that. The scope of this blueprint is to suggest best practices and intermediate
implementation steps for Pike release as well.

Problem Description
===================

Pike will be released with a notion of hybrid deployments, which is some
services may be running in containers and managed by docker daemon, and
some may be managed by systemd or Pacemaker and placed on hosts directly.

The notion of composable deployments as well assumes end users and
developers may want to deploy some services non-containerized and tripleo
heat templates shall not prevent them from doing so.

Despite the service placement type, end users and developers shall get all
logs persisted, consistent and available for future analysis.

Proposed Change
===============

Overview
--------

.. note:: As the spec transitions from Pike, some of the sections below are
   split into the Pike and Queens parts.

The scope of this document for Pike is limited to recommendations for
developers of containerized services, bearing in mind use cases for hybrid
environments. It addresses only intermediate implementation steps for Pike and
smooth UX with upgrades from Ocata to Pike, and with future upgrades from Pike
as well.

A `12factor <https://12factor.net/logs>`_ is the general guideline for logging
in containerized apps. Based on it, we rephrase our main design assumption as:
"each running process writes its only event stream to be persisted outside
of its container". And we put an additional design constraint: "each container
has its only running foreground process, nothing else requires persistent
logs that may outlast the container execution time". This assumes all streams
but the main event stream are ephemeral and live no longer than the container
instance does.

.. note:: HA statefull services may require another approach, see the
  alternatives section for more details.

The scope for future releases, starting from Queens, shall include best
practices for collecting (shipping), storing (persisting), processing (parsing)
and accessing (filtering) logs of hybrid TripleO deployments with advanced
techniques like EFK (Elasticsearch, Fluentd, Kibana) or the like. Hereafter
those are referred as "future steps".

Note, this is limited to OpenStack and Linux HA stack (Pacemaker and Corosync).
We can do nothing to the rest of the supporting and legacy apps like
webservers, load balancing revers proxies, database and message queue clusters.
Even if we could, this stays out of OpenStack specs scope.

Here is a list of suggested best practices for TripleO developers for Pike:

* Host services shall keep writing logs as is, having UIDs, logging configs,
  rotation rules and target directories unchanged.

  .. note:: Host services changing its control plane to systemd or pacemaker
    in Ocata to Pike upgrade process, may have logging configs, rules and
    destinations changed as well, but this is out of the scope of this spec.

* Containerized services that normally log to files under the `/var/log` dir,
  shall keep logging as is inside of containers. The logs shall be persisted
  with hostpath mounted volumes placed under the `/var/log/containers` path.
  This is required because of the hybrid use cases. For example, containerized
  nova services access `/var/log/nova` with different UIDs than the host
  services would have. Given that, nova containers should have log volumes
  mounted as ``-v /var/log/nova:/var/log/containers/nova`` in order to not
  bring conflicts. Persisted log files then can be pulled by a node agent like
  fluentd or rsyslog and forwarded to a central logging service.

* Containerized services that can only log to syslog facilities: bind mount
  /dev/log into all tripleo service containers as well so that the host
  collects the logs via journald. This should be a standard component of our
  container "API": we guarantee (a) a log directory and (b) a syslog socket
  for *every* containerized service. Collected journald logs then can be pulled
  by a node agent like fluentd or rsyslog and forwarded to a central logging
  service.

* Containerized services that leverage Kolla bootstrap, extended start and/or
  config facilities, shall be templated with Heat deployment steps as the
  following:

  * Host prep tasks to ensure target directories pre-created for hosts.

  * Kolla config's permissions to enforce ownership for log dirs (hostpath
    mounted volumes).

  * Init containers steps to chown log directories early otherwise. Kolla
    bootstrap and DB sync containers are normally invoked before the
    `kolla_config` permissions to be set. Therefore come init containers.

* Containerized services that do not use Kolla and run as root in containers
  shall be running from a separate user namespace remapped to a non root host
  user, for security reasons. No such services are currently deployed by
  TripleO, though.

  .. note:: Docker daemon would have to be running under that remapped non root
    user as well. See docker documentation for the ``--userns-remap`` option.

* Containerized services that run under pacemaker (or pacemaker remote)
  control plane and do not fall into any of the given cases: bind mount
  /dev/log as well. At this stage the way services log is in line with the best
  practice w.r.t "dedicated log directory to avoid conflicts". Pacemaker
  bundles isolate the containerized resources' logs on the host into
  `/var/log/pacemaker/bundles/{resource}`.

Future steps TBD.

Alternatives
------------

Those below come for future steps only.

Alternatively to hostpath mounted volumes, create a directory structure such
that each container has a namespace for its logs somewhere under `/var/log`.
So, a container named 12345 would have *all its logs* in the
`/var/log/container-12345` directory structure (requires clarification).
This also alters the assumption that in general there is only one main log
per a container, which is the case for highly available containerized
statefull services bundled with pacemaker remote, with multiple logs to
capture, like `/var/log/pacemaker.log`, logs for cluster bootstrapping
events, control plane agents, helper tools like rsyncd, and the statefull
service itself.

When we have control over the logging API (e.g. via oslo.log), we can forsake
hostpath mounted volumes and configure containerized services to output to
syslog (via bind mount `/dev/log`) so that the host collects the logs via
journald). Or configure services to log only to stdout, so that docker daemon
collects logs and ships them to the journald.

.. note:: The "winning" trend is switching all (including openstack
   services) to syslog and log nothing to the /var/log/, e.g. just bind-mount
   ``-v /dev/null:/var/log`` for containers.

Or use a specialized log driver like the oslo.log fluentd logging driver
(instead of the default journald or json-file) to output to a fluentd log agent
running on the host or containerized as well, which would then aggregate logs
from all containers, annotate with node metadata, and use the fluentd
`secure_forward` protocol to send the logs to a remote fluentd agent like
common logging.

These are not doable for Pike as requiring too many changes impacting upgrade
UX as well. Although, this is the only recommended best practice and end goal for
future releases and future steps coming after Pike.

Security Impact
---------------

As the spec transitions from Pike, the section is split into the Pike and
Queens parts.

UID collisions may happen for users in containers to occasionally match another
user IDs on the host. And to allow those to access logs of foreign services.
This should be mitigated with SELinux policies.

Future steps impact TBD.

Other End User Impact
---------------------

As the spec transitions from Pike, the section is split into the Pike and
Queens parts.

Containerized and host services will be logging under different paths. The former
to the `/var/log/containers/foo` and `/var/log/pacemaker/bundles/*`, the latter
to the `/var/log/foo`. This impacts logs collecting tools like
`sosreport <https://github.com/sosreport/sos>`_ et al.

Future steps impact TBD.

Performance Impact
------------------

As the spec transitions from Pike, the section is split into the Pike and
Queens parts.

Hostpath mounted volumes bring no performance overhead for containerized
services' logs. Host services are not affected by the proposed change.

Future steps impact is that handling of the byte stream of stdout can
have a significant impact on performance.

Other Deployer Impact
---------------------

As the spec transitions from Pike, the section is split into the Pike and
Queens parts.

When upgrading from Ocata to Pike, containerized services will change its
logging destination directory as described in the end user impact section.
This also impacts logs collecting tools like sosreport et al.

Logrotate scripts must be adjusted for the `/var/log/containers` and
`/var/log/pacemaker/bundles/*` as well.

Future steps impact TBD.

Developer Impact
----------------

As the spec transitions from Pike, the section is split into the Pike and
Queens parts.

Developers will have to keep in mind the recommended intermediate best
practices, when designing heat templates for TripleO hybrid deployments.

Developers will have to understand Kolla and Docker runtime internals, although
that's already the case once we have containerized services onboard.

Future steps impact (to be finished):

* The notion of Tracebacks in the events is difficult to handle as a byte
  stream, because it becomes the responsibility of the apps to ensure output
  of new-line separated text is not interleaved. That notion of Tracebacks
  needs to be implemented apps side.

* Oslo.log is really emitting a stream of event points, or trace points, with
  rich metadata to describe those events. Capturing that metadata via a byte
  stream later needs to be implemented.

* Event streams of child processes, forked even temporarily, should or may need
  to be captured by the parent events stream as well.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bogdando

Other contributors:
  michele
  flaper87
  larsks
  dciabrin

Work Items
----------

As the spec transitions from Pike, the work items are split into the Pike and
Queens parts:

* Implement an intermediate logging solution for tripleo-heat-templates for
  containerized services that log under `/var/log` (flaper87, bogdando). Done
  for Pike.
* Come up with an intermediate logging solution for containerized services that
  log to syslog only (larsks). Done for Pike.
* Come up with a solution for HA containerized services managed by Pacemaker
  (michele). Done for Pike.
* Make sure that sosreport collects `/var/log/containers/*` and
  `/var/log/pacemaker/bundles/*` (no assignee). Pending for Pike.
* Adjust logrotate scripts for the `/var/log/containers` and
  `/var/log/pacemaker/bundles/*` paths (no assignee). Pending for Pike.
* Verify if the namespaced `/var/log/` for containers works and fits the case
  (no assignee).
* Address the current state of OpenStack infrastructure apps as they are, and
  gently move them towards these guidelines referred as "future steps" (no
  assignee).

Dependencies
============

None.

Testing
=======

Existing CI coverage fully fits the proposed change needs.

Documentation Impact
====================

The given best practices and intermediate solutions built from those do not
involve changes visible for end users but those given in the end users impact
section. The same is true for developers and dev docs.

References
==========

* `Sosreport tool <https://github.com/sosreport/sos>`_.
* `Pacemaker container bundles <http://lists.clusterlabs.org/pipermail/users/2017-April/005380.html>`_.
* `User namespaces in docker <https://success.docker.com/KBase/Introduction_to_User_Namespaces_in_Docker_Engine>`_.
* `Docker logging drivers <https://docs.docker.com/engine/admin/logging/overview/>`_.
* `Engineering blog posts <http://blog.oddbit.com/2017/06/14/openstack-containers-and-logging/>`_.
