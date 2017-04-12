====================
 Adding New CI Jobs
====================

New CI jobs need to be added following a specific process in order to ensure
they don't block patches unnecessarily and that they aren't ignored by
developers.

Problem Description
===================

We need to have a process for adding CI jobs that is not going to result
in a lot of spurious failures due to the new jobs.  Bogus CI results force
additional rechecks and reduce developer/reviewer confidence in the results.

In addition, maintaining CI jobs is a non-trivial task, and each one we add
increases the load on the team.  Hopefully having a process that requires the
involvement of the new job's proposer makes it clear that the person/team
adding the job has a responsibility to help maintain it.  CI is everyone's
problem.

Policy
======

The following steps should be completed in the order listed when adding a new
job:

#. Create an experimental job or hijack an existing job for a single Gerrit
   change.  See the references section for details on how to add a new job.
   This job should be passing before moving on to the next step.

#. Verify that the new job is providing a reasonable level of logging.  Not
   too much, not too little.  Important logs, such as the OpenStack service
   logs and basic system logs, are necessary to determine why jobs fail.
   However, OpenStack Infra has to store the logs from an enormous number of
   jobs, so it is also important to keep our log artifact sizes under control.
   When in doubt, try to capture about the same amount of logs as the existing
   jobs.

#. Promote the job to check non-voting.  While the job should have been
   passing prior to this, it most likely has not been run a significant number
   of times, so the overall stability is still unknown.

   "Stable" in this case would be defined as not having significantly more
   spurious failures than the ovb-ha job.  Due to the additional complexity of
   an HA deployment, that job tends to fail for reasons unrelated to the patch
   being tested more often than the other jobs.  We do not want to add any
   jobs that are less stable.  Note that failures due to legitimate problems
   being caught by the new job should not count against its stability.

   .. important:: Before adding OVB jobs to the check queue, even as
      non-voting, please check with the CI admins to ensure there is enough
      OVB capacity to run a large number of new jobs.  As of this writing,
      the OVB cloud capacity is significantly more constrained than regular
      OpenStack Infra.

   A job should remain in this state until it has been proven stable over a
   period of time.  A good rule of thumb would be that after a week of
   stability the job can and should move to the next step.

   .. important:: Jobs should not remain non-voting indefinitely.  This causes
      reviewers to ignore the results anyway, so the jobs become a waste of
      resources.  Once a job is believed to be stable, it should be made
      voting as soon as possible.

#. To assist with confirming the stability of a job, it should be added to the
   `CI Status <http://tripleo.org/cistatus.html>`_ page at this point.  This
   can actually be done at any time after the job is moved to the check queue,
   but must be done before the job becomes voting.

   Additionally, contact Sagi Shnaidman (sshnaidm on IRC) to get the job
   added to the `Extended CI Status <http://status-tripleoci.rhcloud.com/>`_
   page.

#. Send an e-mail to openstack-dev, tagged with [tripleo], that explains the
   purpose of the new job and notifies people that it is about to be made
   voting.

#. Make the job voting.  At this point there should be sufficient confidence
   in the job that reviewers can trust the results and should not merge
   anything which does not pass it.

   In addition, be aware that voting multinode jobs are also gating.  If the
   job fails the patch cannot merge.  This means a broken job can block all
   TripleO changes from merging.

#. Keep an eye on the `CI Status <http://tripleo.org/cistatus.html>`_ page to
   ensure the job keeps running smoothly.  If it starts to fail an unusual
   amount, please investigate.

Alternatives & History
======================

Historically, a number of jobs have been added to the check queue when they
were completely broken.  This is bad and reduces developer and reviewer
confidence in the CI results.  It can also block TripleO changes from merging
if the broken job is gating.

We also have a bad habit of leaving jobs in the non-voting state, which makes
them fairly worthless since reviewers will not respect the results.  Per
this policy, we should clean up all of the non-voting jobs by either moving
them back to experimental, or stabilizing them and making them voting.

Implementation
==============

Author(s)
---------

Primary author:
  bnemec

Milestones
----------

This policy would go into effect immediately.

Work Items
----------

This policy is mostly targeted at new jobs, but we do have a number of
non-voting jobs that should be brought into compliance with it.

References
==========

`OpenStack Infra Manual <https://docs.openstack.org/infra/manual/>`_

`Adding a New Job <https://docs.openstack.org/infra/manual/drivers.html#running-jobs-with-zuul>`_

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Pike
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode
