************************
Config Model Basics
************************

**Import package**: actuator.config
    Key names to import: ConfigModel, with_config_options, ConfigClassTask, MultiTask, NullTask, with_dependencies

**Import package**: actuator.modeling
    Key names to import: ctxt

**Import package**: actuator.config_tasks
    Key names to import: PingTask, ScriptTask, CommandTask, ShellTask, CopyFileTask, ProcessCopyFileTask,
    LocalCommandTask, LocalShellCommandTask, WaitForTaskTask

To create a config model, you create a subclass of actuator.config.ConfigModel and fill it with tasks and dependencies
between tasks.

The configuration model is what instructs Actuator as to what to do with the infrastructure in order to make it ready to
run application software. The configuration model has two main aspects:

1. A declaration of the tasks that need to be performed
2. A declaration of the dependencies between the tasks that will dictate
   the order of performance

Together, this provides Actuator the information it needs to perform all configuration tasks relative to the proper
namespace roles and in the proper order.

==============================
Declaring tasks
==============================

Tasks are the entities that define units of work to complete. Actuator supports a number of different tasks that can
move files and run commands and scripts, both locally and remotely.

Tasks are declared relative to a Namespace and its Roles; it is the roles that inform the config model where
the tasks are to ultimately be run. In the following examples, we'll use the this simple namespace that sets up a
target role where some files are to be copied, as well as a couple of Vars that dictate where the files will go.

.. code:: python

    class SimpleNamespace(NamespaceModel):
      with_variables(Var("DEST", "/tmp"),
                     Var("FILE_TO_COPY", "someFile.txt"))
      copy_target = Role("copy_target", host_ref="127.0.0.1")
    ns = SimpleNamespace()

We've a couple of Vars at the model level and a single Role that will be the target of the file we want to copy.
Notice that we've used a hard-coded IP for the host_ref; Actuator will use that as the IP to contact for any tasks
that are associated with this role. The value of host_ref could also have been a replacement parameter and the
IP to use specified as the value of the associated Var at run time.

.. note::
    Actuator uses `Paramiko <https://pypi.python.org/pypi/paramiko>`__ for managing the remote execution of commands
    over SSH. With Paramiko, you can either supply a user name/password to the config, or else a user name and a private
    key file. If the latter is chosen, be sure that the machine referred to by ``host_ref`` has a SSH server running
    as well as the key files installed for the remote user. Clouds often provide a 'key pair' resource that can be
    used for this purpose.

Declaring tasks is a matter of creating one or more instances of various task classes. In this example, we'll declare
two tasks: one which will remove any past files copied to the target (only really needed for non-dynamic hosts), and
another that will copy the files to the target.

.. code:: python

    from actuator.config import (ConfigModel, CopyFileTask,
                                 CommandTask)
    # assume we have access to the namespace model,
    # and the file to copy is in the current working directory

    class SimpleConfig(ConfigModel):
        cleanup = CommandTask("clean-target", "/bin/rm -f !{FILE_TO_COPY}",
                            chdir="!{DEST}",
                            task_role=SimpleNamespace.copy_target)
        copy = CopyFileTask("copy-file", "!{DEST}", src="!{FILE_TO_COPY}",
                          task_role=SimpleNamespace.copy_target)

The CommandTask is saying to change to the directory identified by the ``chdir=`` keyword argument, and then to run
indicated command. The CopyFileTask is saying to copy a file to the specified destination directory; the copy should
the same name as the original.

The model is set up to run these tasks on whatever host is related to the Role identified
by the *task_role* keyword argument. Note the use of replacement parameters in the various argument strings to the
tasks; these will be evaluated against the available Vars visible to the Role identified by task_role.

Note also that these only name specific tasks to perform on some remote system; they don't identify any user credentials
to access that system. Tasks can be configured with user credentials, but if the same user is to be used over and over,
it winds up being easier and more succinct by supplying them when creating an instance of the config model:

.. code:: python

    config = SimpleConfig("simple-conf", remote_user="ubuntu",
                          remote_pass="secret password")

Unless a task specifies its own user and password (or private key file), this global one for the entire config
model will be used.

You can pull this together with the namespace and orchestrator and have Actuator do this work. Modifying the example
from :doc:`orch-basic`, we can have Actuator perform this config:

.. code:: python

    from actuator import ActuatorOrchestration
    from actuator.provisioners.aws import AWSProvisionerProxy
    # assume you have SimpleNamespace and SimpleConfig available

    # there is no dynamic infra so we don't need a provisioner proxy

    # make your namespace
    ns = SimpleNamespace("simple-ns")
    # make the config model
    config = SimpleConfig("simple-conf", remote_user="ubuntu",
                          remote_pass="secret password")

    # make your orchestrator and run it
    ao = ActuatorOrchestration(namespace_model_inst=ns,
                               config_model_inst=config)
    success = ao.initiate_system()
    print("stand up of infra was a success: {}".format(success))
    # nothing was provisioned, so there's nothing to tear down

If the credentials are correct, SSH server is running, and the the source file is in the current directory, Actuator
will be able to execute these tasks on the host of the role named with SimpleNamespace.copy_target. However, this
isn't enough to get proper results, which will be discussed next.

======================
Declaring dependencies
======================

This is a fully functional config model, but not a reliably functioning one. The reason is that Actuator
hasn't been told anything about the order of performing the config tasks. Hence, Actuator will perform these tasks
in parallel, and the end result will simply depend on the relative scheduling timings of the SSH sessions set up
for each task.

What we want to do is add dependency information to the model so that Actuator knows the proper order to perform
the tasks. To do this, we use the with_dependencies() function and the '\|' and '&' symbols to describe the
dependencies between tasks. Adding this to the above config
model, we would get the following:

.. code:: python

    from actuator.config import (ConfigModel, CopyFileTask,
                                 CommandTask, with_dependencies)
    # assume we have access to the namespace model,
    # and the file to copy is in the current working directory

    class SimpleConfig(ConfigModel):
        cleanup = CommandTask("clean-target", "/bin/rm -f !{FILE_TO_COPY}",
                            chdir="!{DEST}",
                            task_role=SimpleNamespace.copy_target)
        copy = CopyFileTask("copy-file", "!{DEST}", src="!{FILE_TO_COPY}",
                          task_role=SimpleNamespace.copy_target)

        with_dependencies(cleanup | copy)

The ``with_dependencies()`` function can be called repeatedly to define dependencies (that is, execution order) between
tasks. In the above snippet, the '|' character indicates serial execution, and says that the cleanup task must
complete before the copy task is started. With this additional information, Actuator can now produce repeatable
results.

======================
Dependency expressions
======================

By using dependency expressions and the with_dependencies() function, dependency graphs of arbitrary complexity can
be declared. Expressions can be formed that indicate serial or parallel performance of tasks, and repeated calls to
the with_dependencies() function can build up complex dependency relationships.

To illustrate the effects of multiple imvocations of with_dependencies(), let's assume a config model with five tasks
in it, t1 through t5 (what the tasks are isn't
important). The following invocations of with_dependencies() will yield identical dependency graphs,
where each task is done in series, and a task doesn't start until the one before it completes.

.. code:: python

    with_dependencies( t1 | t2 | t3 | t4 | t5 )
    # or
    with_dependencies( t1 | t2,
                       t2 | t3,
                       t3 | t4 | t5)
    # or the following, which is two
    # independent invocations of with_dependencies()
    with_dependencies( t1 | t2 | t3 )
    with_dependencies( t3 | t4 | t5 )
    # or various combinations of the above

The with_dependencies() function can take any number of dependency expressions and be invoked any number of times.
It will collect all dependency expressions from all the arguments from each invocation and assemble a dependency
graph that instructs it how to perform the config model's tasks. All tasks that don't appear an any dependency
expression are performed immediately, as well as any task that has nothing it depends upon.

The above example illustrates how to arrange tasks in series, but what about tasks that can be performed in
parallel? To indicate the eligibility of tasks to be performed in parallel, use the '&' operator in dependency
expressions. Using the same five tasks from above, the following would instruct Actuator to perform the identified
tasks in parallel:

.. code:: python

    # Perform t1 first, then t2 and t3 together,
    # and then t4 and t5 serially, but only after
    # both t2 and t3 have finished
    with_dependencies( t1 | (t2 & t3) | t4 | t5 )

    # the same, but with implicit parallelism
    with_dependencies( t1 | t2, t1 | t3, t2 | t4, t3 | t4, t4 | t5 )


    # Perform t1 and t2 together, and when both are
    # done perform t3, followed by t4 and t5 together
    with_dependencies( (t1 & t2) | t3 | (t4 & t5) )

    # the same, but with multiple expressions
    with_dependencies( (t1 & t2) | t3, t3 | (t4 & t5) )

    # the same, but with multiple invocations of with_dependencies
    with_dependencies( (t1 & t2) | t3 )
    with_dependencies( t3 | (t4 & t5) )


    # Perform t1 then t2 in parallel with t3, all of which can be done
    # in parallel with t4 then t5
    with_dependencies( ((t1 | t2) & t3) & (t4 | t5) )


    # Perform t1 through t5 in parallel, but then remember that t1 has to be
    # done before t4 and add that dependency in
    with_dependencies( t1 & t2 & t3 & t4 & t5 )
    with_dependencies( t1 | t4 )

    # the same, but after fixing your original oversight
    with_dependencies( (t1 | t4) & t2 & t3 & t5 )

As we can see, dependency expressions can be arbitrarily nested, and the expressions can be layered on additively
to create complex relationships that difficult or confusing to be expressed with a single expression. The ability
to call with_dependencies() multiple times comes in particularly handy when a config has a great many tasks and those
tasks can be arranged in related groups; one group at a time can get their dependencies specified with a single
call to with_dependencies, which aids readability.

==================
Config Granularity
==================

A config model can be written to be the analogue of an shell script that does the same work, but there are trade-offs
for each approach:

-  While you can write a config model that has a task for each shell command that would have been in an install
   script, the model won't execute as fast as the script would have due to the remote connection times. So if you
   have a very large number of tasks,
   especially if they can't be done in parallel, you may want to consider putting some or all into scripts and
   just have your task copy and run the script on the remote host.
-  On the other hand, config tasks remember if they completed successfully or not, so if a task fails beyond the stated
   retry count, the entire config model can be re-run and it will pick up at the last failed task. Shell scripts
   tend not to be written in such an 'idempotent' fashion, but would need to be written this way in order to get the
   same capabilities as a config model. This makes script writing more complex, although in many cases tests on the
   system can be executed to check if a script command has already been performed.
-  Finally, config tasks are about establishing a repeatable initial state, not holding a system config to be aligned
   to that state (in the way that some configuration management tools do). If you need ongoing management of a system's
   configuration, consider the integration of a tool like Puppet.

A good rule of thumb is that if there are many parallel tasks that are required that can take some time, it is better
to use config tasks than a single script, as parallelism is possible but tricky in a script. On the other hand, if
there are a large number of serial commands to be run that can have tests that can be written to check if the command
has already been written, you may be better off with a config script that is copied to the system and then run with
a single task. If it fails along the way, it should be safe to restart the script and pick up where it left off.


