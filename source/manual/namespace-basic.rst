************************
Namespace Model Basics
************************

**Import package**: actuator.namespace
    Key names to import: NamespaceModel, Var, with_variables, with_roles, Role, RoleGroup, MultiRole, MultiRoleGroup

**Import package**: actuator.modeling
    Key names to import: ctxt

The namespace model defines the logical structure of a system and provides the means for joining the other Actuator
models together. It does this by declaring the logical Roles of a system, relating these Roles to the infrastructure
elements where the Roles are to execute, and providing the means to identify what configuration tasks are to be
carried out for each Role as well as what executables are involved with making the Role function.

A namespace model has four aspects. It provides the means to:

1. Define the logical Roles of a system.
2. Define the relationships between logical Roles and hosts in the infra model where the Roles are to execute.
3. Arrange the Roles in a meaningful hierarchy.
4. Establish names (Vars) within the hierarchy whose values will impact configuration activities and the operation of the
   Roles.

Together, Vars and Roles provide considerable representative power in describing namespaces, but before we dig into
those details, let's look at the basics of creating a namespace model.

==================
A Simple Namespace
==================

Similar to infra models, `namespace models` are created by defining a subclass of ``actuator.namespace.NamespaceModel``
and
filling it with ``Role`` and ``Var`` components. `Roles` are associated to an IP addressable resource, typically defined
in the infra model
(but can actually just be a hard-coded IP address), which indicates where the Role is to be configured and run
`Vars` are name-value pairs that can be established at the model level, or attached to a specific Role.

Here's a simple example that demonstrates the basic features of a namespace. It will model two Roles, an app server
and a computation engine, and use the MyInfraModel infra model from above for certain values:

.. code:: python

    from actuator.modeling import ctxt
    from actuator.namespace import Var, NamespaceModel, Role, with_variables
    # assume that MyInfraModel is already known in this module

    class MyNamespaceModel(NamespaceModel):
        with_variables(Var("COMP_SERVER_PORT", '8081',
                           in_env=False),
                       Var("APP_HOME_DIR", None),
                       Var("COMP_SERVER_HOST",
                           MyInfraModel.server.get_ip,
                           in_env=False),
                       Var("EXTERNAL_APP_SERVER_IP",
                           MyInfraModel.server_fip.get_ip,
                           in_env=False),
                       Var("APP_SERVER_PORT", '8080',
                           in_env=False),
                       Var("ROLE_NAME", ctxt.comp.name),
                       Var("ROLE_HOME", "!{APP_HOME_DIR}/!{ROLE_NAME}"))

        compute_server = Role("compute_server",
                              host_ref=MyInfraModel.server)

        app_server = Role("app_server",
                          host_ref=MyInfraModel.server,
                          variables=(Var("ROLE_NAME", "MyApp"),
                                     Var("PRIVATE", "mind yer bidnizz")))

Though this is fairly simple, there is actually a lot going on here, so we'll have a look at this a piece at a time.

====
Vars
====

As mentioned before, `Vars` are used to create symobolic names with associated values that can be attached to Roles
or namespace models, and are used in the processing of config and exec models.

In this example, the ``with_variables(...)`` call is used to declare Vars that are to apply to the model globally.
Any Vars that are
defined with ``with_variables(...)`` are visible to all Roles in the model. However, Roles may also define Vars that
are only visible to the Role, and if a Role's Var has the same name (first argument to Var()) as a more global Var,
the Role's Var definition takes precedence.

There are several different Vars being created that illustrate different uses of Vars. A Var is created with the
``Var`` class, the first argument being the name of the Var, the second being the value. First, consider this Var:

.. code:: python

    Var("COMP_SERVER_PORT", '8081', in_env=False)

...which is setting the Var with the name COMP_SERVER_PORT to '8081'. *This shows that even if a Var value will be used
as a number, it must be defined with a string.* It also has the final keyword argument ``in_env=False``, which will be
used by config and exec models to determine if a Var should be part of a task's environment or not. This is
saying that this variable should not become an environment variable for any task (the default is to include
a Var in any task environment).

Next the Var:

.. code:: python

    Var("APP_HOME_DIR", None),

...defines the var APP_HOME_DIR but doesn't provide it a value. This is how to create a Var whose value is to be
provided when the model is instantiated and orchestrated. This Var tells us that we don't know the APP_HOME_DIR up
front-- it will be provided to us when we want to make an instance of this model.

Another different Var is:

.. code:: python

    Var("COMP_SERVER_HOST", MyInfraModel.server.get_ip, in_env=False)

This Var is defining the COMP_SERVER_HOST Var, but the value isn't a string-- in Actuator, this is called a
`model reference`. In this case, the reference is to a method on the server resource in the MyInfraModel model that
will return the string IP address of the server. Model references are similar to context expressions, but aren't
quite as powerful. We'll go over the basics of model references below, but the important thing to note here is that
when this Var's value is requested, this method will be called and the resultant string value will be returned as
the value.

We later have this Var:

.. code:: python

    Var("ROLE_NAME", ctxt.comp.name)

...which involves a context expression as the value. This context expression is saying that when a component is
processed, either a Role or the model itself, return the component's name. So depending on which component asks the
question, this expression will yield a different value. For the namespace model itself it will be the name of the
namespace model instance. For one of the Roles, it will be the Role's name (the first argument to ``Role()``).

Finally, we have this Var:

.. code:: python

    Var("ROLE_HOME", "!{APP_HOME_DIR}/!{ROLE_NAME}")

...which defines the Var ROLE_HOME as a string that uses `replacement parameters`. A replacement parameter is a Var
name enclosed within the delimiter ``!{ }``, which tells Actuator to replace the entire string with the value of the
Var named inside the delimeter. In this case, ROLE_HOME will be the concatenation of the value of APP_HOME_DIR (when
it is specified) with that of ROLE_NAME (which will be the value of the Role or namespace model instance evaluating
the Var). This provides a mechanism to create customised Var values based on other Vars, model reference values, and
context expressions.

=====
Roles
=====

As previously mentioned, a `Role` is a logical component of a system, one that generally is associated with something
that can be reached at an IP address. A Role can name a single such resource, or can be the name for a group of
identical resources.

Let's consider the Roles that are declared in the example above. First we have:

.. code:: python

    compute_server = Role("my_compute_server",
                          host_ref=MyInfraModel.server)

...which defines the ``compute_server`` Role in the model (which has the name 'my_compute_server'). Also note the
keyword argument ``host_ref``, which is set to ``MyInfraModel.server``. This tells Actuator that this Role will be
realised on the 'server' resource in MyInfraModel. The value of host_ref may also be a string with an FQDN or
IP address of the host for the Role, or it may also be a model reference to a so-called StaticServer resource
in an infra model class (more on those later).

Next, we declare the app_server Role:

.. code:: python

  app_server = Role("app_server", host_ref=MyInfraModel.server,
                    variables=(Var("ROLE_NAME", "MyApp"),
                               Var("PRIVATE", "mind yer bidnizz", in_env=False)))


Besides a name and host_ref, this Role defines its own Vars. One, "ROLE_NAME", has the same name as a Var defined
at the global model level with ``with_variables()``, and thus overrides the value of that Var with its own hard-coded
value of "MyApp". It also adds a new Var, "PRIVATE", which only the app_server component can access. This is because
any Role's set of visible Vars is the union of the ones more global to it plus the ones that are defined on the
Role directly. In this way, individual Roles can define Vars to which only the Role will have visibility, and Vars
that should impact the entire namespace can be defined globally in the namespace.


======================
Model Reference Primer
======================

We've seen a few instances of model references and context expressions as we look at namespaces and have mentioned
that they cover some of the same territory. Since these entities have different characteristics and
capabilities, we'll take some time here to discuss model references in some detail, and then do some comparisons
with context expressions.

In Actuator, a `model reference` is an object that references a specific component (or a component's attribute)
within an Actuator model. The component being referenced does not have to have significant data in when the reference
is created;
model references provide a way to 'point to' or 'refer to' where a data item will appear at some later time so that
another
component can fetch it when needed. References also contain information as to where the data item is coming from,
which allows Actuator to determine what components depend on other components.

Model references are automatically generated when you attempt to access an Actuator component through a model class or
an instance of a model class. These references can then be used as arguments to other components, and when an
orchestrator is asked to begin its work, it reviews all the model references and builds a dependency graph that
reflects what components require values from other components. Hence, model references provide the means for
Actuator to construct an execution plan for use in orchestrating the stand-up of a model.

As an example, consider this fragment of the MyInfraModel infrastructure model; for this example, we add a class
attribute that is a datetime object for the time of the import called ``now``:

.. code:: python

    from datetime import datetime
    from actuator import ctxt
    from actuator.infra import InfraModel
    from actuator.provisioners.aws.resources import *


    class MyInfraModel(InfraModel):
        now = datetime.utcnow()

        attr = 47

        vpc = VPC("base-vpc",
                  "192.168.1.0/24")

        sn = Subnet("base subnet",
                    "192.168.1.0/24",
                    ctxt.model.vpc)

        def some_method(self):
            print("hello from some_method!")

If we load this into a Python interpreter and start poking around in the class, we find the following:

.. code:: python

    >>> MyInfraModel.now
    datetime.datetime(2020, 4, 29, 18, 38, 52, 488400)
    >>> MyInfraModel.attr
    47
    >>> MyInfraModel.vpc
    <actuator.modeling.ModelReference object at 0x000001C7B746A188>
    >>> MyInfraModel.sn
    <actuator.modeling.ModelReference object at 0x000001C7B746A108>
    >>>

When we have a look at non-Actuator attributes in a model class, we see the kind of results we expect. But notice
when we have a look at the ``vpc`` or ``sn`` attributes, we get ModelReference objects. These can be used as arguments
to other components, as we saw above with host_ref arguments to a Role. They can also be accessed by user-created
code that accesses the model to pull out interesting information, by using the value() method on the reference
as shown below:

.. code:: python

    >>> infra = MyInfraModel("trial")
    >>> infra.vpc
    <actuator.modeling.ModelInstanceReference object at 0x000001E022412988>
    >>> infra.vpc.value()
    <actuator.provisioners.aws.resources.VPC object at 0x000001E022409E48>
    >>>

The observant reader may have noticed that ``MyInfraModel.vpc`` results in a ModelReference, but that ``infra.vpc``
results in a ModelInstanceReference when we access ``infra.vpc``. These two are slightly different in capability and are
used in different circumstances:

-  **ModelReferences** are generated when you access Actuator components through a **model** and are used for modeling
   purposes.
-  **ModelInstanceReferences** are generated when you access Actuator components through a **model instance** and are
   not used in modeling, but for orchestration and acquiring data from models.

ModelInstanceReferences can be generated from a corresponding ModelReference via an instance of a model, but this is all
rather advanced usage and will be covered in more detail in the advanced sections of the doc.

The ``key``
-----------

Model references have a useful property named ``key``. The value of this attribute is the value of the attribute or key
that leads to a modeled object. For example, if the model reference is something like ``MyModel.wibble``, then
``MyModel.wibble.key`` will be ``wibble``. If you used one of the multi-component structuring components (covered in
the advanced section), such as ``MyModel.grids[2]``, then ``MyModel.grids[2].key`` will be ``2``.

This may not seem
terribly useful when creating references, but when using context expressions you don't always know the value of the
``key`` that led you to a particular item, and so it can be very handy to be able to look this up on the fly. We'll
see more sophisticated uses of ``key`` in the advanced sections of this manual.

=======================================
Model References vs Context Expressions
=======================================

So you can see that model references and context expressions are both useful in Actuator. However, they both have
slightly different purposes and domains where they provide more value, though either can be used in some contexts.
The following will summarise these differences and hopefully provide you a guide to selecting the approach that is
right for your use.

Modeling intra-model relationships
----------------------------------

This is when a component in a model needs to refer to another component of the same model, such as shown above when
the Subnet resource needed to refer to the VPC resource. It isn't possible to create a model reference for these
kinds of relationships; because it requires a model class to create a model reference, you can't create a model
reference involving a model class that you are in the midst of defining. So the **only** way to make relationships
between components of the same model class is to use context expressions.

Modeling inter-model relationships
----------------------------------

This one is a bit more subtle. Using model references to identify relationships between components of different models,
for instance to identify the host_ref for a Role in an infrastructure model, has the benefit of clarity; you name
the actual infra model class, and refer to the desired modeling attribute directly. Such references not only read
better than the equivalent context expression, but you can often rely on the completion help from your IDE when
authoring the model in the first place.

However, it isn't as cut and dried as that. Context expressions are representationally more powerful, and can model
more complex relationships than can be with model references (some of these will be covered in the advanced usage section
later in the docs). Context expressions are also *context aware*, and model references aren't, again allowing
context expressions to yield results that model references can't. So some kinds of inter-model relationships demand
the use of context expressions.

Additionally, context expressions couple models more loosely than model references. For example, if a namespace
model only uses context expressions to model Role host_ref relationships to an infra model, you can actually swap
out a model that uses resources for AWS with one that uses Azure resources, as long as the names in both infra models
match and map to semantically equivalent resources (a server in one is a server in another). Model references tie
you to using a specific model class. If your application requires being able to swap out one model class for another,
then you'll want to use context expressions for inter-model relationship modeling.

Extracting data from models
---------------------------

No contest here: model instance references are the **only way** to fetch data that is contained in a model instance in
user applications that use Actuator models. Context expressions simply have no capacity to return data from a model--
they are solely designed to express relationships. Internally, Actuator turns context expressions into model references
so that the data being referred to can be acquired. And in fact, user manipulation of model classes automatically
yields model references on which ``value()`` is invoked to acquire the underlying data behind the reference. So for
activities beyond modeling that involve using Actuator model data, expect to be working with model references.

A good rule of thumb to follow is: use context expressions consistently for modeling unless you have reason not to,
and expect to work with model references for everything else.
