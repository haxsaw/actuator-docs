********************************
Advanced Infra Modeling
********************************

**Import package**: actuator.infra
    Key names to import: InfraModel, ResourceGroup, MultiResource, MultiResourceGroup, with_infra_options

**Import package**: actuator.modeling
    Key names to import: ctxt

**Import package**: actuator.provisioners.aws.resources
    Key names to import: *

**Import package**: actuator.provisioners.openstack.resources
    Key names to import: *

**Import package**: actuator.provisioners.azure.resources
    Key names to import: *

We've only gone over the basics of infra models so far-- how to declare them, add resources, and make instances. In
this section we'll look at some higher-level modeling constructs that allow for more concise usage, the creation of
reusable components, dynamic resource declarations, and options available when creating models.

======================
Resource Groups
======================

Sometimes you have a set of resources that are always used together. This may be boilerplate for standard
configurations, or groups of resources that you always want to have created in sets. The ``ResourceGroup`` component
allows you to create such groupings, and depending where they are created they may be re-used in various contexts.

As a first example, here is a ResourceGroup outside of an infra model that gathers the various resources needed
on OpenStack to create a network with a path to the external internet:

.. code:: python

    from actuator import ctxt
    from actuator.infra import ResourceGroup
    from actuator.provisioners.openstack.resources import *

    ec = ResourceGroup("route_out",
                       net=Network("ro_net"),
                       subnet=Subnet("ro_subnet",
                                     ctxt.comp.container.net,
                                     u"192.168.23.0/24",
                                     dns_nameservers=[u'8.8.8.8']),
                       router=Router("ro_router"),
                       gateway=RouterGateway("ro_gateway",
                                             ctxt.comp.container.router,
                                             "external"),
                       interface=RouterInterface("ro_inter",
                                                 ctxt.comp.container.router,
                                                 ctxt.comp.container.subnet))

Note the context expressions used in the resources in the group; they are prefixed with ``ctxt.comp.container``. This
provides a reference to the ResourceGroup itself, and is how you create references to sibling resources. If this feels
a bit ponderous, you could assign the prefix to a descriptive name like `sibling` and then use that in the creation
of the group like so:

.. code:: python

    from actuator import ctxt
    from actuator.infra import ResourceGroup
    from actuator.provisioners.openstack.resources import *

    sibling = ctxt.comp.container
    ec = ResourceGroup("route_out",
                       net=Network("ro_net"),
                       subnet=Subnet("ro_subnet",
                                     sibling.net,
                                     u"192.168.23.0/24",
                                     dns_nameservers=[u'8.8.8.8']),
                       router=Router("ro_router"),
                       gateway=RouterGateway("ro_gateway",
                                             sibling.router,
                                             "external"),
                       interface=RouterInterface("ro_inter",
                                                 sibling.router,
                                                 sibling.subnet))

...which you might find a bit more readable. The end result is identical.

.. note::

    ResourceGroup keyword arguments are not pre-defined; they can be anything you want.

.. note::

    Actuator pre-defines a number of single names for common context expression prefixes used in models; these were
    described in :doc:`../infra-basic`.

When the ResourceGroup is created, it automatically 'grows' attributes for the names of each provided keyword argument,
and thus in the above example, ``ec.net`` is a Network resource, ``ec.subnet`` a Subnet, and so forth. Suppose now that
this definition was in a module named `boilerplate.py`; it could then be used in an infra model as follows:

.. code:: python

    from boilerplate import ec
    from actuator import ctx
    from actuator.infra import InfraModel
    from actuator.provisioners.openstack.resources import *

    class RGInfra(InfraModel):
        external = ec

The ResourceGroup's resources can subsequently used in the model via context expressions such as
``ctxt.model.external.net`` or ``ctxt.model.external.router``.

Factoring out boilerplate is one key use of resource groups; another is to group together resources that should be
provisioned together in dynamic contexts. We'll see this usage a bit later in this section.

====================================
MultiResource and MultiResourceGroup
====================================

MultiResource components provide a way to model a resource that can create multiple copies of a template resource simply by
supplying the MultiResource a unique key for the new copy. This is how Actuator allows the modeling of systems
with a variable number of components from instantiation to instantiation.

To create a MultiResource, you must supply it a template resource, and assign the MultiResource to an attribute in
the model. Then when you have an instance of the model, you can treat the MultiResource attribute like a Python dict
and supply a key to it; this causes the MultiResource to create a new instance of the template with a name that is
a combination of the key and the template resource's name.

Let's look at a useless example before we look at a more useful but complex one. This example will create as many AWS
VPC instances as we give keys to the MultiResource:

.. code:: python

    from actuator.infra import InfraModel, MultiResource
    from actuator import ctxt
    from actuator.provisioners.openstack.resources import VPC

    class MRInfra(InfraModel):
        vpcs = MultiResource(VPC("vpc", "192.168.1.0/24"))

    infra = MRInfra("MR-example")
    infra.vpcs[0]  # this creates a new VPC named 'vpc_0'
    infra.vpcs[1]  # this creates a new VPC named 'vpc_1'
    infra.vpcs['London']  # this creates a new VPC named 'vpc_London'

The act of supplying a unique key to a MultiResource object creates a new instance of the template resource. Any
resource can be a template, include another MultiResource or a ResourceGroup. In fact, this latter usage is so common
a combination class is provided, the ``MultiResourceGroup``, which let's a group of resources be created in one go when
a new key is provided.

Here is a more complex example for AWS. It defines a set of 'slave' servers that can be used for some computation
(we're leaving out the KeyPairs and SecurityGroups to focus on the details :

.. code:: python

    from actuator import ctxt
    from actuator.infra import InfraModel, MultiResourceGroup
    from actuator.provisioners.aws.resources import *

    class ComplexMRInfra(InfraModel):
        # assume some standard network boilerplate that creates:
        # a Subnet 'sn',
        # a SecurityGroup with rules 'sg'
        # and a KeyPair 'kp'
        slaves = MultiResourceGroup("slaves",
                                    ni=NetworkInterface("slave-ni",
                                                        ctxt.model.sn,
                                                        description="something pithy",
                                                        sec_groups=[ctxt.model.sg]),
                                    slave=AWSInstance("slave",
                                                      "ami-09393cef16d65b519",
                                                      instance_type='t3.nano',
                                                      key_pair=ctxt.model.kp,
                                                      network_interfaces=[ctxt.comp.container.ni])
                                    )

Notice how the resources in the MultiResourceGroup refer to each other with context expressions prefixed with
``ctxt.comp.container`` (as noted above in the discussion of create a 'sibling' context expression).
This expression allows you to access sibling resources in the same container. They can also
make reference to resources outside of model, such as the 'sg' SecurtyGroup.

Usage is identical to the first example:

.. code:: python

    infra = ComplexMRInfra('complex-mr')
    infra.slaves[0]  # creates both a NetworkInterface and AWSInstance resource at 'slaves_0'
    infra.slaves['London']  # creates a NetworkInterface and AWSIstance resource at 'slaves_London'

You can nest these arbitrarily, creating MultiResources that contain MultiResourceGroups, and so forth. Each level
of nesting will require a key to create a new instance of the template at each nested level.

The use of MultiResource allows you to create more flexible infra models that can be easily scaled as desired when
creating an instance of a system.

===================
Options
===================

You can invoke an option setting function within an infra model that tunes the behaviour of how the model is processed.
The ``with_infra_options()`` function recognised the following options:

-   **long_names**: boolean, defaults to False. Normally, the name used for component as recorded on the cloud system
    is just the name given the component in the model. However, there there are lots of instances of a system in a
    cloud, it can be confusing as to which resource belongs to which system instance. When `long_names` is True,
    then the cloud is informed that the name for the resource is the name of the model, followed by the names of
    all resources that lead to the resource, and ending with the resource name. This yields a suitable unique name.

Using ``with_infra_options()`` is straightforward; you just call it somewhere inside your infra model:

.. code:: python

    from actuator.infra import InfraModel, with_infra_options
    class OptionsExample(InfraModel):
        with_infra_options(long_names=True)
        # then the rest of the model's content.

While ``with_infra_options()`` can be called anywhere in the model, it is generally most useful put it at the top.
