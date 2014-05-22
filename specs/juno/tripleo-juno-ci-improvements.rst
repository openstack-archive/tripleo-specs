..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Triple CI improvements
======================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-ci-improvements

Tripleo CI is painful at the moment, we have problems with both reliability
and consistency of running job times, this spec is intended to address a
number of the problems we have been facing.

Problem Description
===================

Developers should be able to depend on CI to produce reliable test results with
a minimum number of false negatives reported in a timely fashion, this
currently isn't the case. To date the reliability of tripleo ci has been
heavily effected by network glitches, availability of network resources and
reliability of the CI clouds. This spec is intended to deal with the problems
we have been seeing.

**Problem :** Reliability of hp1 (hp1_reliability_)
  Intermittent failures on jobs running on the hp1 cloud have been causing a
  large number of job failures and sometimes taking this region down
  altogether.  Current thinking is that the root of most of these issues is
  problems with a mellanox driver.

**Problem :** Unreliable access to network resources (net_reliability_)
  Gaining reliable access to various network resources has been inconsistent
  causing a CI outage when any one network resource is unavailable. Also
  inconsistent speeds downloading these resources can make it difficult to
  gauge overall speed improvements made to tripleo.

**Problem :** (system_health_) The health of the overall CI system isn't
  immediately obvious, problems often persist for hours (or occasionally days)
  before we react to them.

**Problem :**  (ci_run_times_) The tripleo devtest story takes time to run,
  this uses up CI resources and developer's time, where possible we should
  reduce the time required to run devtest.

**Problem :** (inefficient_usage_) Hardware on which to run tripleo is a finite
  resource, there is a spec in place to run devtest on an openstack
  deployment[1], this is the best way forward in order to use the resources we
  have in the most efficient way possible. We also have a number of options to
  explore that would help minimise resource wastage.

**Problem :** (system_feedback_) Our CI provides no feedback about trends.
  A good CI system should be more than a system that reports pass or fail, we
  should be getting feedback on metrics allowing us to observe degradations,
  where possible we should make use of services already provided by infra.
  This will allow us to proactively intervene as CI begins to degrade?

**Problem :** (bug_frequency_) We currently have no indication of which CI
  bugs are occurring most often. This frustrates efforts to make CI more
  reliable.

**Problem :** (test_coverage_) Currently CI only tests a subset of what it
  should.


Proposed Change
===============

There are a number of changes required in order to address the problems we have
been seeing, each listed here (in order of priority).

.. _hp1_reliability:

**Solution :**

* Temporarily scale back on CI by removing one of the overcloud jobs (so rh1 has
  the capacity to run CI Solo).
* Remove hp1 from the configuration.
* Run burn-in tests on each hp1 host, removing(or repairing) failing hosts.
  Burn-in tests should consist of running CI on a newly deployed cloud matching
  the load expected to run on the region. Any failure rate should not exceed
  that of currently deployed regions.
* Redeploy testing infrastructure on hp1 and test with tempest, this redeploy
  should be done with our tripleo scripts so it can be repeated and we
  are sure of parity between ci-overcloud deployments.
* Place hp1 back into CI and monitor situation.
* Add back any removed CI jobs.
* Ensure burn-in / tempest tests are followed on future regions being deployed.
* Attempts should be made to deal with problems that develop on already
  deployed clouds, if it becomes obvious they can't be quickly dealt with after
  48 hours they should be temporarily removed from the CI infrastructure and will
  need to pass the burn-in tests before being added back into production.

.. _net_reliability:

**Solution :**

* Deploy a mirror of pypi.openstack.org on each Region.
* Deploy a mirror of the Fedora and Ubuntu package repositories on each region.
* Deploy squid in each region and cache http traffic through it, mirroring
  where possible should be considered our preference but having squid in place
  should cache any resources not mirrored.
* Mirror other resources (e.g. github.com, percona tarballs etc..).
* Any new requirements added to devtest should be cachable with caches in
  place before the requirement is added.

.. _system_health:

**Solution :**

* Monitor our CI clouds and testenvs with Icinga, monitoring should include
  ping, starting (and connecting to) new instances, disk usage etc....
* Monitor CI test results and trigger an alert if "X" number of jobs of the
  same type fail in succession. An example of using logstash to monitor CI
  results can be found here[5].

Once consistency is no longer a problem we will investigate speed improvements
we can make on the speed of CI jobs.

.. _ci_run_times:

**Solution :**

* Investigate if unsafe disk caching strategies will speed up disk image
  creation, if an improvement is found implement it in production CI by one of

  * run "unsafe" disk caching strategy on ci cloud VM's (would involve exposing
    this libvirt option via the nova api).
  * use "eatmydata" to noop disk sync system calls, not currently
    packaged for F20 but we could try and restart that process[2].


.. _inefficient_usage:

**Solution :**

* Abandon on failure : adding a feature to zuul (or turning it on if it already
  exists) to abandon all jobs in a queue for a particular commit as soon as a
  voting commit fails. This would minimize usage of resources running long
  running jobs that we already know will have to be rechecked.

* Adding the collectl element to compute nodes and testenv hosts will allow us
  to find bottle necks and also identify places where it is safe to overcommit
  (e.g. we may find that overcommitting CPU a lot on testenv hosts is viable).

.. _system_feedback:

**Solution :**

* Using a combination of logstash and graphite

  * Output graphs of occurrences of false negative test results.
  * Output graphs of CI run times over time in order to identify trends.
  * Output graphs of CI job peak memory usage over time.
  * Output graphs of CI image sizes over time.

.. _bug_frequency:

**Solution :**

* In order to be able to track false negatives that are hurting us most we
  should agree not to use "recheck no bug", instead recheck with the
  relevant bug number. Adding signatures to Elastic recheck for known CI
  issues should help uptake of this.

.. _test_coverage:

**Solution :**

* Run tempest against the deployed overcloud.
* Test our upgrade story by upgrading to a new images. Initially to avoid
  having to build new images we can edit something on the overcloud qcow images
  in place in order to get a set of images to upgrade too[3].


Alternatives
------------

* As an alternative to deploying our own distro mirrors we could simply point
  directly at a mirror known to be reliable. This is undesirable as a long
  term solution as we still can't control outages.

Security Impact
---------------

None

Other End User Impact
---------------------

* No longer using recheck no bug places a burden on developers to
  investigate why a job failed.

* Adding coverage to our tests will increase the overall time to run a job.

Performance Impact
------------------

Performance of CI should improve overall.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  derekh

Other contributors:
  looking for volunteers...


Work Items
----------

* hp1 upgrade to trusty.
* Potential pypi mirror.
* Fedora Mirrors.
* Ubuntu Mirrors.
* Mirroring other non distro resources.
* Per region caching proxy.
* Document CI.
* Running an unsafe disk caching strategy in the overcloud nodes.
* ZUUL abandon on failure.
* Include collectl on compute and testenv Hosts and analyse output.
* Mechanism to monitor CI run times.
* Mechanism to monitor nodepool connection failures to instances.
* Remove ability to recheck no bug or at the very least discourage its use.
* Monitoring cloud/testenv health.
* Expand ci to include tempest.
* Expand ci to include upgrades.


Dependencies
============

None

Testing
=======

CI failure rate and timings will be tracked to confirm improvements.

Documentation Impact
====================

The tripleo-ci repository needs additional documentation in order to describe
the current layout and should then be updated as changes are made.

References
==========

* [1] spec to run devtest on openstack https://review.openstack.org/#/c/92642/
* [2] eatmydata for Fedora https://bugzilla.redhat.com/show_bug.cgi?id=1007619
* [3] CI upgrades https://review.openstack.org/#/c/87758/
* [4] summit session https://etherpad.openstack.org/p/juno-summit-tripleo-ci
* [5] http://jogo.github.io/gate/tripleo.html
