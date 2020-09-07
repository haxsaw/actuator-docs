************************
Wisdom for All Models
************************

This next section dives into some more details of working with Actuator models that apply to all model classes. These
techniques provide ways of building reusable models, establishing conventions, and writing more expressive and
compact models.

============================
More on context expressions
============================

The infra basics section introduced the notion of context expressions, and the namespace basics section did a brief
comparison of context expressions with model references, where it was asserted that context expressions were more
powerful and better suited to modeling tasks. We'll dig into context expressions further here to show more about their
capabilities.

As mentioned previously, models are composed of generic components, whether they are infra resources, namespace roles,
or config and execute tasks. When Actuator processes a component during orchestration, it creates a `context` for that
processing. This context bundles up certain information and makes it available during processing. This information
consists of:

-  The model the component is part of, which can be found in **ctxt.model**
-  The name of the component being processed, which can be found in
   **ctxt.name** (**NOTE**: this is not the value of the ``name`` argument to
   the component, but rather the name of the attribute in the model that
   holds the ref to the component)
-  The component being processed, which can be found in **ctxt.comp**
-  A gateway to the other models in the model set called the nexus, which can be found in **ctxt.nexus**

Two of these items bear more examination, ``ctxt.comp`` and ``ctxt.nexus``.

The component container
-----------------------

As mentioned above ``ctxt.comp`` is a reference to the component being processed. From here, you can add on any attribute
implemented by the component, for instance ``name``. In fact ``ctxt.name`` is just short for ``ctxt.comp.name``. But
more importantly, you might be interested in the component that contains the current component, or the component's
``container``. This can be found with ``ctxt.comp.container``. If this is a top-level component in a model, then
``ctxt.comp.container`` is the model instance itself, or ``ctxt.model``. However, if this is one of the component grouping
constructs that is described in the advanced section on Actuator, then ``ctxt.comp.container`` will be the grouping
construct. You can then reach sibling components within the container.

The nexus
---------

So far, we've only looked at expressions that involve a component and the model the component lives in. But as was
mentioned earlier, a model can be part of a ``model set``, a logically related group of infra, namespace, config, and
execute models. It often is the case that you need to reach a component in a related model, such as a server in an
infra model for the host_ref in a Role in a namespace model. So far, we've always used models references to refer to
components in other models. However, context expressions provide us a way to find all the models in a group via the
``nexus``. Every model has a nexus, but when a set of models are provided to an orchestrator, they are merged and serve
as a bridge from one model to another. Thus, the nexus is a conduit for accessing other models during orchestration,
and serves as an active bridge during this time.

The ``ctxt.nexus`` expression is a short cut for the nexus belonging to the component's model. There are a few standard
attributes on the nexus that allow you to reach other models in the model group:

-  **ctxt.nexus.inf** is the infra model in the group
-  **ctxt.nexus.ns** is the namespace model in the group
-  **ctxt.nexus.cfg** is the config model in the group
-  **ctxt.nexus.exe** is the execute model in the group

Using the nexus to create cross-model references allows models to be more loosely coupled as they don't need to use
a model reference involving a specific model class. To see how this works, consider the following simple infra with two
static servers (a static server is an already-existing server, not a cloud based one that must be provisioned), and a
namespace that uses them:

.. code:: python

    class CtxtInfra(InfraModel):
        server1 = StaticServer('static-1', '192.168.1.1')
        server2 = StaticServer('static-2', '192.168.1.2')

    class CtxtNamespace(NamespaceModel):
        r1 = Role('role1', host_ref=ctxt.nexus.inf.server1)
        r2 = Role('role2', host_ref=ctxt.nexus.inf.server2)

Note the host_ref uses a context expression that navigates through the nexus to the infra model and to the servers.
This namespace can used with this infra, or a different infra that also had servers named server1 and server2. These
get bound together at orchestration time, thus allowing different infra models to be used as needed.

Expressions for multi-components
--------------------------------

Actuator has the ability to create models with variable numbers of components that are only specified before
orchestration time (these are covered in more detail below). Context expressions have support for this kind of
usage by allowing limited subscripting in context expressions.

For example, consider this abbreviated infra and namespace models:

.. code:: python

    class Multi(InfraModel):
        slaves = MultiResource(AWSInstance('slave', ...))
        # we'll leave the details out of this as MultiResources will be covered later

    class NSMulti(NamespaceModel):
        slave_roles = MultiRole(Role('slave',
                                     host_ref=ctxt.nexus.inf.slaves[ctxt.name]))

In the infra model, we declare ``slaves`` to be a MultiResource; every new instance of the resource will be an
AWSInstance; this inner resource is called the `template`. Instances of the template are created whenever they
are referenced, like:

.. code:: python

    infra = Multi('multi-example')
    for i in range(5):
        _ = infra.slaves[i]  # assigning to '_' means the value is throw-away

This results in 5 new AWSInstance resources with names like 'slave-0'..'slave-4'. However, we want to be able to
access these from the Roles in our namespace. We do that by creating a MultiRole, which behaves like the
MultiResource, with one key difference: the host_ref involves a context expression that uses the name of the current
namespace Role component to drive the name of the associated infra resource. So if instead of the above for loop, we wrote:

.. code:: python

    infra = Multi('multi-infra')
    ns = NSMulti('multi-ns')
    ns.set_infra_model(infra)  # joins each model via a nexus
    for i in range(5):
        _ = ns.slave_roles[i]

We'd not only have five Roles in the slave_roles container, we'd also get five AWSInstances in the slaves container
of our infra model. The host_ref context expression ``ctxt.nexus.inf.slaves[ctxt.name]`` is saying:

-  Get the nexus for the current component (ctxt.nexus)
-  Get the infra model in the nexus (ctxt.nexus.inf)
-  Get the slaves in the model (ctxt.nexus.inf.slaves)
-  Get the name of the current component (ctxt.name, which will be 0..4)
-  Get the slave from the infra based on the current component's name (ctxt.nexus.inf.slaves[ctxt.name])

By naming an item in a MultiResource, Actuator will cause that item to be created from the template if it doesn't
already exist.

This is a powerful notion and will be described further below in the specific model sections dealing with
multi-components. The above provides you a way to understand what is going on from a context expression perspective.

===================================
Base classes for boilerplate
===================================

Sometimes, you find yourself writing the same sort of constructs in a model over and over again, which not only is a
waste of time, but can create maintenance headaches if the conventions being coded repeatedly change, necessitating
changes to lots of models. Actuator provides a number of different ways to factor out common elements of models into
reusable components, allowing model authors to only focus on the aspects that are unique to the software system at
hand. These approaches lean heavily on existing Python constructs for creating libraries, and so their use is well
integrated into the language and supporting tools.

This approach is especially good for infra models. Lots of infra models have identical base elements, such as a VPC,
subnet, internet gateway, etc. Such elements can be put into a base model class in a separate module, and then this can
be imported into an app-specific model and used to provide the base componentry, thus requiring the app specific
model to only deal with the infra aspects that are relevant to the app.

To illustrate this, we'll re-write the example model from the :doc:`../infra-basic` page, splitting it into a base
class model and an app specific on. First, the base class, which we'll assume is in a module named awsbase.py:

.. code:: python

    from actuator import ctxt
    from actuator.infra import InfraModel
    from actuator.provisioners.aws.resources import *

    class AWSBaseInfra(InfraModel):
        # this first bit defines the networking resources we need to
        # get to the internet. first, make the virtual private cloud
        vpc = VPC("base-vpc",
                  "192.168.1.0/24")
        # subnet to attach to the VPC
        sn = Subnet("base subnet",
                    "192.168.1.0/24",
                    ctxt.model.vpc)
        # create an internet gateway to the outside world
        igw = InternetGateway("base gw",
                              ctxt.model.vpc)
        # the routing table for VPC and subnet
        rt = RouteTable("base rt",
                        ctxt.model.vpc,
                        ctxt.model.sn)
        # a route to apply to the routing table that allows all traffic out
        r = Route("base route",
                  ctxt.model.rt,
                  dest_cidr_block="0.0.0.0/0",
                  gateway=ctxt.model.igw)
        # make a security group and rules that allow 'pings' and ssh
        base_sg = SecurityGroup("base-sg",
                                "a common sg to build on",
                                ctxt.model.vpc)
        ping_rule = SecurityGroupRule("test rule",
                                      ctxt.model.base_sg,
                                      "ingress",
                                      "0.0.0.0/0",
                                      -1,
                                      -1,
                                      "icmp")
        ssh_rule = SecurityGroupRule("sshrule",
                                     ctxt.model.base_sg,
                                     "ingress",
                                     "0.0.0.0/0",
                                     22,
                                     22,
                                     "tcp")


Our base model contains basic networking pieces (VPC, Subnet, InternetGateway, RouteTable, Route), and puts on basic
security groups and rules that allow pings and SSH.

This model can then be used to create a new model which is a derived class of this one, but only focusing on the
details for that specific model:

.. code:: python

    from awsbase import AWSBaseInfra
    from actuator import ctxt
    from actuator.provisioners.aws.resources import *

    class MyInfraModel(AWSBaseInfra):
        kp = KeyPair("wibble", public_key_file="actuator-dev-key.pub")
        ni = NetworkInterface("server-ni",
                              ctxt.model.sn,
                              description="something pithy",
                              sec_groups=[ctxt.model.base_sg])  # add the security group to the interface
        server = AWSInstance("server",
                             "ami-09393cef16d65b519",  # or whatever image you choose
                             instance_type='t3.nano',
                             key_pair=ctxt.model.kp,   # install the keypair here
                             network_interfaces=[ctxt.model.ni])
        server_fip = PublicIP("server_fip",
                              domain="vpc",
                              network_interface=ctxt.model.ni)

Our new application infra model is now much simpler: it has inherited all of the networking and base security group
resources from the base class, and these can be used in the declaration of new resources specific to the application.
Details for the base class are readily available, with most IDEs providing easy CTRL-click access to the definition
of imported name.

======================
Adding methods
======================

So far, we've used classes solely to provide the ability to declare the different components and relationships in
a system. But don't forget-- our models are Python classes, which means that we can add behaviour to them if we
wish. Generally, the most straighforward use of this is to add convenience functions to implement what would have
otherwise been done in external code. For example, recall these models and for loop from the section above on
context expressions:

.. code:: python

    class Multi(InfraModel):
        slaves = MultiResource(AWSInstance('slave', ...))
        # we'll leave the details out of this as MultiResources will be covered later

    class NSMulti(NamespaceModel):
        slave_roles = MultiRole(Role('slave',
                                     host_ref=ctxt.nexus.inf.slaves[ctxt.name]))

    infra = Multi('multi-infra')
    ns = NSMulti('multi-ns')
    ns.set_infra_model(infra)  # joins each model's nexus

    for i in range(5):
        _ = ns.slave_roles[i]

If we don't want the users of our models to have to worry about such details on how to get the number of slaves
desired, you can instead move this code into a method on the NSMulti model class. The result code could be:

.. code:: python

    class Multi(InfraModel):
        slaves = MultiResource(AWSInstance('slave', ...))
        # we'll leave the details out of this as MultiResources will be covered later

    class NSMulti(NamespaceModel):
        slave_roles = MultiRole(Role('slave',
                                     host_ref=ctxt.nexus.inf.slaves[ctxt.name]))

        def make_slaves(self, num_slaves):
            for i in range(num_slaves):
                _ = self.slave_roles[i]

    infra = Multi('multi-infra')
    ns = NSMulti('multi-ns')
    ns.set_infra_model(infra)  # joins each model's nexus

    ns.make_slaves(5)

You could also provide a way to rapidly set the infra model so that nexus is set up quickly:

.. code:: python

    class Multi(InfraModel):
        slaves = MultiResource(AWSInstance('slave', ...))
        # we'll leave the details out of this as MultiResources will be covered later

    class NSMulti(NamespaceModel):
        slave_roles = MultiRole(Role('slave',
                                     host_ref=ctxt.nexus.inf.slaves[ctxt.name]))

        def __init__(self, *args, infra_model=None, **kwargs):
            super(NSMulti, self).__init__(*args, **kwargs)  # must call super!
            if infra_model != None:
                self.set_infra_model(infra_model)

        def make_slaves(self, num_slaves):
            for i in range(num_slaves):
                _ = self.slave_roles[i]

    infra = Multi('multi-infra')
    ns = NSMulti('multi-ns', infra_model=infra)

    ns.make_slaves(5)

As you can see, we can continue to move implementation details into model classes in order to establish usage
conventions and make things easier for users.
