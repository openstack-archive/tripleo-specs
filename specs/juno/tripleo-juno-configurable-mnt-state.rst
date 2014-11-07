..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Configurable directory for persistent and stateful data
=======================================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-configurable-mnt-state

Make the hardcoded /mnt/state path for stateful data be configurable.

Problem Description
===================

1. A hard coded directory of /mnt/state for persistent data is incompatible
with Red Hat based distros available mechanism for a stateful data path. Red
Hat based distros, such as Fedora, RHEL, and CentOS, have a feature that uses
bind mounts for mounting paths onto a stateful data partition and does not
require manually reconfiguring software to use /mnt/state.

2. Distros that use SELinux have pre-existing policy that allows access to
specific paths. Reconfiguring these paths to be under /mnt/state, results
in SELinux denials for existing services, requiring additional policy to be
written and maintained.

3. Some operators and administrators find the reconfiguring of many services to
not use well known default values for filesystem paths to be disruptive and
inconsistent. They do not expect these changes when using a distro that they've
come to learn and anticipate certain configurations. These types of changes
also require documentation changes to existing documents and processes.


Proposed Change
===============
Deployers will be able to choose a configurable path instead of the hardcoded
value of /mnt/state for the stateful path.

A new element, stateful-path will be added that defines the value for the
stateful path. The default will be /mnt/state.

There are 3 areas that need to respect the configurable path:

os-apply-config template generation
   The stateful-path element will set the stateful path value by installing a
   JSON file to a well known location for os-collect-config to use as a local
   data source. This will require a new local data source collector to be added
   to os-collect-config (See `Dependencies`_).

   The JSON file's contents will be based on $STATEFUL_PATH, e.g.:

    {'stateful-path': '/mnt/state'}

   File templates (files under os-apply-config in an element) will then be
   updated to replace the hard coded /mnt/state with {{stateful-path}}.

   Currently, there is a mix of root locations of the os-apply-config templates.
   Most are written under /, although some are written under /mnt/state. The
   /mnt/state is hard coded in the directory tree under os-apply-config in these
   elements, so this will be removed to have the templates just written under /.
   Symlinks could instead be used in these elements to setup the correct paths.
   Support can also be added to os-apply-config's control file mechanism to
   indicate these files should be written under the stateful path. An example
   patch that does this is at: https://review.openstack.org/#/c/113651/

os-refresh-config scripts run at boot time
   In order to make the stateful path configurable, all of the hard coded
   references to /mnt/state in os-refresh-config scripts will be replaced with an
   environment variable, $STATEFUL_PATH.

   The stateful-path element will provide an environment.d script for
   os-refresh-config that reads the value from os-apply-config:

    export STATEFUL_PATH=$(os-apply-config --key stateful-path --type raw)

Hook scripts run at image build time
   The stateful-path element will provide an environment.d script for use at
   image build time:

    export STATEFUL_PATH=${STATEFUL_PATH:-"/mnt/state"}

The use-ephemeral element will depend on the stateful-path element, effectively
making the default stateful path remain /mnt/state.

The stateful path can be reconfigured by defining $STATEFUL_PATH either A) in
the environment before an image build; or B) in an element with an
environment.d script which runs earlier than the stateful-path environment.d
script.


Alternatives
------------
None come to mind, the point of this spec is to enable an alternative to what's
already existing. There may be additional alternatives out there other folks
may wish to add support for.

Security Impact
---------------
None

Other End User Impact
---------------------
End users using elements that change the stateful path location from /mnt/state
to something else will see this change reflected in configuration files and in
the directories used for persistent and stateful data. They will have to know
how the stateful path is configured and accessed.

Different TripleO installs would appear different if used with elements that
configured the stateful path differently.

This also adds some complexity when reading TripleO code, because instead of
there being an explicit path, there would instead be a reference to a
configurable value.

Performance Impact
------------------
There will be additional logic in os-refresh-config to determine and set the
stateful path, and an additional local collector that os-collect-config would
use. However, these are negligible in terms of negatively impacting
performance.


Other Deployer Impact
---------------------
Deployers will be able to choose different elements that may reconfigure the
stateful path or change the value for $STATEFUL_PATH. The default will remain
unchanged however.

Deployers would have to know what the stateful path is, and if it's different
across their environment, this could be confusing. However, this seems unlikely
as deployers are likely to be standardizing on one set of common elements,
distro, etc.

In the future, if TripleO CI and CD clouds that are based on Red Hat distros
make use of this feature to enable Red Hat read only root support, then these
clouds would be configured differently from clouds that are configured to use
/mnt/state. As a team, the tripleo-cd-admins will have to know which
configuration has been used.

Developer Impact
----------------
1. Developers need to use the $STATEFUL_PATH and {{stateful-path}}
substitutions when they intend to refer to the stateful path.

2. Code that needs to know the stateful path will need access to the variable
defining the path, it won't be able to assume the path is /mnt/state. A call to
os-apply-config to query the key defining the path could be done to get
the value, as long as os-collect-config has already run at least once.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  james-slagle

Work Items
----------

tripleo-incubator
^^^^^^^^^^^^^^^^^
* Update troubleshooting docs to mention that /mnt/state is a configurable
  path, and could be different in local environments.

tripleo-image-elements
^^^^^^^^^^^^^^^^^^^^^^
* Add a new stateful-path element that configures stateful-path and $STATEFUL_PATH
  to /mnt/state
* Update os-apply-config templates to replace /mnt/state with {{stateful-path}}
* Update os-refresh-config scripts to replace /mnt/state with $STATEFUL_PATH
* Update all elements that have os-apply-config template files under /mnt/state
  to just be under /.

  * update os-apply-config element to call os-apply-config with a --root
    $STATEFUL_PATH option
  * update elements that have paths to os-apply-config generated files (such
    as /etc/nova/nova.conf) to refer to those paths as
    $STATEFUL_PATH/path/to/file.

* make use-ephemeral element depend on stateful-path element

Dependencies
============
1. os-collect-config will need a new feature to read from a local data source
   directory that elements can install JSON files into, such as a source.d. There
   will be a new spec filed on this feature.
   https://review.openstack.org/#/c/100965/

2. os-apply-config will need an option in its control file to support
   generating templates under the configurable stateful path. There is a patch
   here: https://review.openstack.org/#/c/113651/


Testing
=======

There is currently no testing that all stateful and persistent data is actually
written to a stateful partition.

We should add tempest tests that directly exercise the preserve_ephemeral
option, and have tests that check that all stateful data has been preserved
across a "nova rebuild". Tempest seems like a reasonable place to add these
tests since preserve_ephemeral is a Nova OpenStack feature. Plus, once TripleO
CI is running tempest against the deployed OverCloud, we will be testing this
feature.

We should also test in TripleO CI that state is preserved across a rebuild by
adding stateful data before a rebuild and verifying it is still present after a
rebuild.

Documentation Impact
====================

We will document the new stateful-path element.

TripleO documentation will need to mention the potential difference in
configuration files and the location of persistent data if a value other than
/mnt/state is used.


References
==========

os-collect-config local datasource collector spec:

* https://review.openstack.org/100965

Red Hat style stateful partition support this will enable:

* https://git.fedorahosted.org/cgit/initscripts.git/tree/systemd/fedora-readonly
* https://git.fedorahosted.org/cgit/initscripts.git/tree/sysconfig/readonly-root
* https://git.fedorahosted.org/cgit/initscripts.git/tree/statetab
* https://git.fedorahosted.org/cgit/initscripts.git/tree/rwtab
