# Example of Docker images with WebLogic server in MSI Mode

This Dockerfile extends the Oracle WebLogic image by creating a domain that configures a managed
server in MSI mode (or Managed Server Independence mode). In this mode, a managed server can run
without the need of an admin server. Such a managed server is not driven by admin server for
configuration or deployment changes. However, it can handle all configuration and deployments
already in config.xml like any other managed server. For use cases where the managed server does
not need to be updated for configuration or deployments, this image can be used by itself without
running a admin server or node manager

## How to build and run the base image
First make sure you have built **oracle/weblogic:12.2.1.2-developer**. 

Next, to build the base msi image, run:

`$ docker build -t 12212-msiserver .`

Finally, to start the Managed Server in MSI mode, run:

`$ docker run -d -p 8011:8011 --name msi-server 12212-msiserver`

You can examine the stdout of this container using logs command:

`$ docker logs msi-server`

Among the top few lines from logs, you'll notice a line that states the randomly chosen server name

`MS Name to be used:  ms9`

Further down you'll notice a catalog message BEA-150018 indicating server started in Managed Server
Independence mode

`<May 16, 2017, 6:30:09,993 PM GMT> <Info> <Configuration Management> <cf579fd131fc> <> <Thread-11> <> <> <> <1494959409993> <[severity-value: 64] [partition-id: 0] [partition-name: DOMAIN] > <BEA-150018> <This server is being started in Managed Server independence mode in the absence of the Administration Server.>`

Also note that the this server does not have a publicly accessible URL since no application is
deployed to it yet. When additional images are created by adding application(s) to this base image
the same command (as above) may be used to launch the server and accessed using the URL
http://localhost:8011/<relevant-context-root>

Randomly generated managed server name can be overridden using build arguments or runtime variables.

### Build Arguments
#### ms_name_prefix
This argument may be used to alter the prefix of managed server name, and a random
number prefix is appended to it. For example, the following command may be used to
run a managed with a name managedServer<RandomNumber>

`$ docker build -t 12212-msiserver --build-arg ms_name_prefix=managedServer .`

#### number_of_ms
By default, this image comes configured with 10 managed servers, ms1 to ms10. However, the image
can be built with configurable number of managed servers using NUMBER_OF_MS argument

`$ docker build -t 12212-msiserver --build-arg number_of_ms=15 .`

#### domain_name
domain_name may be used to identify the name of generated domain that gets packed into MSI image. Default value msi-sample

#### domains_dir
domains_dir may be used identify the directory under $ORACLE_HOME where the domain home directory is created. Default value wlserver/samples/domains 

#### ms_port
ms_port may be used to configure port of the managed server, default is 8011

#### prod_or_dev
prod_or_dev may be used to identify whether server is started in production or development mode. Defaults to "dev" for development mode.

### Runtime Arguments
#### MS_NAME

This argument may be used to completely override the managed server name.
For example, the following command may be used to run a managed server with name ms1

`$ docker run --name msiserver --env MS_NAME=managedServer1 12212-msiserver`

## How to use the base image to add application
To build, run:
`$ docker build -t 12212-summercamps-msiserver -f Dockerfile.addapp --build-arg name=summercamps  --build-arg source=apps/summercamps.ear --build-arg simple_filename=summercamps.ear .`

Dockerfile.addapp adds an app called summercamps.ear targeted to the cluster that
the managed server is a part of

To start the Managed Server in MSI mode with this app, run:

`$ docker run -d -p 8011:8011 12212-summercamps-msiserver`

summercamps app will now be accessible at http://localhost:8011/

As with the base image, you can still override managed server name and cluster name

### Build arguments

Three build arguments may be used to customize the image to include an application of user
choice. By default, the build arguments point to an example application included in this
sample. The "name" argument helps identify the name of deployment, while the "source"
argument helps identify the source of the application. The source is copied into the image.
So "simple_filename" helps identify the name of the file where the source is copied to

## Using swarm service creation with this image
The image called 12212-summercamps-msiserver can be used for Docker service creation
to scale out to multiple replicas. If you don't have publishing rights to a Docker
registry you can start one locally. More details can be found here
https://docs.docker.com/registry/deploying/

A service can be created using this image. You may need init swarm mode if not already done so. Please refer to Docker swarm
documentation https://docs.docker.com/engine/swarm/

`$ docker service create --name city_activity_guide -p 8011:8011 --hostname "msihost" --host "msihost:127.0.0.1" --env "MS_NAME=ms{{.Task.Slot}}" --replicas 3 localhost:5000/12212-summercamps-msiserver:latest`

The following experimental feature in 17.03.1-ce is useful for examining logs for all
service replicas

`$ docker service logs city_activity_guide`

In the above example, a service is created with 3 replicas. We use templatized --env
to pass in a custom managed server name to the service. In this example a built-in
load balancer exposes port 8011. Accessing the URL will send the request to the three
replicas based on default load balancing algorithm. You should be able to see managed
server name printed in response:

`$ curl http://localhost:8011/`

If you use a browser, the session will be sticky. And you can see a session
variable called "Sports camps" being updated

You can scale the service using another service command. For example, you can scale up to
5 replicas using the following command

`$ docker service scale city_activity_guide=5`

Or you can scale down to say one replica

`$ docker service scale city_activity_guide=1`

If you are testing on different browser instances and you can verify that each browser is
hitting a different replica, you'll notice them all move to the same replica when you scale
down to 1

You can shutdown the service using

`$ docker service rm city_activity_guide`

Copyright (c) 2017 Oracle and/or its affiliates. All rights reserved.
