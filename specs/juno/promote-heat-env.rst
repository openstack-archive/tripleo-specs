..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
Promote HEAT_ENV
================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-promote-heat-env

Promote values set in the Heat environment file to take precedence over
input environment variables.

Problem Description
===================

Historically TripleO scripts have consulted the environment for many items of
configuration. This raises risks of scope leakage and the number of environment
variables required often forces users to manage their environment with scripts.
Consequently, there's a push to prefer data files like the Heat environment
file (HEAT_ENV) which may be set by passing -e to Heat. To allow this file to
provide an unambiguous source of truth, the environment must not be allowed to
override the values from this file. That is to say, precedence must be
transferred.

A key distinction is whether the value of an environment variable is obtained
from the environment passed to it by its parent process (either directly or
through derivation). Those which are will be referred to as "input variables"
and are deprecated by this spec. Those which are not will be called "local
variables" and may be introduced freely. Variables containing values
synthesised from multiple sources must be handled on a case-by-case basis.


Proposed Change
===============

Since changes I5b7c8a27a9348d850d1a6e4ab79304cf13697828 and
I42a9d4b85edcc99d13f7525e964baf214cdb7cbf, ENV_JSON (the contents of the file
named by HEAT_ENV) is constructed in devtest_undercloud.sh like so::

  ENV_JSON=$(jq '.parameters = {
    "MysqlInnodbBufferPoolSize": 100
  } + .parameters + {
    "AdminPassword": "'"${UNDERCLOUD_ADMIN_PASSWORD}"'",
    "AdminToken": "'"${UNDERCLOUD_ADMIN_TOKEN}"'",
    "CeilometerPassword": "'"${UNDERCLOUD_CEILOMETER_PASSWORD}"'",
    "GlancePassword": "'"${UNDERCLOUD_GLANCE_PASSWORD}"'",
    "HeatPassword": "'"${UNDERCLOUD_HEAT_PASSWORD}"'",
    "NovaPassword": "'"${UNDERCLOUD_NOVA_PASSWORD}"'",
    "NeutronPassword": "'"${UNDERCLOUD_NEUTRON_PASSWORD}"'",
    "NeutronPublicInterface": "'"${NeutronPublicInterface}"'",
    "undercloudImage": "'"${UNDERCLOUD_ID}"'",
    "BaremetalArch": "'"${NODE_ARCH}"'",
    "PowerSSHPrivateKey": "'"${POWER_KEY}"'",
    "NtpServer": "'"${UNDERCLOUD_NTP_SERVER}"'"
  }' <<< $ENV_JSON)

This is broadly equivalent to "A + B + C", where values from B override those
from A and values from C override those from either. Currently section C
contains a mix of input variables and local variables. It is proposed that
current and future environment variables are allocated such that:

* A only contains default values.
* B is the contents of the HEAT_ENV file (from either the user or a prior run).
* C only contains computed values (from local variables).

The following are currently in section C but are not local vars::

  NeutronPublicInterface (default 'eth0')
  UNDERCLOUD_NTP_SERVER (default '')

The input variables will be ignored and the defaults moved into section A::

  ENV_JSON=$(jq '.parameters = {
    "MysqlInnodbBufferPoolSize": 100,
    "NeutronPublicInterface": "eth0",
    "NtpServer": ""
  } + .parameters + {
    ... elided ...
  }' <<< $ENV_JSON)

devtest_overcloud.sh will be dealt with similarly. These are the variables
which need to be removed and their defaults added to section A::

  OVERCLOUD_NAME (default '')
  OVERCLOUD_HYPERVISOR_PHYSICAL_BRIDGE (default '')
  OVERCLOUD_HYPERVISOR_PUBLIC_INTERFACE (default '')
  OVERCLOUD_BRIDGE_MAPPINGS (default '')
  OVERCLOUD_FLAT_NETWORKS (default '')
  NeutronPublicInterface (default 'eth0')
  OVERCLOUD_LIBVIRT_TYPE (default 'qemu')
  OVERCLOUD_NTP_SERVER (default '')

Only one out of all these input variables is used outside of these two scripts
and consequently the rest are safe to remove.

The exception is OVERCLOUD_LIBVIRT_TYPE. This is saved by the script
'write-tripleorc'. As it will now be preserved in HEAT_ENV, it does not need to
also be preserved by write-tripleorc and can be removed from there.

----

So that users know they need to start setting these values through HEAT_ENV
rather than input variables, it is further proposed that for an interim period
each script echo a message to STDERR if deprecated input variables are set. For
example::

  for OLD_VAR in OVERCLOUD_NAME; do
    if [ ! -z "${!OLD_VAR}" ]; then
      echo "WARNING: ${OLD_VAR} is deprecated, please set this in the" \
           "HEAT_ENV file (${HEAT_ENV})" 1>&2
    fi
  done

----

To separate user input from generated values further, it is proposed that user
values be read from a new file - USER_HEAT_ENV. This will default to
{under,over}cloud-user-env.json. A new commandline parameter, --user-heat-env,
will be added to both scripts so that this can be changed.

#. ENV_JSON is initialised with default values.
#. ENV_JSON is overlaid by HEAT_ENV.
#. ENV_JSON is overlaid by USER_HEAT_ENV.
#. ENV_JSON is overlaid by computed values.
#. ENV_JSON is saved to HEAT_ENV.

See http://paste.openstack.org/show/83551/ for an example of how to accomplish
this. In short::

  ENV_JSON=$(cat ${HEAT_ENV} ${USER_HEAT_ENV} | jq -s '
    .[0] + .[1] + {"parameters":
      ({..defaults..} + .[0].parameters + {..computed..} + .[1].parameters)}')
  cat > "${HEAT_ENV}" <<< ${ENV_JSON}

Choosing to move user data into a new file, compared to moving the merged data,
makes USER_HEAT_ENV optional. If users wish, they can continue providing their
values in HEAT_ENV. The complementary solution requires users to clean
precomputed values out of HEAT_ENV, or they risk unintentionally preventing the
values from being recomputed.

Loading computed values after user values sacrifices user control in favour of
correctness. Considering that any devtest user must be rather technical, if a
computation is incorrect they can fix or at least hack the computation
themselves.

Alternatives
------------

Instead of removing the input variables entirely, an interim form could be
used::

  ENV_JSON=$(jq '.parameters = {
    "MysqlInnodbBufferPoolSize": 100,
    "NeutronPublicInterface": "'"${NeutronPublicInterface}"'",
    "NtpServer": "'"${UNDERCLOUD_NTP_SERVER}"'"
  } + .parameters + {
    ...
  }

However, the input variables would only have an effect if the keys they affect
are not present in HEAT_ENV. As HEAT_ENV is written each time devtest runs, the
keys will usually be present unless the file is deleted each time (rendering it
pointless). So this form is more likely to cause confusion than aid
transition.

----

jq includes an 'alternative operator', ``//``, which is intended for providing
defaults::

  A filter of the form a // b produces the same results as a, if a produces
  results other than false and null. Otherwise, a // b produces the same
  results as b.

This has not been used in the proposal for two reasons:

#. It only works on individual keys, not whole maps.
#. It doesn't work in jq 1.2, still included by Ubuntu 13.04 (Saucy).

Security Impact
---------------

None.

Other End User Impact
---------------------

An announcement will be made on the mailing list when this change merges. This
coupled with the warnings given if the deprecated variables are set should
provide sufficient notice.

As HEAT_ENV is rewritten every time devtest executes, we can safely assume it
matches the last environment used. However users who use scripts to switch
their environment may be surprised. Overall the change should be a benefit to
these users, as they can use two separate HEAT_ENV files (passing --heat-env to
specify which to activate) instead of needing to maintain scripts to set up
their environment and risking settings leaking from one to the other.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

lxsli

Work Items
----------

* Add USER_HEAT_ENV to both scripts.
* Move variables in both scripts.
* Add deprecated variables warning to both scripts.
* Remove OVERCLOUD_LIBVIRT_TYPE from write-tripleorc.


Dependencies
============

None.


Testing
=======

The change will be tested in isolation from the rest of the script.


Documentation Impact
====================

* Update usage docs with env var deprecation warnings.
* Update usage docs to recommend HEAT_ENV.


References
==========

#. http://stedolan.github.io/jq/manual/ - JQ manual
#. http://jqplay.herokuapp.com/ - JQ interactive demo
