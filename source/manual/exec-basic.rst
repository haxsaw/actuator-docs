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

Let's use the SimpleNamespace from the :doc:`config models<config-basic>` example and create an execute model to use
with it:

.. code:: python

    pass

======================================
Forking and Running in the Background
======================================

Unlike tasks for a config model, tasks in an execute model are generally long-running processes. Hence, the model
author needs to be aware of the start-up behaviour of the processes they are writing tasks for, as a blocking
process can halt the progress of the execute model, and the entire orchestrator.

Actuator provides no specific support here: it doesn't know about daemonising a process, shell metacharacters that
cause background execution, or the use of nohup. This may be a feature that is added later, but currently there is
no support in Actuator starting processes in the background.

Hence, it is up to execute task authors to make appropriate choices and understand the behaviour of the software their
tasks are running. If a program can daemonise itself, then there's little for a task author to do but simply run
the program and allow it to create the daemon. If the program doesn't understand that sort of processing, it may be
necessary to use a shell task and run the program under 'nohup' and put a shell '&' at the end of the command line so
the program is run in the background.

Regardless of the method, execute task authors have the responsibility to know how to keep their tasks from blocking
and stopping the progress of execute model processing.

