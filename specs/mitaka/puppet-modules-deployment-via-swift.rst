..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Puppet Module Deployment via Swift
==================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/puppet-modules-deployment-via-swift

The ability to deploy a local directory of puppet modules to an overcloud
using the OpenStack swift object service.

Problem Description
===================

When deploying puppet modules to the overcloud there are currently three
 options:

 * pre-install the puppet modules into a "golden" image. You can pre-install
   modules via git sources or by using a distro package.

 * use a "firstboot" script to rsync the modules from the undercloud (or
   some other rsync server that is available).

 * post-install the puppet modules via a package upgrade onto a running
   Overcloud server by using a (RPM, Deb, etc.)

None of the above mechanisms provides an easy workflow when making
minor (ad-hoc) changes to the puppet modules and only distro packages can be
used to provide updated puppet modules to an already deployed overcloud.
While we do have a way to rsync over updated modules on "firstboot" via
rsync this isn't a useful mechanism for operator who may wish to
use heat stack-update to deploy puppet changes without having to build
a new RPM/Deb package for each revision.

Proposed Change
===============

Overview
--------

Create an optional (opt-in) workflow that if enabled will allow an operator
to create and deploy a local artifact (tarball, distro package, etc.) of
puppet modules to a new or existing overcloud via heat stack-create and
stack-update.  The mechanism would use the OpenStack object store service
(rather than rsync) which we already have available on the undercloud.
The new workflow would work like this:

  * A puppet modules artifact (tarball, distro package, etc.) would be uploaded
    into a swift container.

  * The container would be configured so that a Swift Temp URL can be generated

  * A Swift Temp URL would be generated for the puppet modules URL that is
    stored in swift

  * A heat environment would be generated which sets a DeployArtifactURLs
    parameter to this swift URL. (the parameter could be a list so that
    multiple URLs could also be downloaded.)

  * The TripleO Heat Templates would be modified so that they include a new
    'script' step which if it detects a custom DeployArtifactURLs parameter
    would automatically download the artifact from the provided URL, and
    deploy it locally on each overcloud role during the deployment workflow.
    By "deploy locally" we mean a tarball would be extracted, and RPM would
    get installed, etc. The actual deployment mechanism will be pluggable
    such that both tarballs and distro packages will be supported and future
    additions might be added as well so long as they also fit into the generic
    DeployArtifactURLs abstraction.

  * The Operator could then use the generated heat environment to deploy
    a new set of puppet modules via heat stack-create or heat stack-update.

  * TripleO client could be modified so that it automically loads
    generated heat environments in a convienent location. This (optional)
    extra step would make enabling the above workflow transparent and
    only require the operator to run a 'upload-puppet-modules' tool to
    upload and configure new puppet modules for deployment via Swift.

Alternatives
------------

There are many alternatives we could use to obtain a similar workflow that
allows the operator to more deploy puppet modules from a local directory:

  * Setting up a puppet master would allow a similar workflow. The downside
    of this approach is that it would require a bit of overhead, and it
    is puppet specific (the deployment mechanism would need to be re-worked
    if we ever had other types of on-disk files to update).

  * Rsync. We already support rsync for firstboot scripts. The downside of
    rsync is it requires extra setup, and doesn't have an API like
    OpenStack swift does allowing for local or remote management and updates
    to the puppet modules.

Security Impact
---------------

The new deployment would use a Swift Temp URL over HTTP/HTTPS. The duration
of the Swift Temp URL's can be controlled when they are signed via
swift-temp-url if extra security is desired. By using a Swift Temp URL we
avoid the need to pass the administrators credentials onto each overcloud
node for swiftclient and instead can simply use curl (or wget) to download
the updated puppet modules. Given we already deploy images over http/https
using an undercloud the use of Swift in this manner should pose minimal extra
security risks.

Other End User Impact
---------------------

The ability to deploy puppet modules via Swift will be opt-in so the
impact on end users would be minimal. The heat templates will contain
a new script deployment that may take a few extra seconds to deploy on
each node (even if the feature is not enabled). We could avoid the extra
deployment time perhaps by noop'ing out the heat resource for the new
swift puppet module deployment.

Performance Impact
------------------

Developers and Operators would likely be able to deploy puppet module changes
more quickly (without having to create a distro package). The actual deployment
of puppet modules via swift (downloading and extracting the tarball) would
likely be just as fast as as a tarball.

Other Deployer Impact
---------------------

None.


Developer Impact
----------------

Being able to more easily deploy updated puppet modules to an overcloud would
likely speed up the development update and testing cycle of puppet modules.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dan-prince

Work Items
----------

 * Create an upload-puppet-modules script in tripleo-common. Initially this
   may be a bash script which we ultimately refine into a Python version if
   it proves useful.

 * Modify tripleo-heat-templates so that it supports a DeployArtifactURLs
   parameter (if the parameter is set) attempt to deploy the list of
   files from this parameter. The actual contents of the file might be
   a tarball or a distribution package (RPM).

 * Modify tripleoclient so that the workflow around using upload-puppet-modules
   can be "transparent". Simply running upload-puppet-modules would not only
   upload the puppet modules it would also generate a Heat environment that
   would then automatically configure heat stack-update/create commands
   to use the new URL via a custom heat environment.

 * Update our CI scripts in tripleo-ci and/or tripleo-common so that we
   make use of the new Puppet modules deployment mechanism.

 * Update tripleo-docs to make note of the new feature.

Dependencies
============

None.

Testing
=======

We would likely want to switch to use this feature in our CI because
it allows us to avoid git cloning the same puppet modules for both
the undercloud and overcloud nodes. Simply calling the extra
upload-puppet-modules script on the undercloud as part of our
deployment workflow would enable the feature and allow it to be tested.

Documentation Impact
====================

We would need to document the additional (optional) workflow associated
with deploying puppet modules via Swift.


References
==========

 * https://review.openstack.org/#/c/245314/ (Add support for DeployArtifactURLs)
 * https://review.openstack.org/#/c/245310/ (Add scripts/upload-swift-artifacts)
 * https://review.openstack.org/#/c/245172/ (tripleoclient --environment)
