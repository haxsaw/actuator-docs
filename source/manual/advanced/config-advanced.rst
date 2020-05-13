**************************
Advanced Config Modeling
**************************

**Import package**: actuator.config
    Key names to import: ConfigModel, with_dependencies, MultiTask, NullTask,
        with_config_options, ConfigClassTask

**Import package**: actuator.config, actuator.config_tasks
    Key names to import: NamespaceModel, Var, with_variables, with_roles, Role, RoleGroup, MultiRole, MultiRoleGroup

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

==================================
Config Model Tasks and MultiTasks
==================================

There are situations where you may wind up wanting to perform the same config tasks on a number of different
instances. To address the need for being able to run the same set of tasks on multiple instances easily, Actuator
supports treating an entire config model as a single task that can be used in another config model with the
``ConfigClassTask``. This capability
is often used to create config models for specific reusable software components which can then be composed into larger
systems. The enables:

-   The details of a specific package to be kept separate from those of a application that the package is a part of,
    enabling modular design of configuration systems.
-   Increasing re-use of the configuration in multiple application systems.

In order to support the ability to be re-used, ``ConfigClassTasks`` do not specify a ``task_role`` or ``run_from``
host; these are supplied by the model that is using the config model as a task, so the config can be used on role in
a namespace.

To illustrate this, here is a config model we defined previously in :doc:`../config-basic`, which we want to now use as
a common task in another model:

.. code:: python

    from actuator.config import (ConfigModel, CopyFileTask,
                                 CommandTask, with_dependencies)

    class SimpleConfig(ConfigModel):
        cleanup = CommandTask("clean-target", "/bin/rm -f !{FILE_TO_COPY}",
                            chdir="!{DEST}")
        copy = CopyFileTask("copy-file", "!{DEST}", src="!{FILE_TO_COPY}")

        with_dependencies(cleanup | copy)


We've removed the ``task_role=`` from each task in this model, so currently Actuator wouldn't know where to run
each.

Assuming the above model is in a Python module ``simple_cfg.py``, we can now write another model that uses this model
on more than one role's host_ref:

.. code:: python

    from actuator.config import (ConfigModel, with_dependencies,
                                 ConfigClassTask)
    from simple_cfg import SimpleConfig

    class LSNS(NamespaceModel):  # LSNS means 'less simple namespace model'
        role1 = Role('r1', host_ref='127.0.0.1')
        role2 = Role('r2', host_ref='127.0.0.1')

    class LessSimpleConfig(ConfigModel):
        copy_task1 = ConfigClassTask("file-upload1", SimpleConfig,
                                      init_args=("upload1",),
                                      task_role=ctxt.nexus.ns.role1)
        copy_task2 = ConfigClassTask("file-upload2", SimpleConfig,
                                      init_args=("upload2",),
                                      task_role=ctxt.nexus.ns.role2)

Here, we've imported the SimpleConfig model and used the model class as an argument to ``ConfigClassTask``, which
wraps the model with a task interface.
``ConfigClassTask`` looks to where it has been told to run and creates an instance of the wrapped config class that
will run on the indicated host.

This particular form passes the ``task_role=`` of ``ConfigClassTask`` into the dynamically created instance of
``SimpleConfig``, which uses it for performing the tasks on the supplied Role's host_ref. The way this model is
written with no dependencies between ``copy_task1`` and ``copy_task2``, the processing in SimpleConfig for each Role
will happen in parallel.

But even this represents some repetitive code; we need to create a ``ConfigClassTask`` for each Role in the
namespace model what we want to suite of tasks to apply to. There's another way to model this, and learning this
will be a lead-in to our next topic.

Since essentially we just use the same declarations on a number of different Roles in the namespace, we can use a
a task structuring component to direct the creation of multiple ``ConfigClassTasks`` on a variety of roles; we do
this with the ``MultiTask`` component. This task-like component applies some template tasks to a collection of Roles
from the namespace model. ``MultiTask`` creates an instance of of the template task each Role it discovers.

To see this, let's re-write the above config model class to use ``MultiTask``:

.. code:: python

    from actuator.config import ConfigModel, MultiTask, ConfigClassTask

    class LSNS(NamespaceModel):  # LSNS means 'less simple namespace model'
        role1 = Role('r1', host_ref='127.0.0.1')
        role2 = Role('r2', host_ref='127.0.0.1')

    class LessSimpleConfig(ConfigModel):
        copy_tasks = MultiTask(ConfigClassTask('file-upload',
                                               SimpleConfig,
                                               init_args=("upload",)),
                               [LSNS.role1, LSNS.role2])

So what's happening here? The ``MultiTask`` component has two arguments, a template task and a list of Roles the task
should be
applied to. In our case, the first argument, the template task, is a ``ConfigClassTask`` wraps the ``SimpleConfig``
config model in a task
wrapper. The second argument is a list of model references to Roles in the namespace model that we want to
apply the template task to.

During orchestration, Actuator launches copies of the template task in parallel for each MultiTask, each given one
of the Roles from the list of roles provided, and allowing each
to proceed at their own pace. The entire MultiTask isn't completed until all launched copies of the template have
completed.

``MultiTasks`` can appear in ``with_dependencies()`` calls to establish their order of performance just like a regular
task, and serve as a `container` for each of the spawned copies of the template.

.. note::

    Currently, Actuator does not support context expressions for the list of Roles for MultiTask to use.
    You must use either model references or selection expressions (described next).

=============================================================
Selection expressions-- tasks for arbitrary numbers of Roles
=============================================================

Listing all roles for a MultiTask fine when there are a fixed number of Roles in a namespace, but what about when
there are an unknown
number of Roles due to the use of a ``MultiRole`` in the namespace model? In such circumstances, you need to be able
to spawn as many tasks as there are Roles in the namespace.

To make this easier, Actuator provides a way to create a `selection expression` that queries the namespace and returns
a collection of Roles that meet some query criteria. This collection can then be handed to MultiTask instead of
explicit list of model references to Roles.

Selection expressions are part of model references; they can't currently be created with context expressions. A
selection expression is initiated by using the ``q`` attribute on a namespace model, and then accessing model
attributes via ``q``. This results in an expression whose value is only determined when the MultiTask needs to know
how many copies of its template to make.

Let's look at an example: first, consider the following namespace model (we'll aim all ``host_ref=`` arguments to
the localhost to keep things simple):

.. code:: python

    from actuator.namespace import NamespaceModel, Role, MultiRole

        class GridNamespace(NamespaceModel):
            with_variable(Var("HOME", "/home/!{USER}"),
                          Var("USER", "ubuntu"),
                          Var("TEMP", "/tmp"),
                          Var("HOME_TEMP", "!{HOME}!{TEMP}))
            grid_roles = MultiRole(Role('grid-node', host_ref="127.0.0.1"))

This names an arbitrary number of ``grid_roles``; we won't know how many until an instance of the model has been
created and some ``grid_roles`` are accessed via keys.

Suppose we wanted to be ensure there was a ``tmp`` directory in our user's home, and that it was empty (we want to be
sure it didn't already exist and have some cruft left over from other activities). We can do this with a
``ShellTask`` like the following (assuming we're logged in as a premissioned user):

.. code:: python

    ShellTask('make-home-temp', '/bin/mkdir -p !{HOME_TEMP}; /bin/rm -rf !{HOME_TEMP}/*')

So we want that task to run on every host associated with all our ``grid_roles`` in our namespace. We can use
MultiRole with a selection expression as follows:

.. code:: python

    from actuator.config import ConfigModel, ShellTask, MultiTask

    class GridConfig(ConfigModel):
        make_homes = MultiTask(ShellTask('make-home-temp',
                                         '/bin/mkdir -p !{HOME_TEMP}; /bin/rm -rf !{HOME_TEMP}/*'),
                               GridNamespace.q.grid_roles.all())

The argument we're interested in here the the final one to ``MultiRole``, ``GridNamespace.q.grid_roles.all()``. This
is a selection expression, and in this case we're simply selecting all the ``grid_roles`` to run the ``ShellTask``.
Taking this expression a bit at a time, we have:

-   **GridNamespace.q** opens the selection expression.
-   **grid_roles** identifies the MultiRole in which we're interested.
-   **all()** says that we want all of the Roles within this MultiRole

As a shortcut, if you simply want all the Roles in ``MultiRole``, then you can leave off the 'all()'; hence:

.. code:: python

    GridNamespace.q.grid_roles

is the same as:

.. code:: python

    GridNamespace.q.grid_roles.all()

Selection operations
--------------------

Selection expressions can do more than just identify a whole group of Roles; they can be used to aggregate groups,
identify specific members of groups, and even find roles buried in nested ``MultiRole`` constructions.

With the exception of the ``union()`` operation, all operations in a select expression can be chained into longer,
more complex expressions that can include applying operations to sub-components. When some component satisfies one
operator, the result is a set of components that subsequent operations can be applied to. You can then continue the
expression down a common attribute of the selected set and then apply another operation. This allows fairly deep and
precise selection of Roles for ``MultiTask`` tasks.

union()
^^^^^^^

If you have a disjoint set of Roles that you'd like to apply a task to using ``MultiTask``, you can easily do so
using the ``union()`` operation on the selection expression. For example, below we have two ``MultiRole`` components
in a namespace model, but we can use ``union()`` to associate a MultiTask to them both:

.. code:: python

    from actuator.namespace import NamespaceModel, Role, MultiRole
    from actuator.config import ConfigModel, MultiTask, ShellTask

    class UnionNamespace(NamespaceModel):
        # we'll leave the hostref= off of the Roles to keep things simple
        grid1 = MultiRole(Role('grid-1'))
        grid2 = MultiRole(Role('grid-2'))

    class UnionConfig(ConfigModel):
        upgrade_all = MultiTask(ShellTask('upgrade-packages',
                                          'sudo /usr/bin/apt-get upgrade -y'),
                                UnionNamespace.q.union(UnionNamespace.q.grid1,
                                                       UnionNamespace.q.grid2))

The ``union()`` operation is available right on the ``q`` attribute to union together any other Roles that are
selected from the namespace model. This would allow the ``ShellTask`` to be run on all hosts associated with the
selected roles.

key()
^^^^^

There may be cases where you want run tasks on some subset of ``Roles`` in a ``MultiRole`` component. The ``key()``
operator provides a way to select just some of the Roles in such a situation.

To see this, let's reuse an example from :doc:`namespace-advanced` that dealt with ``MultiRoleGroups``:

.. code:: python

    from actuator import ctxt
    from actuator.infra import StaticServer, InfraModel, MultiResourceGroup, MultiResource
    from actuator.namespace import NamespaceModel, Role, MultiRoleGroup, MultiRole

    class GridInfra(InfraModel):
        grids = MultiResourceGroup('grids',
                                   master=StaticServer('master', "192.168.1.1"),
                                   slaves=MultiResource(StaticServer('slave', "192.168.1.2"))

    class GridNS(NamespaceModel):
        grid_roles = MultiRoleGroup('grid',
                                    master=Role('master',
                                                host_ref=ctxt.nexus.inf.grids[ctxt.comp.container.key].master),
                                    slaves=MultiRole(Role('slave',
                                                          host_ref=ctxt.nexus.inf.grids[ctxt.comp.container.container.key].slaves[ctxt.name])))

This has a ``RoleGroup`` with an embedded ``MultiRole``. Suppose we wanted to use this model for setting up location-based
compute grids, and so we populated an instance of the namespace model as follows:

.. code:: python

    # suppose we make some differently sized grids depending on location
    # so we can build a simple dict that has locations as keys and
    # the number of compute slaves as values:

    location_sizes = {'LN': 20,
                      'NY': 40,
                      'TK': 15}

    # and then used it to populate an instance of our namespace
    ns = GridNS('all-example')
    for k, v in location_sizes.items():
        for i in range(v):
            _ = ns.grid_roles[k].slaves[i]

So what's happening here? The value of ``k`` in the outer ``for`` loop will be one of 'LN', 'NY', or 'TK', which will result
in the generation of ``RoleGroups`` accessible via those keys. Then within each of those ``RoleGroups``, we'll create
``v`` slaves in the inner ``for`` loop with keys of ``0`` to ``v-1``. So there will be a total of 75 slaves, but they
will be unevenly distributed across the locations.

Now suppose that from a config model perspective we need to perform a specific config step for any hosts at all in
the NY location, whether a slave or not. To do this, we can use the ``key()`` operation to only select model
components under the NY key:

.. code:: python

    from actuator.config import ConfigModel, ShellTask, MultiTask

    class GridConfig1(ConfigModel):
        NY_only_tasks = MultiTask(ShellTask("NY-task", "/some/NY/only/command"),
                                  GridNS.q.union(GridNS.q.grid_roles.key('NY').master,
                                                 GridNS.q.grid_roles.key('NY').slaves.all()))

So we've used ``key`` in two different selection expressions and then applied ``union()`` to the results to get a
collection of Roles to apply the ``ShellTask`` to.

The first expression is ``GridNS.q.grid_roles.key('NY').master``; broken down, this says:

-   Use **GridNS.q.grid_roles** to pick the MultiRoleGroup in the namespace model.
-   Use **.key('NY')** to pick the only the RoleGroup under the NY key.
-   Use **.master** to select the master ``Role`` in NY

Then union that result to ``GridNS.q.grid_roles.key('NY').slaves.all()``; broken down this is:

-   Use **GridNS.q.grid_roles** to start the selection expression and to pick the MultiRoleGroup in the namespace model.
-   Use **.key('NY')** to pick the only the RoleGroup under the NY key.
-   Use **.slaves.all()** to select all of the slave ``Roles`` in the ``RoleGroup``

This gives us a way to take a vertical slice through the namespace and apply tasks to the roles in the slice.

We could also have looked only for specific slaves. Suppose we only wanted to run the special task on the first slave
in each location, the slave with a key of 0. To do this, we could write the selection expression as follows:

.. code:: python

    from actuator.config import ConfigModel, ShellTask, MultiTask

    class GridConfig2(ConfigModel):
        zero_only_tasks = MultiTask(ShellTask("zero-task", "/some/zero/command"),
                                    GridNS.q.grid_roles.all().slaves.key(0))

We can use a single selection expression for this so union isn't required. The expression,
``GridNS.q.grid_roles.all().slaves.key(0)`` breaks down as follows:

-   **GridNS.q.grid_roles** starts the selection expression and names the MultiRoleGroup in the namespace model.
-   **.all()** selects `all` of the RoleGroups; any further further attributes or operators in the expression will be
    applied to all of the selected ``RoleGroups``. Unlike when a collection is the final element in selection
    expression, when in the middle the ``all()`` operation is required.
-   **.slaves** selects the ``slaves`` ``MultiRole`` in each selected ``RoleGroup``.
-   **.key(0)** only selects the Role from each ``RoleGroup`` that has the key `0`.

This example shows more of how we can take a `horizontal` slice of Roles in a namespace's Role hierarchy. Note that this
example doesn't select any of the ``master`` Roles.

keyin()
^^^^^^^

Sometimes you want to select Roles with one of several key values. The ``keyin()`` operation allows you to specify a
sequence of key values and any component with that key will be selected.

For example, here's a version that specifies to use the slaves from only NY on LN:

.. code:: python

    from actuator.config import ConfigModel, ShellTask, MultiTask

    class GridConfig3(ConfigModel):
        nyln_only_tasks = MultiTask(ShellTask("NYLN-task", "/some/ny-ln/command"),
                                    GridNS.q.grid_roles.keyin(['NY', 'LN']).slaves.all())

Here, the sub-expression ``GridNS.q.grid_roles.keyin(['NY', 'LN'])`` selected at most two RoleGroups, and then from
there it selected all of the ``slaves`` in each.

.. note::

    A similar effect can be had by just using ``key()`` multiple times within a ``union()`` operation.

match() and no_match()
^^^^^^^^^^^^^^^^^^^^^^

Another way to select components is match their keys to a regular expression using ``match()``. The argument to match
is a standard Python regular expression. If a key matches the RE then that item is selected.

Here's a variation of the previous config model with a MultiTask that only runs the special command on slaves in
locations that start with 'T':

.. code:: python

    from actuator.config import ConfigModel, ShellTask, MultiTask

    class GridConfig4(ConfigModel):
        t_only_tasks = MultiTask(ShellTask("t-only-task", "/some/t/command"),
                                 GridNS.q.grid_roles.match("T.*").slaves.all())

The ``no_match()`` operation is almost identical, but it matches all of the keys that `do not` match the supplied
regular expression.

pred()
^^^^^^

The ``pred()`` operation allows you to supply your own function that will be passed a key it can evaluate and, if the
item should be included in the set of selected items, return True, otherwise return False.

Here's an example that only selects slaves with an even numbered key:

.. code:: python

    from actuator.config import ConfigModel, ShellTask, MultiTask

    def just_evens(key):
        return int(key) % 2 == 0

    class GridConfig5(ConfigModel):
        evens_tasks = MultiTask(ShellTask("evens-task", "/some/even/command"),
                                GridNS.q.grid_roles.all().slaves.pred(just_evens))

.. note::

    For any of the Multi* components in Actuator, all keys are coerced to strings, and so if another data type is
    desired when looking at a key, it must be converted back.

.. note::

    The ``pred()`` operation is of limited value in that it does not reanimate from the persisted form reliably. This
    is a restriction that may be lifted in the future.

