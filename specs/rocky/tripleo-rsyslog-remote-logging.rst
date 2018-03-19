..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
        TripleO Remote Logging
==========================================

https://blueprints.launchpad.net/tripleo/+spec/remote-logging

This spec is meant to extend the tripleo-logging spec also for queens to
address key issues about log transport and storage that are separate from
the technical requirements created by logging for containerized processes.

Problem Description
===================

Having logs stuck on individual overcloud nodes isn't a workable solution
for a modern system deployed at scale. But log aggregation is complex both
to implement and to scale. TripleO should provide a robust, well documented,
and scalable solution that will serve the majority of users needs and be
easily extensible for others.


Proposed Change
===============

Overview
--------

In addition to the rsyslog logging to stdout defined for containers in the
triple-logging spec this spec outlines how logging to remote targets should
work in detail.

Essentially this comes down to a set of options for the config
of the rsyslog container. Other services will have a fixed rsyslog config
that forwards messages to the rsyslog container to pick up over journald.

1. Logging destination, local, remote direct, or remote aggregator.

Remote direct means to go direct to a storage solution, in this case
Elasticsearch or plaintext on the disk. Remote aggregator is a design where
the processing, formatting, and insertion of the logs is a task left to the
aggregator server. Using aggregators it's possible to scale log collection to
hundreds of overcloud nodes without overwhelming the storage backend with
inefficient connections.

2. Log caching for remote targets

In the case of remote targets a caching system can be setup, where logs are
stored temporarily on the local machine in a configurable disk or memory cache
until they can be uploaded to an aggregator or storage system. While some in
memory cache is mandatory users may select a disk cache depending on how
important it is that all logs be saved and stored. This allows recovery
without loss of messages during network outages or service outages.


3. Log security in transit

In some cases encryption during transit may be required. rsyslog offers
ssl based encryption that should be easily deployable.

4. Standard and extensible format

By default logs should be formatted as outlined by the Redhat common logging
initiative. By standardizing logging format where possible various tools
and analytics become more portable.

Mandatory fields for this standard formatting include.

version: the version of the logging template
level: loglevel
message: the log message
tags: user specific tagging info

Additional fields must be added in the format of

<subject>.<subfield name>

See an example by rsyslog for storage in Elasticsearch below.

@timestamp 		November 27th 2017, 08:54:40.091
@version 		2016.01.06-0
_id 		AV_9wiWQzdGOuK5_zY5J
_index 		logstash-2017.11.27.08
_score
_type 		rsyslog
browbeat.cloud_name 		openstack-12-noncontainers-beta
hostname 		lorenzo.perf.lab.eng.rdu.redhat.com
level 		info
message 		Stopping LVM2 PV scan on device 8:2...
pid 		1
rsyslog.appname 		systemd
rsyslog.facility 		daemon
rsyslog.fromhost-ip 		10.12.20.155
rsyslog.inputname 		imptcp
rsyslog.protocol-version 		1
syslog.timegenerated 		November 27th 2017, 08:54:40.092
systemd.t.BOOT_ID 		1e99848dbba047edaf04b150313f67a8
systemd.t.CAP_EFFECTIVE 		1fffffffff
systemd.t.CMDLINE 		/usr/lib/systemd/systemd --switched-root --system --deserialize 21
systemd.t.COMM 		systemd
systemd.t.EXE 		/usr/lib/systemd/systemd
systemd.t.GID 		0
systemd.t.MACHINE_ID 		0d7fed5b203f4664b0b4be90e4a8a992
systemd.t.SELINUX_CONTEXT 		system_u:system_r:init_t:s0
systemd.t.SOURCE_REALTIME_TIMESTAMP 		1511790880089672
systemd.t.SYSTEMD_CGROUP 		/
systemd.t.TRANSPORT 		journal
systemd.t.UID 		0
systemd.u.CODE_FILE 		src/core/unit.c
systemd.u.CODE_FUNCTION 		unit_status_log_starting_stopping_reloading
systemd.u.CODE_LINE 		1417
systemd.u.MESSAGE_ID 		de5b426a63be47a7b6ac3eaac82e2f6f
systemd.u.UNIT 		lvm2-pvscan@8:2.service
tags

As a visual aid here's a quick diagram of the flow of data.

<rsyslog in process container> -> <journald> -> <rsyslog container> -> <rsyslog aggregator / Elasticsearch>

In the process container logs from the application are packaged with metadata
from systemd and other components depending on how rsyslog is configured,
journald acts as a transport aggregating this input across all containers for
the rsyslog container which formats this data into storable json and handles
things like transforming fields and adding additional metadta as desired.
Finally the data is inserted into elasticsearch or further held by an
aggrebator for a few seconds before being bulk inserted into Elasticsearch.


Alternatives
------------

TripleO already has some level of FluentD integration, but performance issues
make it unusable at scale. Furthermore it's not well prepared for container
logging.

Ideally FluentD as a logging backend would be maintained, improved, and modified
to use the common logging format for easy swapping of solutions.

Security Impact
---------------

The security of remotely stored data and the log storage database is outside
of the scope of this spec. The major remaining concerns are security in
in transit and the changes required to systemd for rsyslog to send data
remotely.

A new systemd policy will have to be put into place to ensure that systemd
can successfully log to remote targets. By default the syslog rules prevent
any outside world access or port access, both of which are required for
log forwarding.

For log encryption in transit a ssl certificate will have to be generated and
distributed to all nodes in the cloud securely, probably during deployment.
Special care should be taken to ensure that any misconfigured instance of
rsyslog without a certificate where one is required do not transmit logs
by accident.


Other End User Impact
---------------------

Ideally users will read some documentation and pass an extra 5-6 variables to
TripleO to deploy with logging aggregation. It's very important that logging
be easy to setup with sane defaults and no requirement on the user to implement
their own formatting or template.

Users may also have to setup a database for log storage and an aggregator if
their deployment is large enough that they need one. Playbooks to do this
automatically will be provided, but probably don't belong in TripleO.

Special care will have to be taken to size storage and aggregation hardware
to the task, while rsyslog is very efficient storage quickly becomes a problem
when a cloud can generate 100gb of logs a day. Especially since log storage
systems leave it up to the user to put in place rotation rules.


Performance Impact
------------------

For small clouds rsyslog direct to Elasticsearch will perform just fine.
As scale increases an aggregator (also running rsyslog, except configured
to accept and format input) is required. I have yet to test a large enough
cloud that an aggregator was at all stressed. Hundreds of gigs of logs a day
are possible with a single 32gb ram VM as an Elastic instance.

For the Overcloud nodes forwarding their logs the impact is variable depending
on the users configuration. CPU requirements don't exceed single digits of a
single core even under heavy load but storage requirements can balloon if a
large on disk cache was specified and connectivity with the aggregator or
database is lost for prolonged periods.

Memory usage is no more than a few hundred mb and most of that is the default
in memory log cache. Which once again could be expanded by the user.


Other Deployer Impact
---------------------

N/A

Developer Impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  jkilpatr

Other contributors:
  jaosorior

Work Items
----------

rsyslog container - jaosorior

rsyslog templating and deployment role - jkilpatr

aggregator and storage server deployment tooling - jkilpatr


Dependencies
============

Blueprint dependencies:

https://blueprints.launchpad.net/tripleo/+spec/logging-stdout-rsyslog

Package dependencies:

rsyslog, rsyslog-elasticsearch, rsyslog-mmjsonparse

specifically version 8 of rsyslog, which is the earliest
supported by rsyslog-elasticsearch, these are packaged in
Centos and rhel 7.4 extras.

Testing
=======

Logging aggregation can be tested in CI by deploying it during any existing CI job.

For extra validation have a script to check the output into Elasticsearch.


Documentation Impact
====================

Documentation will need to be written about the various modes and tunables for
logging and how to deploy them. As well as sizing recommendations for the log
storage system and aggregators where required.


References
==========

https://review.openstack.org/#/c/490047/

https://review.openstack.org/#/c/521083/

https://blueprints.launchpad.net/tripleo/+spec/logging-stdout-rsyslog
