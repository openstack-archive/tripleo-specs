==========
TripleO UI
==========

We need a graphical user interface that will support deploying OpenStack using
TripleO.

Problem Description
===================

Tuskar-UI, the only currently existing GUI capable of TripleO deployments, has
several significant issues.

Firstly, its back-end relies on an obsolete version of the Tuskar API, which is
insufficient for complex overcloud deployments.

Secondly, it is implemented as a Horizon plugin and placed under the Horizon
umbrella, which has proven to be suboptimal, for several reasons:

 * The placement under the Horizon program. In order to be able to develop the
   Tuskar-UI, one needs deep familiarity with both Horizon and TripleO projects.
   Furthermore, in order to be able to approve patches, one needs to be a
   Horizon core reviewer. This restriction reduces the number of people who can
   contribute drastically, as well as makes it hard for Tuskar-UI developers to
   actually land code.

 * The complexity of the Horizon Django application. Horizon is a very complex
   heavyweight application comprised of many OpenStack services. It has become
   very large, inflexible and consists of several unnecessary middle layers. As
   a result of this, we have been witnessing the emergence of several new GUIs
   implemented as independent (usually fully client-side JavaScript) applications,
   rather than as Horizon plugins. Ironic webclient[1] is one such example. This
   downside of Horizon has been recognized and an attempt to address it is
   described in the next point.

 * The move to Angular JS (version 1). In an attempt to address the issues listed
   above, the Horizon community decided to rewrite it in Angular JS. However,
   instead of doing a total rewrite, they opted for a more gradual approach,
   resulting in even more middle layers (the original Django layer turned into an
   API for Angular based front end). Although the intention is to eventually
   get rid of the unwanted layers, the move is happening very slowly. In
   addition, this rewrite of Horizon is to AngularJS version 1, which may soon
   become obsolete, with version 2 just around the corner. This probably means
   another complete rewrite in not too distant future.

 * Packaging issues. The move to AngularJS brought along a new set of issues
   related to the poor state of packaging of nodejs based tooling in all major
   Linux distributions.

Proposed Change
===============

Overview
--------

In order to address the need for a TripleO based GUI, while avoiding the issues
listed above, we propose introducing a new GUI project, *TripleO UI*, under the
TripleO program.

As it is a TripleO specific UI, TripleO GUI will be placed under the TripleO
program, which will bring it to attention of TripleO reviewers and allow
TripleO core reviewers to approve patches. This should facilitate the code
contribution process.

TripleO UI will be a web UI designed for overcloud deployment and
management. It will be a lightweight, independent client-side application,
designed for flexibility, adaptability and reusability.

TripleO UI will be a fully client-side JavaScript application. It will be
stateless and contain no business logic. It will consume the TripleO REST API[2],
which will expose the overcloud deployment workflow business logic implemented
in the tripleo-common library[3]. As opposed to the previous architecture which
included many unwanted middle layers, this one will be very simple, consisting
only of the REST API serving JSON, and the client-side JavaScript application
consuming it.

The development stack will consist of ReactJS[4] and Flux[5]. We will use ReactJS
to implement the web UI components, and Flux for architecture design.

Due to the packaging problems described above, we will not provide any packages
for the application for now. We will simply make the code available for use.

Alternatives
------------

The alternative is to keep developing Tuskar-UI under the Horizon umbrella. In
addition to all the problems outlined above, this approach would also mean a
complete re-write of Tuskar-UI back-end to make it use the new tripleo-common
library.

Security Impact
---------------

This proposal introduces a brand new application; all the standard security
concerns which come with building a client-side web application apply.

Other End User Impact
---------------------

We plan to build a standalone web UI which will be capable of deploying
OpenStack with TripleO. Since as of now no such GUIs exist, this can be a huge
boost for adoption of TripleO.

Performance Impact
------------------

The proposed technology stack, ReactJS and Flux, have excellent performance
characteristics. TripleO UI should be a lightweight, fast, flexible application.


Other Deployer Impact
---------------------

None

Developer Impact
----------------

Right now, development on Tuskar-UI is uncomfortable for the reasons
detailed above. This proposal should result in more comfortable development
as it logically places TripleO UI under the TripleO program, which brings
it under the direct attention of TripleO developers and core reviewers.

Implementation
==============

Assignee(s)
-----------
Primary assignees:

* jtomasek
* flfuchs
* jrist
* <TBD person with JS & CI skills>

Work Items
----------

This is a general proposal regarding the adoption of a new graphical user
interface under the TripleO program. The implementation of specific features
will be covered in subsequent proposals.

Dependencies
============

We are dependent upon the creation of the TripleO REST API[2], which in turn
depends on the tripleo-common[3] library containing all the functionality
necessary for advanced overcloud deployment.

Alternatively, using Mistral to provide a REST API, instead of building a new
API, is currently being investigated as another option.

Testing
=======

TripleO UI should be thoroughly tested, including unit tests and integration
tests. Every new feature and bug fix should be accompanied by appropriate tests.

The TripleO CI should be updated to test the TripleO UI.

Documentation Impact
====================

TripleO UI will have to be well-documented and meet OpenStack standards.
We will need both developer and deployment documentation. Documentation will
live in the tripleo-docs repository.

References
==========

[1] https://github.com/openstack/ironic-webclient
[2] https://review.openstack.org/#/c/230432
[3] http://specs.openstack.org/openstack/tripleo-specs/specs/mitaka/tripleo-overcloud-deployment-library.html
[4] https://facebook.github.io/react/
[5] https://facebook.github.io/flux/
