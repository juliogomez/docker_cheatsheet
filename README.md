# Docker - From zero to Hero

  * [1. The Basics](#1-the-basics)
  * [2. Build your own image](#2-build-your-own-image)
    + [2.1 Manually](#21-manually)
    + [2.2 With Dockerfile](#22-with-dockerfile)
  * [3. Dockerhub and other registries](#3-dockerhub-and-other-registries)
  * [4. Basic Networking](#4-basic-networking)
    + [4.1 Linking containers with variables](#41-linking-containers-with-variables)
    + [4.2 Linking containers with links](#42-linking-containers-with-links)
    + [4.3 Linking containers with user networks](#43-linking-containers-with-user-networks)
  * [5. Docker Compose](#5-docker-compose)
    + [5.1 Example 1 - Connectivity](#51-example-1---connectivity)
    + [5.2 Example 2 - Load balance](#52-example-2---load-balance)
    + [5.3 Example 3 - WordPress](#53-example-3---wordpress)
  * [6. Storage](#6-storage)
    + [6.1 Sharing a directory between host and container](#61-sharing-a-directory-between-host-and-container)
    + [6.2 Sharing a Docker volume between containers](#62-sharing-a-docker-volume-between-containers)
  * [7. Docker Swarm](#7-docker-swarm)
    + [7.1 Single-service stack](#71-single-service-stack)
    + [7.2 Multi-service stack](#72-multi-service-stack)
    + [7.3 Overlay networking](#73-overlay-networking)

## 1. The Basics

After [downloading and installing docker CE](https://www.docker.com/community-edition#/download) you should add your user to the docker group, so that you don’t have to sudo every command:

	sudo usermod -aG docker <user>

Test your installation with:

    docker run hello-world
    docker version
    
Run ubuntu in a container and echo a message:

    docker run ubuntu /bin/echo ‘hello world’

Run an interactive shell in ubuntu:

    docker run -t -i ubuntu /bin/bash
        ls
        cat /etc/issue
        ps aux
        exit
    
Run a script in ubuntu and check its logs:

    docker run --name script -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
   	docker logs -f script

See complete list of containers (running & exited):

	docker ps -a
	
Run a Nginx web server (in detached mode so that it runs in the background) in TCP port 80, open a shell, install vim and edit the default homepage to see it updates in real-time:
	
    docker run -d -p 80:80 --name webserver nginx
    docker exec -it webserver /bin/bash
        apt-get update && apt-get install vim
        cd /usr/share/nginx/html
        vim index.html

You may see the top processes running inside your container with:

    docker top webserver
    
(Only for Mac users) Run Firefox in a container, by opening a display server, finding out your IP address, and allowing connections from your local machine IP:

    open -a XQuartz
    ip=$(ifconfig en0 | grep inet | awk '$1=="inet" {print $2}')
    xhost + $ip										—	
    docker run -d --name firefox -e DISPLAY=$ip:0 -v /tmp/.X11-unix:/tmp/.X11-unix jess/firefox

Stop a container, sending SIGTERM+SIGKILL:

    docker stop <container_id>
    
Delete a container, by deleting its R/W container layers:

    docker rm <container_id>
    
Stop and delete the container:

    docker rm -f <container_id>
    
Delete a container image, by deleting its R/O container layers:

    docker rmi <container_image_name>
    
Delete all containers:

    docker rm $(docker ps -aq)
    
Delete all stopped/exited containers:

    docker rm $(docker ps -aqf status=exited)
    
List all local images:

    docker images
    
Pull a specific image from a remote repo:

    docker pull <user/repo_name:tag>
    
## 2. Build your own image

### 2.1 Manually

Run Ubuntu container and create an app inside:

    docker run -it --name testapp ubuntu /bin/bash
        echo -e "#! /bin/bash\necho It works!" > myapp.sh
        chmod +x myapp.sh
        exit

Check the differences between your container and the base image:

    docker diff testapp

Save container layers as R/O, and create a new R/W layer (new image):

    docker commit testapp <your_docker_id>/<your_app_name>
    
Delete the container:

    docker rm testapp
    
Run the app inside the container:

    docker run --name testapp <your_docker_id>/<your_app_name> /myapp.sh
    
Set executing myapp.sh as default when running <your_docker_id>/<your_app_name>

    docker commit --change='CMD [“/myapp.sh”]' testapp <your_docker_id>/<your_app_name>
    
Run a container from the new docker without specifying the app inside the container:

    docker rm testapp
    docker run --rm <your_docker_id>/<your_app_name>
    
### 2.2 With Dockerfile

Create 'myapp.sh' in your local environment, with the following content:

	#! /bin/bash
	echo "I am a cow!" | /usr/games/cowsay | /usr/games/lolcat -f 
			
Assign permissions to execute it:

    chmod +x myapp.sh
    
Create a Dockerfile in your local environment, with the following content:

	FROM ubuntu
	RUN apt-get update && apt-get install -y cowsay lolcat && rm -rf /var/lib/apt/lists/*
	COPY myapp.sh /myapp/myapp.sh
	CMD ["/myapp/myapp.sh"]

Build and run your app:

    docker build -t <your_docker_id>/myapp .
    docker run --rm <your_docker_id>/myapp
    
See layers created for each command in Dockerfile:
    
    docker history <your_docker_id>/myapp | head -n 4
    
## 3. Dockerhub and other registries
    
Upload it to the default dockerhub, delete the local copy and run it again from there:

    docker login
    docker push <your_docker_id>/myapp
    docker rmi <your_docker_id>/myapp
    docker run --rm <your_docker_id>/myapp
    
You can rename an image by changing its tag:

    docker tag <your_docker_id>/myapp <your_docker_id>/cowapp:ver2
    
In order to upload your container to a different registry you need to build the image with the full registry name:

    docker build -t <full_domain_name>/<your_id>/myapp .
    docker push <full_domain_name>/<your_id>/myapp
    
## 4. Basic Networking

There is an eth0 for the container and a peer veth in the host, with a virtual bridge from host to container. iptables make sure that traffic only flows between containers in the same bridge  (default docker0).

Check existing local container networks and their associated driver:

    docker network ls
    
'bridge' is the default one where all new containers will be connected to, if not specified otherwise, and maps to docker0 (shown when you run ifconfig in the host)

'host' maps container to host (not recommended)

'none' provides no connectivity

Every network created with the 'bridge' driver is based on a virtual bridge in Linux. You may use the 'brctl' to list virtual bridges in your host:

    brctl show
    
You may also see details about docker0 virtual bridge with:

    ip a

Initially 'brctl' shows that there are no interfaces connected to 'docker0', but let's run the following container and check how it is automatically connected by default:

    docker run -dt --name test ubuntu sleep infinity
    brctl show 
    docker network inspect bridge
    
You may check connectivity by pinging from your host to the IP address of the container (shown in the last command).    

You may also check connectivity to the outside world from your container (but first you need to install the 'ping' utility):

    docker exec -it test /bin/bash
        apt-get update && apt-get install -y iputils-ping
        ping -c5 www.github.com
        exit
        
        
    
Before we start digging into the networking demo, let's first create a container that responds to HTTP requests with its own IP address:

1.- In your local environment create the following directory structure:

    mkdir ./www/
    mkdir ./www/cgi-bin
    
2.- Create the script to find out container's IP address

    vi ./www/cgi-bin/ip
        #! /bin/sh
        echo
        echo "Container IP: $(ifconfig eth0 | awk '/inet addr/{print substr($2,6)}')"
    
3.- Create a Dockerfile with the following content:

    FROM alpine:3.3
	ADD www /www
	RUN apk add --no-cache bash curl && chmod -R 700 /www/cgi-bin
	EXPOSE 8000
	CMD ["/bin/busybox", "httpd", "-f", "-h", "/www", "-p", "8000"] 

4.- Build the image:

    docker build -t <your_docker_id>/containerip .
    
5.- Run the container that offers its service in TCP port 8000 (but not available from the host!):

    docker run -d --name myapp <your_docker_id>/containerip
    
6.- Connect to the container, check interface eth0 (the one connected to docker0 virtual bridge), and the IP returned by the HTTP server:

    docker exec -it myapp /bin/bash
        ifconfig
        curl localhost:8000/cgi-bin/ip
        exit
    docker rm -f myapp

7.- Let's make it now available from the host by mapping ports, and test that it shows the internal IP address of the container:

    docker run -d -p 8000:8000 --name myapp <your_docker_id>/containerip
    curl localhost:8000/cgi-bin/ip
    
8.- Check the exposed port for the image and container:

    docker inspect --format "{{ .ContainerConfig.ExposedPorts }}" <your_docker_id>/containerip
    docker port myapp
    
9.- Assign the port in the host to a variable in your local environment, and run the request again:

    PORT=$(docker port myapp | cut -d ":" -f 2)
    curl "localhost:$PORT/cgi-bin/ip"

### 4.1 Linking containers with variables

This way of linking containers is static and restarting containers will need variables to be updated:

Run one container and store its IP address in a host variable:

    docker run -d --name myapp <your_docker_id>/containerip
    TESTING_IP=$(docker inspect --format "{{ .NetworkSettings.IPAddress }}" myapp)
    echo $TESTING_IP

You may inspect the network and see the IP address assigned to your container:

    docker network inspect bridge 

Run an additional container and pass it the IP address of the first container to ping it:

    docker run --rm -it -e TESTING_IP=$TESTING_IP <your_docker_id>/containerip /bin/bash
        ping $TESTING_IP -c 2
    
### 4.2 Linking containers with links

Run one container:

    docker run -d --name myapp <your_docker_id>/containerip
    
Run an additional container and create a link/alias to the first one to ping it:

    docker run --rm -it --link myapp:container1 <your_docker_id>/containerip /bin/bash
        ping container1 -c 2
    
This link option updates /etc/hosts in the new container with an entry for the linked container IP. But if that container restarts then this /etc/hosts is not updated and will keep pointing to the old IP.

    cat /etc/hosts

### 4.3 Linking containers with user networks

User-defined bridges that automatically discover containers and provide DNS (not /etc/hosts at all)

Create a new docker network:

    docker network create mynet

Create two containers connected to that network:
    
    docker run -d --net=mynet --name myapp1 <your_docker_id>/containerip
    docker run -d --net=mynet --name myapp2 <your_docker_id>/containerip
    
You may inspect the network and see the IP addresses assigned to your containers:

    docker network inspect mynet 
    
Connect to the first container and try pinging the second one using its name:
    
    docker exec -it myapp2 /bin/bash
        ping myapp1 -c 2

You can now manually disconnect one of the containers from the network, and inspect the network again:

    docker network disconnect mynet myapp2
    docker network inspect mynet

Delete your network:

    docker network rm mynet
    
## 5. Docker Compose

This is a tool to orchestrate a number of containers inside a service, by mean of a YML file.

### 5.1 Example 1 - Connectivity

Let's run two containers and verify connectivity between them using names.

Create file docker-compose.yml:

    myapp:
	  image: <your_docker_id>/containerip
	someclient:
	  image: <your_docker_id>/containerip
	  container_name: someclient
	  command: sleep 500
	  links:
	  - myapp

Start the two defined containers in detached mode:

    docker-compose up -d
    
Check that the container for 'myapp' has been assigned a pseudo-random name, and 'someclient' is executing a different command.

    dockers ps -a
    
Connect to 'someclient' and ping 'myapp' by its name, defined in the yml file:

    docker exec -it someclient /bin/bash
        ping myapp -c 2
        
There is connectivity between the two containers because of the 'links' definition.

Stop both containers:

    docker-compose kill
    
Remove both containers:

    docker-compose rm -f
    
### 5.2 Example 2 - Load balance

'haproxy' is a load-balancer serving in port 80 and redirecting traffic to containers in 'myapp'

Create file docker-compose.yml:

    myapp:
      image: <your_docker_id>/containerip
    haproxy:
      image: dockercloud/haproxy
      container_name: haproxy
      links:
        - myapp
      ports:
        - 80:80

Start two containers for 'myapp' and one for 'haproxy', all in detached mode:

    docker-compose up --scale myapp=2 -d
    
Check port mapping for 'haproxy' (host 80 to haproxy 80) and 'containerip' using a totally different port (8000)

    docker-compose ps
    
Obtain 'myapp' containers IP addresses by requesting 'haproxy' for it several times:

    curl localhost:80/cgi-bin/ip

Stop all containers:

    docker-compose kill
    
Remove all containers:

    docker-compose rm -f

### 5.3 Example 3 - WordPress

WordPress deployment with two containers defined in this docker-compose.yml file:

    wordpress:
      image: wordpress
      links:
        - db:mysql
      ports:
        - 8080:80
    
    db:
      image: mariadb
      environment:
        MYSQL_ROOT_PASSWORD: Nbv12345!

Start the two defined containers in detached mode:

    docker-compose up -d
    
Browse to your new WordPress installation at localhost:8080

Stop both containers:

    docker-compose kill
    
Remove both containers:

    docker-compose rm -f
    
## 6. Storage

### 6.1 Sharing a directory between host and container

How to mount a local directory into a container:

    mkdir ~/temp
    cd ~/temp
    echo "Hello" > testfile
    docker run --rm -it -v ~/temp:/data ubuntu /bin/bash

Check you can see the content of that directory, and update the file inside it:

    cat /data/testfile
    cd /data/
    echo “appended from $(hostname)” >> testfile
    exit
    
Check in your local host the file has been modified from inside the container, even though it has now been terminated:

    cat ~/temp/testfile
    
### 6.2 Sharing a Docker volume between containers

Run a container and create a volume:

    docker run -itd --name myapp -v /data ubuntu
    
Connect to it and update a file:

    docker exec -it myapp /bin/bash
        cd /data
        echo "appended from $(hostname)" >> volfile
        exit

Create another container, map the volume from the first container, and check the hostname appended is the one from the first container:

    docker run --rm -it --volumes-from myapp ubuntu /bin/bash
        cat /data/volfile
        hostname
        
Check the names of existing docker volumes:

    docker volume ls
    
They are randomly named, so better to specify a volume name when creating them:

    docker run --rm -it -v myvol:/data ubuntu /bin/bash
        exit
    docker volume ls

## 7. Docker Swarm

Swarm is Docker's orchestrator for containers, and it is composed by two types of nodes: managers and workers. For this example let's create 2 VMs with VirtualBox to work as nodes in your local environment:

    docker-machine create --driver virtualbox myvm1
    docker-machine create --driver virtualbox myvm2
    
List the machines and get their IPs:

    docker-machine ls
    
Configure the first VM as a manager node:

    docker-machine ssh myvm1 "docker swarm init --advertise-addr <myvm1 ip>"
    
And use the resulting command in the second VM, so that it becomes a worker node:

    docker-machine ssh myvm2 "docker swarm join --token <token> <ip>:2377"
    
Check the nodes in your swarm, by querying the manager node:

    docker-machine ssh myvm1 "docker node ls"
    
Configure your shell to run commands against the manager node:

    docker-machine env myvm1
    eval $(docker-machine env myvm1)    

Check that the active machine is now 'myvm1' by the asterisk next to it:

    docker-machine ls
    
### 7.1 Single-service stack

Create a docker-compose.yml file for a new service, specifying you want to deploy 3 copies of your container, HW requirements, ports mapping and network to use (by default load-balanced to all containers in your 'web' service):

    version: "3"
    services:
      web:
        image: <your_docker_id>/containerip
        deploy:
          replicas: 3
          resources:
            limits:
              cpus: "0.1"
              memory: 50M
          restart_policy:
            condition: on-failure
        ports:
          - "80:8000"
        networks:
          - webnet
    networks:
      webnet:

Deploy the service in your swarm:

    docker stack deploy -c docker-compose.yml getstartedlab
    
Check your new service ID, number of replicas, image name and exposed ports:

    docker service ls
    
List the containers (aka 'tasks') for your service and how they are distributed across the VMs:

    docker service ps getstartedlab_web

Access the service via the IP of any VM several times, so that you check how IP changes when different containers serve different requests:

    curl <VMx_IP>/cgi-bin/ip
    
The network you created is shared and load-balanced by default, all swarm nodes participate in an ingress routing mesh, and reserve the same service port in all nodes.

Change the number of replicas in the docker-compose.yml file for your service and re-deploy:

    docker stack deploy -c docker-compose.yml getstartedlab
    
Take the stack down:

    docker stack rm getstartedlab_web
    
### 7.2 Multi-service stack

Let's add a second service to your previous docker-compose.yml file, so that we create a new stack with two services. The file will request to deploy 5 copies of 'containerip' and 1 copy of 'visualizer'. Apart from the previous parameters it will also map Docker's socket to have access to the list of running containers, and a placement constraint to deploy it only in the manager node:

    version: "3"
    services:
      [...]
      visualizer:
        image: dockersamples/visualizer:stable
        ports:
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock"
        deploy:
          placement:
            constraints: [node.role == manager]
        networks:
          - webnet
    networks:
      webnet:

Update the stack in your swarm:

    docker stack deploy -c docker-compose.yml getstartedlab
    
Verify both services are now running in your stack:

    docker service ls
    
Point your browser to the IP of any of your VMs (VMx_IP:8080) and see how your containers have been distributed between both VMs:

You may check the allocation is correct by running:

    docker stack ps getstartedlab

When you are finished you may remove the stack:

     docker stack rm getstartedlab_web

### 7.3 Overlay networking

As you may remember we have already discussed single-host networking in past chapters. Now is the time to investigate multi-host networking with overlay *networks*.

From *manager* let's create a new overlay network called *overnet* by specifying the driver (`-d`) to use as *overlay*:

```
docker network create -d overlay overnet
docker network ls
```


If you run `docker network ls` from your worker node you will not see *overnet*. The reason is there are no service containers running in this new network yet.

Let's now create a new service from the manager node:

```
docker service create --name myservice --network overnet --replicas 2 ubuntu sleep infinity
```

Check until the service and both replicas are up with `docker service ls`.

Now verify that there is one replica running in each of your Swarm nodes:

```
docker service ps myservice
```


If you check the networks again in your worker node with `docker network ls`, *overnet* will be there. The reason is that now one of the container using *overnet* resides in your worker node.


Please use `docker network inspect overnet` in your worker node, and write down the IP address of the container running in that worker node.


Now go to your manager node and list your running containers with `docker ps`, copy its ID and connect to it. Then install *ping* utility, and ping the IP address of the container running in the worker node (the one you noted in the previous step).

```
docker attach -it <container ID> /bin/bash
  apt-get update && apt-get install -y iputils-ping
  ping -c5 <worker2_container IP>
```

Congrats!! You have just verified the connectivity between containers in different hosts, via an overlay network.

Let's now learn about Service Discovery.

From inside the container where we were in last step, please run `cat /etc/resolv.conf` and look for the *nameserver* entry. That IP address belongs to a DNS resolver embedded in all Docker containers (ie. 127.0.0.11:53).

You may go ahead and ping 'myservice' from inside the container, with `ping -c2 myservice`. Please note down the IP address answering your ping requests.

Now go to your manager node, and from that host run `docker service inspect myservice`. Towards the bottom of the output you will find the IP address of that service Virtual IP. You will see it is the same as the IP answering your ping requests in the previous step.

Let's clean up by removing the service from your manager node, with `docker service rm myservice`.




Finally you can remove the swarm from your system:

    docker-machine ssh myvm2 "docker swarm leave"
    docker-machine ssh myvm1 "docker swarm leave --force"
    
Disconnect the shell from 'myvm1' environment:

    eval $(docker-machine env -u)
    
Please note if you restart your local host you will need to restart your stopped VMs:

    docker-machine ls
    docker-machine start myvm1
    docker-machine start myvm2
    
Finally you may remove both VMs:

    docker-machine rm myvm2
    docker-machine rm myvm1