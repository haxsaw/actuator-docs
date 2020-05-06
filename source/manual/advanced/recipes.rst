***************
Recipes
***************

This section describes some useful patterns getting work done or ways to deal with tricky problems.

====================
Templates
====================

Quite often you need to copy a config file to a server during config model processing, but the config file requires
some data that was generated during another phase or processing, often provisioning the infra model, or just from
some external source of the information. This is a
straightforward task with the ProcessCopyFileTask Task object. Like CopyFileTask, this Task copies a file from the
local system to some remote host, but it does so by first searching through the file for replacement parameters and
then finding their value in the namespace model relative to the Task's Role.

As an example, here's a snippet of the Zabbix agent's config file (Zabbix is an open source monitoring system) that
identifies the IP of 'zabbix server', the only host that is allowed to connect to the agent's port:

.. code::

    Server=!{ZABBIX_SERVER}

Normally, an admin would config the value of Server above with the name or IP of one of the Zabbix servers. But we want
Actuator to do this for us. We can do this in a namespace model by setting up a Var as follows:

.. code:: python

    class TemplateNamespace(NamespaceModel):
        with_variables(Var("ZABBIX_SERVER", '192.168.1.1'))

        some_role = Role('example-role',
                         host_ref=ctxt.nexus.inf.example_server)

    class TemplateConfig(ConfigModel)
        zabbix_config = ProcessCopyFileTask("zabbix-agent-config",
                                            "/config/dest/path",
                                            src="/config/src/path",
                                            task_role=ctxt.nexus.ns.some_role)

In this example, we made a Var with a hard coded IP address for a Zabbix server and a demo role, and then in the
config class we created a ProcessCopyFileTask that will pick up a template config file that contains the
``Server=!{ZABBIX_SERVER}`` line it in, and then replace that with the value of the variable in the namespace.

Where to locate templates? In our view, it is best to locate them in a templates sub-directory of the directory where
the model files live. This way it is simple for these templates to be found when the models are executed. Alternative,
they could be in some repository and fetched with a LocalCommandTask, and then sent to the remote host using
ProcessCopyFileTask.

.. note::

    The value of ZABBIX_SERVER could have been provided a number of different ways: if the server was part of the infra,
    either as something provisioned or a StaticServer, the value for the ZABBIX_SERVER Var could have been something
    like ``ctxt.nexus.inf.zabbix_server.get_ip``. Or, the value could have been None, and just after the namespace has
    been instantiated but prior to orchestration, the value could have been updated with the ``add_variable()`` method
    on the namespace model, or overridden with the ``add_override()`` method, that puts in an effective Var value into
    the namespace until told otherwise. The larger point is that there are a number of ways to get values into
    Vars.


================================================
Creating a single Var from a group of components
================================================

**Import package**: actuator.namespace
    Key names to import: multicomp_string_builder

There are lots of cases when creating config models that you find yourself having to populate a config file with a
list of values that come from various components in another model. You can do this if there are a fixed number of
components, but it isn't very nice, but it gets tricky if there are variable numbers of components in your model. You
can play games with shell commands written in a ShellTask, but it's awkward and more work than it should be. Actuator
provides some tools for making this an easier task with the ``multicomp_string_builder()`` decorator. Let's look at an
example involving creating a config file for HAProxy that has all the IP addresses for the servers the managed by the
load balancer.

The ``backend servers`` section of an HAProxy config file looks like the following:

.. code::

    backend servers
        server server1 127.0.0.1:8000 maxconn 32
        # and another line for each server the proxy load balances across

Actuator can easily provision all the servers required in this situation, but straightforward template create only
really works with a fixed number of servers:

.. code:: python

    class LB1Infra(InfraModel):
        servers = MultiResource(AWSInstance("creation arguments"))

    class LB1Namespace(NamespaceModel):
        with_variables(Var("Server1-IP", ctxt.nexus.inf.server[0].get_ip),
                       Var("Server2-IP", ctxt.nexus.inf.server[1].get_ip),
                       # and so forth
                      )
        # now we need roles for each server we want data from
        server_roles = MultiRole(Role(host_ref=ctxt.nexus.inf.server[ctxt.name]))

...and then a config template file that looks like:

.. code::

    backend servers
        server server1 !{Server1-IP}:8000 maxconn 32
        server server2 !{Server2-IP}:8000 maxconn 32
        # and so forth

Yikes.

There is a *much* better way to deal with these kinds of situations, and that is by using ``multicomp_string_builder()``.
By applying this decorator to a simple method that returns a single formatted component of the config file section you
wish to populate, and then creating a Var whose value is a context expression aimed at the function, you can then
use the Var in your template file and Actuator will take care of aggregating the lines for you.

This may sound like a mouthful, but it's actually pretty simple-- let's see it in action. We'll start again with the
same infra model as before, but then declare a method on our namespace that formats a single line in the config file:

.. code:: python

    class LB1Infra(InfraModel):
        servers = MultiResource(AWSInstance("creation arguments"))

    class LB2Namespace(NamespaceModel):
        server_roles = MultiRole(Role('server', host_ref=ctxt.nexus.inf.servers[ctxt.name]))

        @multicomp_string_builder(ctxt.model.server_roles, sep_str="\n")
        def make_haproxy_backend_server_line(self, r):
            return "    server {} {}:8000 maxconn 32".format(r.host_ref.name.value(),
                                                             r.host_ref.get_ip())

        with_variables(Var("BACKEND_IPS", ctxt.model.make_haproxy_backend_server_line))

...and then a much simpler config template file like:

.. code::

    backend servers
    !{BACKEND_IPS}

That's a lot more flexible and descriptive, and will work with any number of roles and servers.

There's a lot to unpack here, so the best approach is to work backwards from the config file. There, we want a single
Var's replacement parameter to get replaced with a list of configuration lines, one for each server. To do that, we
go back into the namespace model and define the Var BACKEND_IPS as a context expression that refers to a method of our
namespace model ``make_haproxy_backend_server_line()``. We write this method such that it is given a single Role and from
there returns a single line formatted the way HAProxy needs for the ``backend servers`` section of its config file. To
get this method to receive a single role, we use the ``multicomp_string_builder()`` decorator: the first argument is a
context expression of our model's MultiRole ``server_roles`` component that we want to get the associated IPs for, and
the second argument is what string we want to use to join multiple returned lines with, in this case a newline.

The ``multicomp_string_builder()`` decorator is able to take the context expressions that it is given and execute the
method it decorates once for each object indicated by the expression. You can supply multiple expressions if you wish,
any any expression may refer to a single component or some kind of multi-component. In this way,
``multicomp_string_builder()`` servers as an output *aggregator*, calling the method it decorates multiple times and then
aggregating the separate results into a single value. In this example, this aggregated value becomes the value of
the BACKEND_IPS Var, which gets put into our template when it is copied with a ProcessCopyFileTask() task in a config
model.

.. note::

    The ``multicomp_string_builder()`` decorator is for *methods only*, not stand-alone functions. This has to do with
    issues around persistence and reanimation of Actuator models and objects. It's arguably better from a documentation
    perspective anyway to put such methods directly into the namespace model that uses them.

.. note::

    Notice that we passed a context expression leading to roles into ``multicomp_string_builder()``,
    not to the associated servers. Doing the former means we have to go through the host_ref attribute to find the
    server and extract the values we want. We could have instead written a context expression to the servers themselves,
    like ``ctxt.nexus.inf.servers``, and then just adjusted the ``make_haproxy_backend_server_line()`` method to work with
    servers not roles. However, by only using roles we keep the coupling to the infra model low and only through the
    Role's host_ref; we make no assumptions as to the names of servers in any other part of the model. This is preferred
    method for supplying context expressions to ``multicomp_string_builder()``, although either will work.

Just as a final note, the ``sep_str=`` keyword parameter can be changed to put all values onto a single line rather than
multiple lines. Suppose that instead of multiple lines, we needed all server IPs on a single line separated by a space.
We could then have written the above decorated function as follows:

.. code:: python

    class LB2Namespace(NamespaceModel):
        server_roles = MultiRole(Role('server', host_ref=ctxt.nexus.inf.servers[ctxt.name]))

        @multicomp_string_builder(ctxt.model.server_roles, sep_str=" ")
        def make_haproxy_backend_server_line(self, r):
            return "{}".format(r.host_ref.get_ip())

        with_variables(Var("BACKEND_IPS", ctxt.model.make_haproxy_backend_server_line))

Then ``make_haproxy_backend_server_line()`` would simply return strings with IPs, and ``multicomp_string_builder()``
would join those separate IPs with a space, returning the resulting single string.
