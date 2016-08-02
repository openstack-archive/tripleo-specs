..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================
Undercloud Upgrade
==================

https://blueprints.launchpad.net/tripleo/+spec/undercloud-upgrade

Our currently documented upgrade path for the undercloud is very problematic.
In fact, it doesn't work.  A number of different patches are attempting to
address this problem (see the `References`_ section), but they all take slightly
different approaches that are not necessarily compatible with each other.

Problem Description
===================

The undercloud upgrade must be carefully orchestrated.  A few of the problems
that can be encountered during an undercloud upgrade if things are not done
or not done in the proper order:

#. Services may fail and get stuck in a restart loop

#. Service databases may not be properly upgraded

#. Services may fail to stop and prevent the upgraded version from starting

Currently there is not agreement over who should be responsible for running
the various steps of the undercloud upgrade.  Getting everyone on the same
page regarding this is the ultimate goal of this spec.

Also of note is the MariaDB major version update flow from
`Upgrade documentation (under and overcloud)`_.  This will need to be
addressed as part of whatever upgrade solution we decide to pursue.

Proposed Change
===============

I'm going to present my proposed solution here, but will try to give a fair
overview of the other proposals in the `Alternatives`_ section.  Others
should feel free to push modifications or follow-ups if I miss anything
important, however.

Overview
--------

Services must be stopped before their respective package update is run.
This is because the RPM specs for the services include a mandatory restart to
ensure that the new code is running after the package is updated.  On a major
version upgrade, this can and does result in broken services because the config
files are not always forward compatible, so until Puppet is run again to
configure them appropriately the service cannot start.  The broken services
can cause other problems as well, such as the yum update taking an excessively
long time because it times out waiting for the service to restart.  It's worth
noting that this problem does not exist on an HA overcloud because Pacemaker
stubs out the service restarts in the systemd services so the package update
restart becomes a noop.

Because the undercloud is not required to have extremely high uptime, I am in
favor of just stopping all of the services, updating all the packages, then
re-running the undercloud install to apply the new configs and start the
services again.  This ensures that the services are not restarted by the
package update - which only happens if the service was running at the time of
the update - and that there is no chance of an old version of a service being
left running and interfering with the new version, as can happen when moving
a service from a standalone API process to httpd.

instack-undercloud will be responsible for implementing the process described
above.  However, to avoid complications with instack-undercloud trying to
update itself, tripleoclient will be responsible for updating
instack-undercloud and its dependencies first.  This two-step approach
should allow us to sanely use an older tripleoclient to run the upgrade
because the code in the client will be minimal and should not change from
release to release.  Upgrade-related backports to stable clients should not
be needed in any foreseeable case.  Any potential version-specific logic can
live in instack-undercloud.  The one exception being that we may need to
initially backport this new process to the previous stable branch so we can
start using it without waiting an entire cycle.  Since the current upgrade
process does not work correctly there, I think this would be a valid bug fix
backport.

A potential drawback of this approach is that it will not automatically
trigger the Puppet service db-syncs because Puppet is not aware that the
version has changed if we update the packages separately.  However, I feel
this is a case we need to handle sanely anyway in case a package is updated
outside Puppet either intentionally or accidentally.  To that end, we've
already merged a patch to always run db-syncs on the undercloud since they're
idempotent anyway.  See `Stop all services before upgrading`_ for a link to
the patch.

MariaDB
-------

Regarding the MariaDB issue mentioned above, I believe that regardless of the
approach we take, we should automate the dump and restore of the database as
much as possible.  Either solution should be able to look at the version of
mariadb before yum update and the version after, and decide whether the db
needs to be dumped.  If a user updates the package manually outside the
undercloud upgrade flow then they will be responsible for the db upgrade
themselves.  I think this is the best we can do, short of writing some sort
of heuristic that can figure out whether the existing db files are for an
older version of MariaDB and doing the dump/restore based on that.

Updates vs. Upgrades
--------------------

I am also proposing that we not differentiate between minor updates and major
upgrades on the undercloud.  Because we don't need to be as concerned with
uptime there, any additional time required to treat all upgrades as a
potential major version upgrade should be negligible, and it avoids us
having to maintain and test multiple paths.

Additionally, the difference between a major and minor upgrade becomes very
fuzzy for anyone upgrading between versions of master.  There may be db
or rpc changes that require the major upgrade flow anyway.  Also, the whole
argument assumes we can even come up with a sane, yet less-invasive update
strategy for the undercloud anyway, and I think our time is better spent
elsewhere.

Alternatives
------------

As shown in `Don't update whole system on undercloud upgrade`_, another
option is to limit the manual yum update to just instack-undercloud and make
Puppet responsible for updating everything else.  This would allow Puppet
to handle all of the upgrade logic internally.  As of this writing, there is
at least one significant problem with the patch as proposed because it does
not update the Puppet modules installed on the undercloud, which leaves us
in a chicken and egg situation with a newer instack-undercloud calling older
Puppet modules to run the update.  I believe this could be solved by also
updating the Puppet modules along with instack-undercloud.

Drawbacks of this approach would be that each service needs to be orchestrated
correctly in Puppet (this could also be a feature, from a Puppet CI
perspective), and it does not automatically handle things like services moving
from standalone to httpd.  This could be mitigated by the undercloud upgrade
CI job catching most such problems before they merge.

I still personally feel this is more complicated than the proposal above, but
I believe it could work, and as noted could have benefits for CI'ing upgrades
in Puppet modules.

There is one other concern with this that is less a functional issue, which is
that it significantly alters our previous upgrade methods, and might be
problematic to backport as older versions of instack-undercloud were assuming
an external package update.  It's probably not an insurmountable obstacle, but
I do feel it's worth noting.  Either approach is going to require some amount
of backporting, but this may require backporting in non-tripleo Puppet modules
which may be more difficult to do.

Security Impact
---------------

No significant security impact one way or another.

Other End User Impact
---------------------

This will likely have an impact on how a user runs undercloud upgrades,
especially compared to our existing documented upgrade method.
Ideally all of the implementation will happen behind the ``openstack undercloud
upgrade`` command regardless of which approach is taken, but even that is a
change from before.

Performance Impact
------------------

The method I am suggesting can do an undercloud upgrade in 20-25
minutes end-to-end in a scripted CI job.

The performance impact of the Puppet approach is unknown to me.

The performance of the existing method where service packages are updated with
the service still running is terrible - upwards of two hours for a full
upgrade in some cases, assuming the upgrade completes at all.  This is largely
due to the aforementioned problem with services restarting before their config
files have been updated.

Other Deployer Impact
---------------------

Same as the end user impact.  In this case I believe they're the same person.

Developer Impact
----------------

Discussed somewhat in the proposals, but I believe my approach is a little
simpler from the developer perspective.  They don't have to worry about the
orchestration of the upgrade, they only have to provide a valid configuration
for a given version of OpenStack.  The one drawback is that if we add any new
services on the undercloud, their db-sync must be wired into the "always run
db-syncs" list.


Implementation
==============

Assignee(s)
-----------

Primary assignees:

* bnemec
* EmilienM

Other contributors (I'm essentially listing everyone who has been involved in
upgrade work so far):

* lbezdick
* bandini
* marios
* jistr

Work Items
----------

* Implement an undercloud upgrade CI job to test upgrades.
* Implement the selected approach in the undercloud upgrade command.


Dependencies
============

None

Testing
=======

A CI job is already underway.  See `Undercloud Upgrade CI Job`_.  This should
provide reasonable coverage on a per-patch basis.  We may also want to test
undercloud upgrades in periodic jobs to ensure that it is possible to deploy
an overcloud with an upgraded undercloud.  This probably takes too long to be
done in the regular CI jobs, however.

There has also been discussion of running Tempest API tests on the upgraded
undercloud, but I'm unsure of the status of that work.  It would be good to
have in the standalone undercloud upgrade job though.


Documentation Impact
====================

The docs will need to be updated to reflect the new upgrade method.  Hopefully
this will be as simple as "Run openstack undercloud upgrade", but that remains
to be seen.


References
==========

Stop all services before upgrading
----------------------------------
Code: https://review.openstack.org/331804

Docs: https://review.openstack.org/315683

Always db-sync: https://review.openstack.org/#/c/346138/

Don't update whole system on undercloud upgrade
-----------------------------------------------
https://review.openstack.org/327176

Upgrade documentation (under and overcloud)
-------------------------------------------
https://review.openstack.org/308985

Undercloud Upgrade CI Job
-------------------------
https://review.openstack.org/346995
