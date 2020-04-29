************************
Infra Model Basics
************************

Although the namespace model is the one that is central in an
Actuator model set, it actually helps to start with the infra model as building an infra model first can yield
immediate benefits. The infra model describes all the dynamically
provisionable and static infra resources for a system and describes how they relate to each
other. The model can define groups of resources and resources that can
be repeated an arbitrary number of times, allowing them to be nested in
very complex configurations.

=======================
A Basic AWS Infra Model
=======================

In Actuator, an infra model is created by defining a subclass of actuator.infra.InfraModel and declaring its
components.

The following shows the basic pieces needed to create a single-server infra model on EC2 using a virtual private
cloud on AWS. We're going to start by ignoring providing a way to access the instance just to make the base code
simpler:

.. code:: python

    from actuator import ctxt
    from actuator.infra import InfraModel
    from actuator.provisioners.aws.resources import *

    class MyInfraModel(InfraModel):
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

        # the following gets us our specific server, network interface, and
        # gives the interface a floating public IP we can reach
        ni = NetworkInterface("server-ni",
                              ctxt.model.sn,
                              description="something pithy")
        server = AWSInstance("server",
                             "ami-09393cef16d65b519",  # or whatever image you choose
                             instance_type='t3.nano',
                             network_interfaces=[ctxt.model.ni])
        server_fip = PublicIP("server_fip",
                              domain="vpc",
                              network_interface=ctxt.model.ni)

Like with many clouds, there's a lot of IaaS boilerplate that needs to be added to a model to be able to reach a
server from the public internet. Actuator provides tools to hide these details if you wish, and those are covered
in the Infra Models section under More Advanced Uses. But for now, we'll consider the details.

The basic networking setup involves the creation of VPC, Subnet, InternetGateway, RouteTable, and Route resources
in the model. The order of creation of these resources isn't important, as Actuator will sort out what depends on
what. The important aspect to notice is that in creating some of the resources, there are arguments that start
with `ctxt.`. These are `context expressions,` and are used to express the relationships between resources in a
number of different circumstances, in this case in an unfinished model. This is an important enough topic to
warrant a short digression on context expressions in some detail.

==========================
Context Expressions Primer
==========================

In Actuator, an expression that begins `ctxt.` signals the start of a `context expression`. Context expressions are
very powerful tools for creating linkages between components in a loosely-coupled way. They essentially describe a
navigation path though a set of objects such as resources, tasks, or roles, and when actually evaluated they have
some knowledge of the location or `context` in which they are evaluated.

Every time a component in any model is processed by Actuator (be it a resource or some other
component), a processing context is created. The *context* wraps up:

-  the instance of the model the component is part of,
-  the component itself,
-  the name of the component,
-  the other models that this model is related to,
-  and other advanced information.

The context expression is bound to this context when a component is processed, and it provides a way to access other
parts of a model or a related model in a decoupled way. This is because the context expression is only evaluated during
component processing, and the context it's bound to allows this evaluation to yield a specific value.

For example, in the above infra model, the creation of the Subnet resource involves naming the VPC that the
subnet should be attached to, which is indicated with the context expression `ctxt.model.vpc`. When the Subnet
resource is processed by Actuator during orchestration, the context is set up as follows:

- `ctxt.model` is the instance of MyInfraModel being processed
- `ctxt.comp` is the Subnet resource (component) itself
- `ctxt.name` is the name of the Subnet resource, in this case "base subnet"
- `ctxt.nexus` is the gateway to the other models in the model group (namespace, config, etc)

So when the Subnet resource is processed, the context expression `ctxt.model.vpc` is evaluated, and this results in
accessing the model instance's vpc attribute in that instance, which is requried in order to create an AWS Subnet.

Context expressions can be assigned to variables and then further used to create new context expressions. This is handy
if a construction begins to get a bit long and unwieldy. For example you could put in your code:

.. code:: python

    cmod = ctxt.model
    # and then later refer to the VPC in the above model as:
    cmod.vpc

Thus giving you a way to abbreviate a longer context expression.

.. note::
    In the actuator.modeling module, where ctxt is defined, Actuator also defines a short list of standard
    'shortcuts' for common context expression prefixes:

    -  *ccomp* is the same as ctxt.comp
    -  *cparent* and *cpar* are the same as ctxt.parent
    -  *cmodel* and *cmod* are the same as ctxt.model
    -  *cnexus* and *cnex* are the same as ctxt.nexus

.. note::
    Whenever a model instance is created, new instances of all the components are created as well. Otherwise different
    model instances would share the same component, resulting in colliding uses of the component.

With a context expression, you can navigate around a model, access its components, and even provide access to specific
component values. Context expressions have a lot of other capabilities, which will be introduced later in the doc.

=====================
Building on the Model
=====================

So the the big problem with the above example is that it doesn't provide any way to actually connect to the server.
Although it has a public IP, no keys have been installed, and there are no security groups set up to allow remote
access from anywhere. Let's update the example with further declarations that cover these aspects.

.. code:: python

    from actuator import ctxt
    from actuator.infra import InfraModel
    from actuator.provisioners.aws.resources import *

    class MyInfraModel(InfraModel):
        # make the virtual private cloud
        vpc = VPC("base-vpc",
                  "192.168.1.0/24")
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
        # make the other network resources
        sn = Subnet("base subnet",
                    "192.168.1.0/24",
                    ctxt.model.vpc)
        igw = InternetGateway("base gw",
                              ctxt.model.vpc)
        rt = RouteTable("base rt",
                        ctxt.model.vpc,
                        ctxt.model.sn)
        r = Route("base route",
                  ctxt.model.rt,
                  dest_cidr_block="0.0.0.0/0",
                  gateway=ctxt.model.igw)

        # the above was largely boilerplate for networking; the following gets
        # us our specific server and gives it a floating public IP where we
        # can reach it. first, make a keypair that can be installed on the
        # server instance
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

The above version of the model added KeyPair (kp), SecurityGroup (base_sg) and SecurityGroupRule (ssh_rule,
ping_rule) components to the model.

.. note::
    The public part of the key-pair specified in the KeyPair component's creation was made with ssh-keygen. Actuator
    comes with keypair you can use for experimentation, actuator-dev-key(.pub), but your own keypair should be
    generated for development and production uses. Keypairs should not have a password to use; Actuator currently
    doesn't support password protected keys.

.. note::
    On Linux, Actuator currently requires ssh access via either password or keypair in order for config models to
    put content on hosts. So a security group rule that allows ssh from a known location is required.

=============
Trying it out
=============

Using the orchestrator we built in :doc:`orch-basic`, we can create an instance of the server in AWS. If you put the
above model code into a Python module my_models.py, you can then run the code and see you instance get built on AWS,
and also have Actuator tear the instance down.

We'll use this final example infra model when having a look at the other models.
