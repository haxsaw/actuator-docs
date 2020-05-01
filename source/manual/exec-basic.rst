************************
Execute Model Basics
************************

**Import package**: actuator.execute
    Key names to import: ExecuteModel, with_execute_options, ExecuteClassTask, MultiTask, with_dependencies

**Import package**: actuator.modeling
    Key names to import: ctxt

**Import package**: actuator.execute_tasks
    Key names to import: RemoteExecTask, RemoteShellExecTask, RemoteScriptExecTask, LocalExecTask,
    LocalShellExecTask, LocalScriptExecTask, WaitForExecTaskTask

To make an execute model, you create a subclass of the actuator.execute.ExecuteModel class and then fill it with your
run-time tasks and any dependencies between those tasks.

The execute model is how you describe the software components to run once configuration is complete. Not every system
requires an execute model, as sometimes the act of installation or other configuration takes care of running software
(inetd.conf for example). Further, it is possible to carry out the work that would have been done in an execute model
in a config model. However, there are some slight differences in the semantics of their components that really set
these two classes apart.

===========================
Similarity to Config Models
===========================

As you may suspect, execute models are very similar to config models. In execute models:

-  ...you define tasks to perform which are the software programs to start,
-  ...you relate tasks to Roles in a namespace to direct where they are to be run,
-  ...you use with_dependencies() to indicate any dependency relationships between tasks,
-  ...you use the same mechanisms for specifying remote users and credentials as with config models

In short, if you're got your head wrapped around :doc:`config models<config-basic>`, you've learned the bulk of the
details for execute models.

=================
Running a System
=================

Providing a meaningful simple example of an execute model for illustrative purposes is tricky since having multiple
executables for a single system usually means there is a more elaborate application that is to be run, and that
complexity overwhelms the modeling problem. So the path we'll take here is to show a fairly trivial but complete
example and leave a more comprehensive example for a bit later in the doc.

This model will simply start up python's built-in web server to serve files in a named directory.

.. code:: python

    from actuator import ActuatorOrchestration
    from actuator.namespace import NamespaceModel, Role, with_variables
    from actuator.execute import ExecuteModel
    from actuator.execute_tasks import RemoteShellExecTask

    class PythonWebserverNamespace(NamespaceModel):
        with_variables(Var("FILES_DIR", "/tmp"))

        server_role = Role("server-role", host_ref='127.0.0.1')

    class PythonWebserverExecute(ExecuteModel):
        server_task = RemoteShellExecTask("run-webserve-task",
                                          "nohup /usr/bin/python3 -m http.server &",
                                          chdir="!{FILES_DIR}",
                                          task_role=PythonWebserverNamespace.server_role)

    if __name__ == "__main__":
        ns = PythonWebserverNamespace('server-ns')
        em = PythonWebserverExecute('server-exec', remote_user="whoever", remote_pass="whatever")
        orch = ActuatorOrchestration(namespace_model_inst=ns,
                                     execute_model_inst=em)
        success = orch.initiate_system()
        print("Start of web server returned {}".format(success))

Similar to what was done in the examples for config models, we've defined a namespace model with a single role that
uses a hard-coded IP address for the host_ref, just to make testing easier. In the execute model, we used a
RemoteShellExecTask to run our server, as we wanted to use the shell 'nohup' command to disconnect the process from
the control terminal so it doesn't get killed when the command logs out of the remote host, and second so that we
can use the shell '&' control character to tell the shell to put the process in the background so as not to block the
progress of the execution model.

======================================
Forking and Running in the Background
======================================

Unlike tasks for a config model, tasks in an execute model are generally long-running processes. Hence, the model
author needs to be aware of the start-up behaviour of the processes they are writing tasks for, as a blocking
process can halt the progress of the execute model, and the entire orchestrator.

Actuator provides no specific support here: it doesn't know about daemonising a process, shell control characters that
cause background execution, or the use of nohup. This may be a feature that is added later, but currently there is
no support in Actuator starting processes in the background.

Hence, it is up to execute task authors to make appropriate choices and understand the behaviour of the software their
tasks are running. If a program can daemonise itself, then there's little for a task author to do but simply run
the program and allow it to create the daemon. If the program doesn't understand that sort of processing, it may be
necessary to use a shell task and run the program under 'nohup' and put a shell '&' at the end of the command line so
the program is run in the background.

Regardless of the method, execute task authors have the responsibility to know how to keep their tasks from blocking
and stopping the progress of execute model processing.

=====================================================
Differences between shell and command task variants
=====================================================

Readers may have noted that there are plain RemoteExecTask classes as well as RemoteShellExecTask classes. The
difference is whether or not a shell is run and a command is run inside the shell, or if the command is run
directly. If your command would benefit from the availability of shell control characters, then one of the shell
variants of of the exec task classes is what you'll want to use in your model.
