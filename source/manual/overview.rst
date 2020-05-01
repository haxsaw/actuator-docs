********
Overview
********

=====
Intro
=====

Actuator is focused on providing a toolset that supports the creation of Python-based models of application systems
that not only can be used to start up instances of the modelled system, but to open up the model itself for other
enterprise processes. Actuator's perspective is that a model of a system has value beyond the bread-and-butter task
of starting up a system, and so is structured to provide easy access to the information in the model, and to make it
easy to integrate models with other systems in order to help automate the coordination of model data used in other
enterprise processes.

Actuator provides an end-to-end set of tools for spinning up
systems in the cloud, from provisioning the infra, defining the logical roles
in the system and the names that govern their operation,
configuring the infra for the software that is to be run, and then
executing that system's code on the configured infra.

It does this by providing facilities that allow a system to be described
as a collection of *models* in a declarative fashion directly in Python
code, in a manner similar to various declarative systems for ORMs
(SQLAlchemy *declarative* being a prime example). Being in Python, these models:

-  can be very flexible and dynamic in their composition
-  can be integrated with other Python packages
-  can be authored and browsed in existing IDEs
-  can be debugged with standard tools
-  can be inspected for auditing and other purposes
-  and can be factored into collections of reusable sets of
   declarative components

And while each model provides capabilties on their own, they can be
inter-related to not only exchange information, but to allow instances
of a model to tailor the content of other models.

Actuator uses the Python *class* construct as the basis for defining a model, and
the class's contents serves as a logical description of the item being modeled; for
instance a collection of infrastructure resources for a system. These
model classes can have both static and dynamic aspects, allowing them to be used to describe systems
that include fixed assets as well as dynamic cloud-based ones..

================
The Four Models
================

Actuator defines four models for fully describing a system, although it is possible to use Actuator with fewer models
to handle various tasks. This further enhances Actuator's flexibility in how it can be used and the tasks it can
address. The models defined by Actuator are:

Infrastructure Model
--------------------

The infrastructure model, or *infra model*,
defines all the infrastructure resources of a system and their
inter-relationships. Infra models can have fixed resources that are
always provisioned with each instance of the model class, as well as
variable-sized resource containers that allow multiple copies of
resources to be easily created on an instance-by-instance basis. The
infra model also has facilities to define groups of resources that can
be created as a whole, and an arbitrary number of copies of these groups
can be created for each instance. References into the infra model can be
held by other models, and these references can be subsequently evaluated
against an instance of the infra model to extract data from that
particular instance. For example, a namespace model may need the IP
address from a particular server in an infra model, and so the namespace
model may hold a reference into the infra model for the IP address
attribute that yields the actual IP address of a provisioned server when
an instance of that infra model is provisioned.

Namespace Model
---------------

The *namespace model* defines a hierarchical namespace based around system
*roles* which defines all the names that are important to the
configuration and run-time operation of a system. A role can be
thought of as a software component of a system; for instance, a system
might have a database role, an app server role, or a grid node role.
Names in the namespace can be used for a variety of purposes, such as
setting up environment variables, or establishing name-value pairs for
processing template files such as scripts or properties files. The names
in the namespace are associated with the namespace's roles, and each
system role's view of the namespace is composed of any names specific to
that role, plus the names that are defined higher up in the namespace
hierarchy. Values for the names can be baked into the model, supplied at
model class instantiation, by setting values on the model class
instance, or can be acquired by resolving references to other models
such as the infra model.

Configuration Model
-------------------

The configuration model, or *config model*, defines all the tasks to perform on the system roles'
infrastructure that make them ready to run the system's executables. The
config model defines tasks to be performed on the logical system
roles of the namespace model, which in turn indicates what
infrastructure is involved in the configuration tasks. The configuration
model also captures task dependencies so that the configuration tasks
are all performed in the proper order. Configuration models can also be
treated as tasks and used within other configuration models.

Execution Model
~~~~~~~~~~~~~~~

The execution model, or *exec model*, defines the actual processes to run for each system
role named in the namespace model. Like with the configuration model,
dependencies between the executables can be expressed so that a
particular startup order can be enforced.

.. note::
    A logically related grouping of an infra, namespace, config, and execute models is called a **model set**.

Actuator then provides a number of support tools that can take instances
of these models and processes their information, turning it into
actions in the cloud. So for instance, an orchestrator can take an infra
model instance and manage the process of provisioning the infra it
describes, and another can marry that instance with a namespace to fully
populate a namespace model instance so that the configurator can carry
out configuration tasks, and so on.

=======================
Basic Operational Flow
=======================

Using Actuator generally involves the following steps:

1. *Create the required models*. The infrastructure model can be used alone with no other models, but to
   use a configuration or execution model, you must also have a namespace model. Which you need depends on what you're
   trying to accomplish.
2. *Create the appropriate proxy*. The provisioner proxy object performs two tasks: it holds the credentials used to
   communicate with particular cloud provider, and it also provides various factory methods to yield API objects that
   facilitate communication with the cloud's API.
3. *Create instances of the models*. These are just Python classes, and so you create instances of them in the normal
   manner you would for instance creation.
4. *Create an orchestrator*, providing it the model instances and the proxy.
5. *Instruct the orchestrator to initiate the system instance* described by the models.

Step 1 is only done once; steps 2-5 are often done in code in order to make an instance of the system being described
by the models.
