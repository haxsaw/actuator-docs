************************
Orchestration Basics
************************

**Import package**: actuator
    Key names to import: ActuatorOrchestration

**Import package**: actuator.provisioners.aws
    Key names to import: AWSProvisionerProxy (to support provisioning AWS infra resources)

**Import package**: actuator.provisioners.azure
    Key names to import: AzureProvisionerProxy (to support provisioning Azure infra resources)

**Import package**: actuator.provisioners.openstack
    Key names to import: OpenStackProvisionerProxy (to support provisioning Openstack infra resources)

Although not part of modelling, you can't get Actuator to do any provisioning without an orchestrator (although you can
do lots of other things with models), so we'll start with going over ochestrator basics so you have something to test
models with.

Orchestration brings all of the models together and manages their processing in order to provision, configure and
execute an instance of the system being modeled. Orchestration is flexible in that it can only do part of the job if
that's required; for instance, if you only need to have infra provisioned you can simply supply the infra model and let
the orchestrator handle that, or alternatively if you have a namespace that's populated with fixed IP/hostnames for
host_ref values for all Roles, you can have the orchestrator manage just the configuration tasks against the set of
hosts in the namespace model. This allows you to use the orchestrator in variety of circumstances, such as config
model development or provisioning of infra for other purposes, as well as standing up whole systems.

The orchestrator is an instance of the ``actuator.ActuatorOrchestration`` class. You give it a
number of different models as keyword arguments as well as a list of provisioners for the resources in the infra model,
and then ask it to 'initiate' the system the models represent.

The following Python code shows a simple example of using an AWS proxy to orchestrate the creation of an infra
model named MyInfraModel in the my_models.py module. To keep things simple, error checking has been removed:

.. code:: python

    import sys
    from actuator import ActuatorOrchestration
    from actuator.provisioners.aws import AWSProvisionerProxy
    from my_models import MyInfraModel

    # make the provisioner proxy. normally you wouldn't hardcode credentials
    # in a script, but we'll do so here in order to keep the example brief

    # make your provisioner
    proxy = AWSProvisionerProxy("aws-proxy",
                                default_region="eu-west-2",
                                aws_access_key="your test key",
                                aws_secret_access_key="your secret key")
    # make your model instance
    myInfra = MyInfraModel("test-instance")

    # make your orchestrator and run it
    ao = ActuatorOrchestration(infra_model_inst=myInfra,
                               provisioner_proxies=[proxy])
    success = ao.initiate_system()
    print("stand up of infra was a success: {}".format(success))
    print("hit return to tear the system down")
    _ = sys.stdin.readline()
    ao.teardown_system()

For the above to work, you'll obviously need an AWS account and credentials that has permission to create resources
in the region named with ``default_region``.

The orchestrator method ``initiate_system()`` blocks until all work specified in the models is complete, and returns
True if the orchestration was successful. In the example, the script simply reports the return result of
``initiate_system()`` and then waits for the user to hit <return>, at which point it starts to tear the system back
down. If the orchestrator encounters unrecoverable errors, these are logged to standard error as critical log messages
so the user can see what has gone awry.

You can use this as a starting point for developing your first models for testing as you read about model development
in the following sections.
