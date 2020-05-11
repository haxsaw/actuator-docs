**************************
Advanced Config Modeling
**************************

Config models have a few more capabilities than infra and namespace models; this is due to:

#.  Config models must be told explicitly about dependencies between model components.
#.  Config models actually `query` namespace models for where to perform tasks.
#.  Config models often need to carry out similar steps on multiple servers even if they play different roles in a
    system.
#.  The system you want to affect and from where you produce the effect aren't necessarily the same.

So to dig into more advanced config modeling, we need to establish a few concepts first.

=====================
task_role vs run_from
=====================

Each task in a config model has two keyword arguments available, ``task_role`` and ``run_from``, both of which are a
reference to a namespace Role (which can be either a model reference or context expression). The differences are
important:

-   ``task_role`` identifies what Role a task is aimed at impacting. This means that from the perspective of the Vars
    that are made available for replacement, they are the ones available from the ``task_role``. Normally, this is also
    where the task is to be executed. There are a few ways to determine the ``task_role``, the most common is to use
    the ``task_role=`` keyword argument. Other ways will be described below.
-   ``run_from``, an optional keyword argument, also identifies a Role, but in this case this is the role of where
    the task to to be *executed*. The ``run_from`` Role doesn't impact any of the Var values that may be used when
    running a task; those are still defined by the ``task_role``. The ``run_from`` Role simply defines a different
    location from where to run the task. ``run_from`` Roles imply that Actuator will log into the ``run_from`` Role's
    host to perform the task, but with the Vars from the ``task_role``.

This arrangement provides certain flexibility in how to get work done that is pertinent to a specific Role. For
instance, you can use a bastion as your ``run_from``, and thus only have limited contact with your infra, and through
this host perform the rest of your configuration tasks for all hosts beyond the bastion.

===================
Vars as environment
===================

Namespace Vars perform two functions in config models:

#.  They provide values for replacement expressions in template files and command strings.
#.  They optionally can provide environment values for tasks that are run on the provisioned infra.

So in essence, the set of Vars that are visible to a Role in a namespace can be used to populate the environment for
any task that is run on a Role's host. Vars have an optional keyword argument, ``in_env`` (which defaults to True),
that can be set to False to keep a Var from being put into a Task's environment. This is handy is you ever want to use
a Var for a password that is specified at ``initiate_system()`` time but don't want it to show up in any process's
environment.

=======================
Options
=======================

The ``actuator.config`` module defines the ``with_config_options()`` function that, if called from within a config
model, can set the following options for the model:

-   ``default_task_role``- This establishes a default Role to use for a task if it doesn't have an explicit
    ``task_role`` set on it. Either a model reference or a context expression.
-   ``remote_user``- This is a string that identifies a remote user name to use for remote access to a host
    if one isn't set on the task itself.
-   ``private_key_file``- string path to the private key file of a SSH keypair to use as a default private key for the
    login user identified for a task
-   ``default_run_from``- This establishes a default ``run_from`` Role for tasks, so unless a task specifies its
    own ``run_from``, this will be the value used.


