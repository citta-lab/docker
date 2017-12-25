Docker [ juggling pebbles ]
--------------------------
Deep dive of docker, mesos and marathon to deploy applications.

### Introduction

In VM architecture hypervisors runs on top of the physical host and provide a platform for guest VM's (VM's we use to deploy apps) on top and independent of the actual physical machine. So hypervisors makes the underlying software independent of the server hardware, and helps to run multiple VM's and make use of the server resource, and easy to migrate, and guest VM's are independent of each other so any malware attack on VM wouldn't effect the other, backing of VM are nothing but taking the snapshot of VM's memory space and saving it in the server disk ( A VM is essentially little more than code operating in a server's memory space ).
    However the problem is each guest VM need it's own operating system, which consumes extra memory and disk space, and probably each operating system needs licensing. So if we have 10 VM's and each VM needs 20GB of hard disk, 4GB of RAM and 5% host CPU then it would be 200GB, 40GB and 50% of total resource needed even before the app's are installed on the VM's.

1. Containers ?

Containers are created by splitting the linux `user space` into smaller chunks instead of typical design of one user space where app is installed. These smaller user space acts as containers and talks to linux kernel ( so we would only need one instance of linux operating system unlike VM's ). Other than that containers are light weight and easy to manage.

2. How does containers work ?

Each containers will it's own view of root file system, independent process tree (no PID conflict), and network tree so they are independent of each other and will have it's own root directories. These feature is leveraged by linux `Kernel Namespace` which isolates these containers, and we can control resource using ( CPU, Space, RAM etc ) `cGroup` which will have 1-1 mapping with the containers. So containerA can have more CPU in the host and RAM than containerB using cGroups ( cGroupA can be assigned with more resource than cGroupB ). Similarly `Capabilities` are used for user access and alter the root access per container.

### Docker

Docker uses `container` framework to create standard `runtime` which can be ran on any machines. i.e it will have all the environment that the application needs to run ( hence runtime, example: JDK version, privileges, api's etc ).So the docker containers runs on docker engine which talks to base platform ( such as AWS, Azure, Laptop etc). The way Docker Engine ( Docker Demon ) talks to linux kernel is via libcontainer ( written by Docker team ) which is an execution driver replaced old LXC ( linux execution driver ). Now libcontainer is part of Docker Engine.

1. It works on my laptop, not in staging ( certification ) ?

Most often we code in our workstation / laptop and testing it before deploying it to development server and/or test server and then to staging. But the chances of making it to staging perfectly is very hard and it doesn't work the same way it worked on our laptop. So docker container standardizes this workflow and make it more efficient, developers can code and deploy apps in local docker container ( running locally on laptop ) and deploy the unchanged code to test or staging docker containers. Hence these containers are uniform and independent of the operating system it will work great.

### Docker nuts and bolts

>> Docker Engine    ----->   Docker Image ------> Docker Containers

#### Docker Engine

Docker Demon / Docker runtime / Docker Engine provides the infrastructure needed for the application ( such as env variables, resource allocation, root file access, kernel access, network access etc ) and all docker engines are exact replica ( so they are all same ). In short "it's standardized runtime environment which runs and looks the same".

>> " Helps to run Docker Containers"

#### Docker Images

Docker images are manifest / list of instructions on how to build build and items in the container. `Docker container` is launched from `Docker Image`, so we need to have the Docker Image locally on `Docker Engine` or pull from the `Docker Hub` if docker image copy is not present locally.

1.0 Run Docker Image
```java
docker run -it fedora /bin/bash
// docker run 'interactive' 'image_name' `running_shell`
```
2.0 Pull Docker Image (latest)
```java
docker pull fedora
//docker pull `image_name`
```
2.1 Pull Docker Image (all)
```java
docker pull -a fedora
//docker pull `all` `image_name`
```
3.0 Check Images
```java
docker images fedora
//docker images `images_name`
```
Docker images are locally stored on `/var/lib/docker/<storage_driver>/`.In Mac it is stored in `Users/user_name/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64.linux/` or we can also open this by opening Docker App > Preferences > Advance > (Disk Image Location) Open in Finder.

>> " Contains data and Meta-data to start Docker Container "

Docker Images are made-up of one or many image layers. Example: The OS ( Ubuntu ) can act as base layer 0 and application Nginx can be installed on Ubuntu as layer 1 etc.These image layers gets it's own image id. All these layers are mounted together to form a single view and the mounting is called `union mount`.

```java
docker images --tree //layered tree display
```

#### Docker Containers

Docker Images are for build time construct, and containers are for run time. So we can say `Docker Containers` are running instances of `Docker Images` and is launched from Docker Image. `docker run` command launches a container by grabbing the specified image, unpacks it, stacks it in layer and builds it into running container.
  So docker containers are bare-minimum running instances of the image so it will be in MB's instead of GB's ( typically when installed the OS ).

1.0 Check Running Containers
```java
docker ps
//docker ps -a ( will list all the containers ever ran on the host )
/*Example:
CONTAINER_ID IMAGE    COMMAD       CREATED STATUS PORT NAME
x12ergykol  fedora:20 "/bin/bash"  1day    Up
*/
```
1.1 Accessing the Container
```java
docker attach x12ergykol
//docker attach CONTAINER_ID
```
1.2 Exit Container ( without killing )
`Ctrl + P + Q`

>> "Running instance of an Docker Image "

#### Docker Commands

1. Docker Run:

`docker run "xxxxx"`

-it : interactive container with a shell.
`docker run -it`

--cpu-shares : controls how many CPU container can have.
`docker run --cpu-shares=256` #gets quarter of cpu.

memory=1g : allocates 1g of memory to the container.
`docker run memory=1g`

-d  : detach container in the background.
`docker run -d ubuntu:14.04.1 /bin/bash -c "ping 8.8.8.8"`
Above command returns the container and can check by executing `docker ps` and verify the details by executing `docker inspect CONTAINER_ID`

attach : attach the container
`docker attach CONTAINER_ID/NAME`
container exist once the command running inside the container exists.

2. Docker Container:

`docker run -it ubuntu:14.04.1 /bin/bash`
running interactive container with ubuntu. If we want to detach from the container then we can execute `Ctl + P + Q`, so container still be running and can verify by executing `docker ps`. If we want to exist from the container then we can execute `Ctl + C`.

`docker stop CONTAINER_ID`
stopping the container outside the interactive terminal.

`docker ps -l`
shows the last container ran.

`docker start CONTAINER_ID`
`docker attach CONTAINER_ID`
the first command will start the container and the second will attach the container so we can resume back to the state where we left of before stopping the container.

`docker restart CONTAINER_ID`
this will restart the container and we can verify by executing `docker ps` and look for the timestamp.

`docker stop CONTAINER_ID`
`docker rm CONTAINER_ID`
we cannot remove running container by executing `rm` and hence we need to stop it. If we need to remove forcefully we can do by doing `docker rm -f CONTAINER_ID`

`docker logs CONTAINER_ID`
executing this from the docker host we can see all the details about the container.

Attach vs Shell:
when we do `docker attach` we are attaching to the standard streams of container PID. In here we cannot execute `shell` command and existing the container shell will exist the container as well. We can verify that by executing `docker ps`. To rescue this and run fully blown shell we do the below,
```java
docker inspect CONTAINER_ID | grep Pid // returns the PID running inside the container.
nsenter -m -u -n -p -i -t PID /bin/bash // namespace enter and list of name spaces in the command along with the PID we got from the first command.
```
but with latest version of docker we can achieve the same using `exec` command as follows,
```java
docker exec -it CONTAINER_ID /bin/bash
```
