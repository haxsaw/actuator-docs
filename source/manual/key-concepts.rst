**************
Key Concepts
**************

There are a handful of key concepts upon which Actuator depends, and which will be referred to repeatedly in the
pages that follow. These concepts are:

======
Model
======

As discussed in the :doc:`overview`, models are a representation of one of the infrastructure, namespace, configuration,
or execution aspects of an application system. Models can be used in a variety of ways, but the central use within
Actuator is to give them to an orchestrator to create an instance of the system being modelled.

Actuator uses the Python class construct as the means to create models, as there's a good analog between how classes and their
instances are used in Python as how models and their instances are used in Actuator. In both, the class is an entity
that is used to create a description of the data and behaviour for a real-world object, and an instance of that class
is used to represent an occurrence of the entity being described.

=========
Component
=========

A model uses some kind of *component* to represent the different aspects of the model that are to be described. There
are different kinds of components that are used with the different kinds of models. For example, an infrastructure model
uses *resource* components, a namespace uses *role* components, and configuration and execution models use *task*
components. Additionally, there are structuring components available that allow models to put components into reusable
groups, or define ways to generate multiple instances of a particular component in an instance of a model.

===============
Model Reference
===============

When models are used together, sometimes components must be able to refer to another component, or a data attribute of
another component, in another model. For example, a role component in a namespace model needs to know the server where the
role is going to operate from in the infra model. However, the actual server component won't be known until the
orchestrator performs the work needed to create the server. To allow Actuator to provide this information at the last
minute, Actuator defines a construct known as a *model reference*; this is a logical reference to information in a
model that will be made available when it comes into existence. Model references can be to components, to data
attributes of components, or to methods of a component that have a particular signature. Actuator uses these references
to determine which components depend on each other, which tells Actuator in what order work must be peformed.

====================
Context Expression
====================

Model references have two limitations: they must be made against a specific model class, and they always refer to the
same logical place in a model. However, circumstances sometimes dictate that a reference needs to be able to vary based
on the component holding the reference, or that the model may not be known when a component is defined. This situations
are handled by context expressions. A *context expression* is a special kind of reference involving the magic *ctxt*
object that provides a way to create a logical path to a component, data attribute, or method, that will result in a
model reference depending on where the context expression is evaluated. For example, the same context expression can
be used to indicate where to pick up an IP address from an infra model without naming a specific infra model in the
expression. This allows the context expression to be used with different infra models. Context expressions are the
only way to create intra-model references between components of the same model, and are also used to express how to
match multiple roles to corresponding infra components. This concept will be given more attention in later sections.

============
Orchestrator
============

The *orchestrator* is an Actuator object that takes one or more models, one or more proxies, and some other optional
data, and can then determine what work is needed to create an instance of the system that the models describe. It
does so by first processing the infra model, then the config model, and finally the exec model in turn. It can also
process sets of models grouped into services, which are described in the Advanced Use section of this documentation. The
orchestrator can report back details of errors encountered, and also tear down previously instantiated systems.

======
Proxy
======

A *proxy* is an Actuator object that stands in for a cloud provisioning (and associated) API. To the end user, it
provides a means to bundle up user credentials and indications of what cloud and/or region to contact, while internally
it provides a set of properly conditioned API endpoints through which the various cloud resources can be queried,
provisioned, and de-commissioned. Multiple proxies for the same cloud but different regions can be provided to an
orchestrator, and resources that indicate they are to be provisioned in the different regions can be defined such
that the orchestrator will ensure that