..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========
GUI logging
===========

The TripleO GUI currently has no way to persist logging information.

Problem Description
===================

The TripleO GUI is a web application without its own dedicated backend.  As
such, any and all client-side errors are lost when the End User reloads the page
or navigates away from the application.  When things go wrong, the End User is
unable to retrieve client-side logs because this information is not persisted.

Proposed Change
===============

Overview
--------

I propose that we use Zaqar as a persistence backend for client-side logging.
At present, the web application is already communicating with Zaqar using
websockets.  We can use this connection to publish new messages to a dedicated
logging queue.

Zaqar messages have a TTL of one hour.  So once every thirty minutes, Mistral
will query Zaqar using crontrigger, and retrieve all messages from the
``tripleo-ui-logging`` queue.  Mistral will then look for a file called
``tripleo-ui-log`` in Swift.  If this file exists, Mistral will check its size.
If the size exceeds a predetermined size (e.g. 10MB), Mistral will rename it to
``tripleo-ui-log-<timestamp>``, and create a new file in its place.  The file
will then receive the messages from Zaqar, one per line.  Once we reach, let's
say, a hundred archives (about 1GB) we can start removing dropping data in order
to prevent unnecessary data accoumulation.

To view the logging data, we can ask Swift for 10 latest messages with a prefix
of ``tripleo-ui-log``.  These files can be presented in the GUI for download.
Should the user require, we can present a "View more" link that will display the
rest of the collected files.

Alternatives
------------

None at this time

Security Impact
---------------

There is a chance of logging sensitive data.  I propose that we apply some
common scrubbing mechanism to the messages before they are stored in Swift.

Other End User Impact
---------------------

Performance Impact
------------------

Sending additional messages over an existing websocket connection should have
a negligible performance impact on the web application.  Likewise, running
hourly cron tasks in Mistral shouldn't impose a significant burden on the
undercloud machine.

Other Deployer Impact
---------------------

Developer Impact
----------------

Developers should also benefit from having a centralized logging system in
place as a means of improving productivity when debugging.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hpokorny

Work Items
----------

* Introduce a central logging system (already in progress, see `blueprint`_)
* Introduce a global error handler
* Convert all logging messages to JSON using a standard format
* Configuration: the name for the Zaqar queue to carry the logging data
* Introduce a Mistral workflow to drain a Zaqar queue and publish the acquired
  data to a file in Swift
* Introduce GUI elements to download the log files

Dependencies
============

Testing
=======

We can write unit tests for the code that handles sending messages over the
websocket connection.  We might be able to write an integration smoke test that
will ensure that a message is received by the undercloud.  We can also add some
testing code to tripleo-common to cover the logic that drains the queue, and
publishes the log data to Swift.

Documentation Impact
====================

We need to document the default name of the Zaqar queue, the maximum size of
each log file, and how many log files can be stored at most.  On the End User
side, we should document the fact that a GUI-oriented log is available, and the
way to get it.

References
==========

.. _blueprint: https://blueprints.launchpad.net/tripleo/+spec/websocket-logging
