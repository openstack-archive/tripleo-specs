..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Remove merge.py from TripleO Heat Templates
===========================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-juno-remove-mergepy

``merge.py`` is where we've historically accumulated the technical debt for our
Heat templates [0]_ with the intention of migrating away from it when Heat meets
our templating needs.

Its main functionality includes combining smaller template snippets into a
single template describing the full TripleO deployment, merging certain
resources together to reduce duplication while keeping the snippets themselves
functional as standalone templates and a support for manual scaling of Heat
resources.

This spec describes the changes necessary to move towards templates
that do not depend on ``merge.py``. We will use native Heat features
where we can and document the rest, possibly driving new additions to
the Heat template format.

It is largely based on the April 2014 discussion in openstack-dev [1]_.


Problem Description
===================

Because of the mostly undocumented nature of ``merge.py`` our templates are
difficult to understand or modify by newcomers (even those already familiar with
Heat).

It has always been considered a short-term measure and Heat can now provide most
of what we need in our templates.


Proposed Change
===============

We will start with making small correctness-preserving changes to our
templates and ``merge.py`` that move us onto using more Heat native
features. Where we cannot make the change for some reason, we will
file a bug with Heat and work with them to unblock the process.

Once we get to a point where we have to do large changes to the
structure of our templates, we will split them off to new files and
enable them in our CI as parallel implementations.

Once we are confident that the new templates fulfill the same
requirements as the original ones, we will deprecate the old ones,
deprecate ``merge.py`` and switch to the new ones as the default.

The list of action items necessary for the full transition is
below.

**1. Remove the custom resource types**

TripleO Heat templates and ``merge.py`` carry two custom types that (after the
move to software config [8]_, [9]_) are no longer used for anything:

* OpenStack::ImageBuilder::Elements
* OpenStack::Role

We will drop them from the templates and deprecate in the merge tool.


**2. Remove combining whitelisted resource types**

If we have two ``AWS::AutoScaling::LaunchConfiguration`` resources with the same
name, ``merge.py`` will combine their ``Properties`` and ``Metadata``. Our
templates are no longer using this after the software-config update.


**3. Port TripleO Heat templates to HOT**

With most of the non-Heat syntax out of the way, porting our CFN/YAML templates
to pure HOT format [2]_ should be straightforward.

We will have to update ``merge.py`` as well. We should be able to support both
the old format and HOT.

We should be able to differentiate between the two by looking for the
``heat_template_version`` top-level section which is mandatory in the HOT
syntax.

Most of the changes to ``merge.py`` should be around spelling (``Parameters`` ->
``parameters``, ``Resources`` -> ``resources``) and different names for
intrinsic functions, etc. (``Fn::GetAtt`` -> ``get_attr``).

This task will require syntactic changes to all of our templates and
unfortunately, it isn't something different people can update bit by bit. We
should be able to update the undercloud and overcloud portions separately, but
we can't e.g. just update a part of the overcloud. We are still putting
templates together with ``merge.py`` at this point and we would end up with a
template that has both CFN and HOT bits.


**4. Move to Provider resources**

Heat allows passing-in multiple templates when deploying a stack. These
templates can map to custom resource types. Each template would represent a role
(compute server, controller, block storage, etc.) and its ``parameters`` and
``outputs`` would map to the custom resource's ``properties`` and
``attributes``.

These roles will be referenced from a master template (``overcloud.yaml``,
``undercloud.yaml``) and eventually wrapped in a scaling resource
(``OS::Heat::ResourceGroup`` [5]_) or whatever scaling mechanism we adopt.

.. note:: Provider resources represent fully functional standalone templates.
          Any provider resource template can be passed to Heat and turned into a
          stack or treated as a custom resource in a larger deployment.

Here's a hypothetical outline of ``compute.yaml``::

    parameters:
      flavor:
        type: string
      image:
        type: string
      amqp_host:
        type: string
      nova_compute_driver:
        type: string

    resources:
      compute_instance:
        type: OS::Nova::Server
        properties:
          flavor: {get_param: flavor}
          image: {get_param: image}

      compute_deployment:
        type: OS::Heat::StructuredDeployment
        properties:
          server: {ref: compute_instance}
          config: {ref: compute_config}
          input_values:
            amqp_host: {get_param: amqp_host}
            nova_compute_driver: {get_param: nova_compute_driver}

      compute_config:
        type: OS::Heat::StructuredConfig
          properties:
            group: os-apply-config
            config:
              amqp:
                host: {get_input: amqp_host}
              nova:
                compute_driver: {get_input: nova_compute_driver}
              ...

We will use a similar structure for all the other roles (``controller.yaml``,
``block-storage.yaml``, ``swift-storage.yaml``, etc.). That is, each role will
contain the ``OS::Nova::Server``, the associated deployments and any other
resources required (random string generators, security groups, ports, floating
IPs, etc.).

We can map the roles to custom types using Heat environments [4]_.

``role_map.yaml``: ::

    resource_registry:
      OS::TripleO::Compute: compute.yaml
      OS::TripleO::Controller: controller.yaml
      OS::TripleO::BlockStorage: block-storage.yaml
      OS::TripleO::SwiftStorage: swift-storage.yaml


Lastly, we'll have a master template that puts it all together.

``overcloud.yaml``::

    parameters:
      compute_flavor:
        type: string
      compute_image:
        type: string
      compute_amqp_host:
        type: string
      compute_driver:
        type: string
      ...

    resources:
      compute0:
        # defined in controller.yaml, type mapping in role_map.yaml
        type: OS::TripleO::Compute
        parameters:
          flavor: {get_param: compute_flavor}
          image: {get_param: compute_image}
          amqp_host: {get_param: compute_amqp_host}
          nova_compute_driver: {get_param: compute_driver}

      controller0:
        # defined in controller.yaml, type mapping in role_map.yaml
        type: OS::TripleO::Controller
        parameters:
          flavor: {get_param: controller_flavor}
          image: {get_param: controller_image}
          ...

    outputs:
      keystone_url:
        description: URL for the Overcloud Keystone service
        # `keystone_url` is an output defined in the `controller.yaml` template.
        # We're referencing it here to expose it to the Heat user.
        value: { get_attr: [controller_0, keystone_url] }

and similarly for ``undercloud.yaml``.

.. note:: The individual roles (``compute.yaml``, ``controller.yaml``) are
          structured in such a way that they can be launched as standalone
          stacks (i.e. in order to test the compute instance, one can type
          ``heat stack-create -f compute.yaml -P ...``). Indeed, Heat treats
          provider resources as nested stacks internally.


**5. Remove FileInclude from ``merge.py``**

The goal of ``FileInclude`` was to keep individual Roles (to borrow a
loaded term from TripleO UI) viable as templates that can be launched
standalone. The canonical example is ``nova-compute-instance.yaml`` [3]_.

With the migration to provider resources, ``FileInclude`` is not necessary.


**6. Move the templates to Heat-native scaling**

Scaling of resources is currently handled by ``merge.py``. The ``--scale``
command line argument takes a resource name and duplicates it as needed (it's
a bit more complicated than that, but that's beside the point).

Heat has a native scaling ``OS::Heat::ResourceGroup`` [5]_ resource that does
essentially the same thing::

    scaled_compute:
      type: OS::Heat::ResourceGroup
      properties:
        count: 42
        resource_def:
          type: OS::TripleO::Compute
          parameters:
            flavor: baremetal
            image: compute-image-rhel7
            ...

This will create 42 instances of compute hosts.


**7. Replace Merge::Map with scaling groups' inner attributes**

We are using the custom ``Merge::Map`` helper function for getting values out of
scaled-out servers:

* `Building a comma-separated list of RabbitMQ nodes`__

__ https://github.com/openstack/tripleo-heat-templates/blob/a7f2a2c928e9c78a18defb68feb40da8c7eb95d6/overcloud-source.yaml#L642

* `Getting the name of the first controller node`__

__ https://github.com/openstack/tripleo-heat-templates/blob/a7f2a2c928e9c78a18defb68feb40da8c7eb95d6/overcloud-source.yaml#L405

* `List of IP addresses of all controllers`__

__ https://github.com/openstack/tripleo-heat-templates/blob/a7f2a2c928e9c78a18defb68feb40da8c7eb95d6/overcloud-source.yaml#L405

* `Building the /etc/hosts file`__

__ https://github.com/openstack/tripleo-heat-templates/blob/a7f2a2c928e9c78a18defb68feb40da8c7eb95d6/overcloud-source.yaml#L585


The ``ResourceGroup`` resource supports selecting an attribute of an inner
resource as well as getting the same attribute from all resources and returning
them as a list.

Example of getting an IP address of the controller node: ::

    {get_attr: [controller_group, resource.0.networks, ctlplane, 0]}

(`controller_group` is the `ResourceGroup` of our controller nodes, `ctlplane`
is the name of our control plane network)

Example of getting the list of names of all of the controller nodes: ::

    {get_attr: [controller_group, name]}

The more complex uses of ``Merge::Map`` involve formatting the returned data in
some way, for example building a list of ``{ip: ..., name: ...}`` dictionaries
for haproxy or generating the ``/etc/hosts`` file.

Since our ResourceGroups will not be using Nova servers directly, but rather the
custom role types using provider resources and environments, we can put this
data formatting into the role's ``outputs`` section and then use the same
mechanism as above.

Example of building out the haproxy node entries::

    # overcloud.yaml:
    resources:
      controller_group:
        type: OS::Heat::ResourceGroup
        properties:
          count: {get_param: controller_scale}
          resource_def:
            type: OS::TripleO::Controller
            properties:
              ...

      controllerConfig:
        type: OS::Heat::StructuredConfig
        properties:
          ...
          haproxy:
            nodes: {get_attr: [controller_group, haproxy_node_entry]}



    # controller.yaml:
    resources:
      ...
      controller:
        type: OS::Nova::Server
        properties:
          ...

    outputs:
      haproxy_node_entry:
        description: A {ip: ..., name: ...} dictionary for configuring the
          haproxy node
        value:
          ip: {get_attr: [controller, networks, ctlplane, 0]}
          name: {get_attr: [controller, name]}



Alternatives
------------

This proposal is very t-h-t and Heat specific. One alternative is to do nothing
and keep using and evolving ``merge.py``. That was never the intent, and most
members of the core team do not consider this a viable long-term option.


Security Impact
---------------

This proposal does not affect the overall functionality of TripleO in any way.
It just changes the way TripleO Heat templates are stored and written.

If anything, this will move us towards more standard and thus more easily
auditable templates.


Other End User Impact
---------------------

There should be no impact for the users of vanilla TripleO.

More advanced users may want to customise the existing Heat templates or write
their own. That will be made easier when we rely on standard Heat features only.


Performance Impact
------------------

This moves some of the template-assembling burden from ``merge.py`` to Heat. It
will likely also end up producing more resources and nested stacks on the
background.

As far as we're aware, no one has tested these features at the scale we are
inevitably going to hit.

Before we land changes that can affect this (provider config and scaling) we
need to have scale tests in Tempest running TripleO to make sure Heat can cope.

These tests can be modeled after the `large_ops`_ scenario: a Heat template that
creates and destroys a stack of 50 Nova server resources with associated
software configs.

We should have two tests to asses the before and after performance:

1. A single HOT template with 50 copies of the same server resource and software
   config/deployment.
2. A template with a single server and its software config/deploys, an
   environment file with a custom type mapping and an overall template that
   wraps the new type in a ResourceGroup with the count of 50.

.. _large_ops: https://github.com/openstack/tempest/blob/master/tempest/scenario/test_large_ops.py


Other Deployer Impact
---------------------

Deployers can keep using ``merge.py`` and the existing Heat templates as before
-- existing scripts ought not break.

With the new templates, Heat will be called directly and will need the resource
registry (in a Heat environment file). This will mean a change in the deployment
process.



Developer Impact
----------------

This should not affect non-Heat and non-TripleO OpenStack developers.

There will likely be a slight learning curve for the TripleO developers who want
to write and understand our Heat templates. Chances are, we will also encounter
bugs or unforeseen complications while swapping ``merge.py`` for Heat features.

The impact on Heat developers would involve processing the bugs and feature
requests we uncover. This will hopefully not be an avalanche.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Tomas Sedovic <lp: tsedovic> <irc: shadower>


Work Items
----------

1. Remove the custom resource types
2. Remove combining whitelisted resource types
3. Port TripleO Heat templates to HOT
4. Move to Provider resources
5. Remove FileInclude from ``merge.py``
6. Move the templates to Heat-native scaling
7. Replace Merge::Map with scaling groups' inner attributes


Dependencies
============

* The Juno release of Heat
* Being able to kill specific nodes in Heat (for scaling down or because they're
  misbehaving)
  - Relevant Heat blueprint: `autoscaling-parameters`_

.. _autoscaling-parameters: https://blueprints.launchpad.net/heat/+spec/autoscaling-parameters


Testing
=======

All of these changes will be made to the tripleo-heat-templates repository and
should be testable by our CI just as any other t-h-t change.

In addition, we will need to add Tempest scenarios for scale to ensure Heat can
handle the load.


Documentation Impact
====================

We will need to update the `devtest`_, `Deploying TripleO`_ and `Using TripleO`_
documentation and create a guide for writing TripleO templates.

.. _devtest: http://docs.openstack.org/developer/tripleo-incubator/devtest.html
.. _Deploying TripleO: http://docs.openstack.org/developer/tripleo-incubator/deploying.html
.. _Using TripleO: http://docs.openstack.org/developer/tripleo-incubator/userguide.html


References
==========

.. [0] https://github.com/openstack/tripleo-heat-templates
.. [1] http://lists.openstack.org/pipermail/openstack-dev/2014-April/031915.html
.. [2] http://docs.openstack.org/developer/heat/template_guide/hot_guide.html
.. [3] https://github.com/openstack/tripleo-heat-templates/blob/master/nova-compute-instance.yaml
.. [4] http://docs.openstack.org/developer/heat/template_guide/environment.html
.. [5] http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::ResourceGroup
.. [6] http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::RandomString
.. [7] http://lists.openstack.org/pipermail/openstack-dev/2014-July/040115.html
.. [8] https://review.openstack.org/#/c/81666/
.. [9] https://review.openstack.org/#/c/93319/
.. [10] http://docs.openstack.org/developer/heat/template_guide/hot_spec.html#str-replace
