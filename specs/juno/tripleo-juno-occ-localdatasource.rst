..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
os-collect-config local data source
===================================


https://blueprints.launchpad.net/tripleo-juno-occ-local-datasource

os-collect-config needs a local data source collector for configuration data.
This will allow individual elements to drop files into a well-known location to
set the initial configuration data of an instance.

There is already a heat_local collector, but that uses a single hard coded path
of /var/lib/heat-cfntools/cfn-init-data.

Problem Description
===================

* Individual elements can not currently influence the configuration available
  to os-apply-config for an instance without overwriting each other.
* Elements that rely on configuration values that must be set the same at both
  image build time and instance run time currently have no way of propagating the
  value used at build time to a run time value.
* Elements have no way to specify default values for configuration they may
  need at runtime (outside of configuration file templates).


Proposed Change
===============

A new collector class will be added to os-collect-config that collects
configuration data from JSON files in a configurable list of directories with a
well known default of /var/lib/os-collect-config/local-data.

The collector will return a list of pairs of JSON files and their content,
sorted by the JSON filename in traditional C collation.  For example, if
/var/lib/os-collect-config/local-data contains bar.json and foo.json

    [ ('bar.json', bar_content),
      ('foo.json', foo_content) ]

This new collector will be configured first in DEFAULT_COLLECTORS in
os-collect-config. This means all later configured collectors will override any
shared configuration keys from the local datasource collector.

Elements making use of this feature can install a json file into the
/var/lib/os-collect-config/local-data directory. The os-collect-config element
will be responsible for creating the /var/lib/os-collect-config/local-data
directory at build time and will create it with 0755 permissions.

Alternatives
------------

OS_CONFIG_FILES
^^^^^^^^^^^^^^^
There is already a mechanism in os-apply-config to specify arbitrary files to
look at for configuration data via setting the OS_CONFIG_FILES environment
variable. However, this is not ideal because each call to os-apply-config would
have to be prefaced with setting OS_CONFIG_FILES, or it would need to be set
globally in the environment (via an environment.d script for instance). As an
element developer, this is not clear. Having a robust and clear documented
location to drop in configuration data will be simpler.

heat_local collector
^^^^^^^^^^^^^^^^^^^^
There is already a collector that reads from local data, but it must be
configured to read explicit file paths. This does not scale well if several
elements want to each provide local configuration data, in that you'd have to
reconfigure os-collect-config itself. We could modify the heat_local collector
to read from directories instead, while maintaining backwards compatibility as
well, instead of writing a whole new collector. However, given that collectors
are pretty simple implementations, I'm proposing just writing a new one, so
that they remain generally single purpose with clear goals.

Security Impact
---------------

* Harmful elements could drop bad configuration data into the well known
  location. This is mitigated somewhat that as a deployer, you should know and
  validate what elements you're using that may inject local configuration.

* We should verify that the local data source files are not world writable and
  are in a directory that is root owned. Checks to dib-lint could be added to
  verify this at image build time. Checks could be added to os-collect-config
  for instance run time.

Other End User Impact
---------------------

None

Performance Impact
------------------

An additional collector will be running as part of os-collect-config, but its
execution time should be minimal.

Other Deployer Impact
---------------------

* There will be an additional configuration option in os-collect-config to
  configure the list of directories to look at for configuration data. This
  will have a reasonable default and will not usually need to be changed.
* Deployers will have to consider what local data source configuration may be
  influencing their current applied configuration.

Developer Impact
----------------

We will need to make clear in documentation when to use this feature versus
what to expose in a template or specify via passthrough configuration.
Configuration needed at image build time where you need access to those values
at instance run time as well are good candidates for using this feature.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    james-slagle

Work Items
----------

* write new collector for os-collect-config
* unit tests for new collector
* document new collector
* add checks to dib-lint to verify JSON files installed to the local data
  source directory are not world writable
* add checks to os-collect-config to verify JSON files read by the local data
  collector are not world writable and that their directory is root owned.

Dependencies
============

* The configurable /mnt/state spec at:
  https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-configurable-mnt-state
  depends on this spec.

Testing
=======

Unit tests will be written for the new collector. The new collector will also
eventually be tested in CI because there will be an existing element that will
configure the persistent data directory to /mnt/state that will make use of
this implementation.


Documentation Impact
====================

The ability of elements to drop configuration data into a well known location
should be documented in tripleo-image-elements itself so folks can be made
better aware of the functionality.

References
==========

* https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-configurable-mnt-state
* https://review.openstack.org/#/c/94876
