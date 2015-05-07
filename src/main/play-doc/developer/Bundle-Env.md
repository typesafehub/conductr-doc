# Typesafe ConductR %PLAY_VERSION%


## The bundle component environment

When your application or service runs then it is presented with a swarth of environment variables that can be used to assist it to run. One popular set of environment variables is that which tells your application or service on which address and port to bind (the _name_\_BIND\_PORT and _name_\_BIND\_IP).

## Standard Environment Variables
For reference, the following standard environment variables are available to a bundle component at runtime:

Name                      | Description
--------------------------|------------
BUNDLE_ID                 | The bundle identifier associated with the bundle and its optional configuration.
BUNDLE_SYSTEM             | A logical name that can be used to associate multiple bundles with each other. This could be an application or service association and should include a version e.g. myapp-1.0.0.
BUNDLE_HOST_IP            | The IP address of a bundle component's host.
CONDUCTR_CONTROL          | A URL for the control protocol of ConductR, composed as $CONDUCTR\_CONTROL\_PROTOCOL://$CONDUCTR\_CONTROL\_IP:$CONDUCTR\_CONTROL\_PORT
CONDUCTR_CONTROL_PROTOCOL | The protocol of the above.
CONDUCTR_CONTROL_IP       | The assigned ConductR's bind IP address.
CONDUCTR_CONTROL_PORT     | The port for the above. Inaccessible to containerized bundles such as those hosted by Docker.
CONDUCTR_STATUS           | A URL for components to report their start status, composed as $CONDUCTR\_STATUS\_PROTOCOL://$CONDUCTR\_STATUS\_IP:$CONDUCTR\_STATUS\_PORT
CONDUCTR_STATUS_PROTOCOL  | The protocol of the above.
CONDUCTR_STATUS_IP        | The assigned ConductR's bind IP address.
CONDUCTR_STATUS_PORT      | The port for the above.
SERVICE_LOCATOR           | A URL composed as $SERVICE\_LOCATOR\_PROTOCOL://$SERVICE\_LOCATOR\_IP:$SERVICE\_LOCATOR\_PORT
SERVICE_LOCATOR_PROTOCOL  | The protocol of the above.
SERVICE_LOCATOR_IP        | The interface of an http service for resolving addresses.
SERVICE_LOCATOR_PORT      | The port of the above.
SERVICE_PROXY_IP          | The interface of this bundle's proxy.
CONTAINER_ENV             | A colon separated list of environment variables that will be passed through to a container. When overriding this be sure to include its original value e.g. CONTAINER_ENV=$CONTAINER_ENV:SOME_OTHER_ENV..

In addition the following environment variables are declared for each component endpoint:

Name                 | Description
---------------------|------------
name_PROTOCOL        | The protocol of a bundle component's endpoint.
name_HOST            | A bundle component's host URL composed as $name\_PROTOCOL://$name\_HOST\_IP:$name_HOST_PORT
name_HOST_PORT       | The port exposed on a bundle's host.
name_BIND_IP         | The interface the component should bind to.
name_BIND_PORT       | The port the component should bind to.
name_OTHER_PROTOCOLS | Any other protocols shared by this bundle with the same endpoint name and system
name_OTHER_IPS       | Any other interfaces shared by this bundle with the same endpoint name and system
name_OTHER_PORTS     | Any other ports shared by this bundle with the same endpoint name and system

The _OTHER_ variables above are `:` separated lists with each element correlating to each other e.g. the second element of `name_OTHER_IPS` will correspond to the second element of `name_OTHER_PORTS`. If the list is empty then any bundle component may assume that it is the first bundle of the associated system to start and no other will start until this one has signalled that it is ready. Otherwise these _OTHER_ variables can be used as "seeds" for other nodes in order to establish clusters.


Docker becomes relevant when there are specific runtime dependencies that are different to ConductR's host OS environment. In particular if a binary program that does not use the JVM is required to be launched from a bundle then it becomes more likely to benefit from using a Docker container.
