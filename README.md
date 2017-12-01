# Cheatsheet for Docker demos

## 1. The Basics

After installing docker you should add your user to the docker group, so that you don’t have to sudo every command:

	sudo usermod -aG docker <user>

Test your installation with:

    docker run hello-world
    docker version
    
Run ubuntu in a container and echo a message:

    docker run ubuntu /bin/echo ‘hello world’

Run an interactive shell in ubuntu:

    docker run -t -i ubuntu /bin/bash
    uname -a
    
Run a script in ubuntu and check its logs:

    docker run -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
   	docker logs -f <docker_NAME>

See complete list of containers (running & exited):

	docker ps -a
	
Run a Nginx web server in TCP port 80, open a shell, install vim and edit the default homepage to see it updates in real-time:
	
    docker run -d -p 80:80 --name webserver nginx
    docker exec -it webserver /bin/bash
        apt-get update && apt-get install vim
        cd /usr/share/nginx/html
        vim index.html
    
Run Firefox in a container, by opening a display server, finding out your IP address, and allowing connections from your local machine IP:

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
    
## 2. Build your own container

### 2.1 Manually

Run Ubuntu container and create an app inside:

    docker run -it --name testapp ubuntu /bin/bash
        echo -e "#! /bin/bash\necho It works!" > myapp.sh
        chmod +x myapp.sh
        exit

Save container layers as R/O, and create a new R/W layer (new image):

    docker commit testapp <your_docker_id>/sinatra
    
Delete the container:

    docker rm testapp
    
Run the app inside the container:

    docker run --name testapp <your_docker_id>/sinatra /myapp.sh
    
Set executing myapp.sh as default when running <your_docker_id>/sinatra

    docker commit --change='CMD [“/myapp.sh”]' testapp <your_docker_id>/sinatra
    
Run a container from the new docker without specifying the app inside the container:

    docker rm testapp
    docker run --rm <your_docker_id>/sinatra
    
### 2.2 With Dockerfile

Create 'myapp.sh' in your local environment, with the following content:

	#! /bin/bash
	echo "I am a cow!" | /usr/games/cowsay | /usr/games/lolcat -f 
			
Assign permissions to execute it:

    chmod +x myapp.sh
    
Create a Dockerfile in your local environment, with the following content:

	FROM ubuntu
	RUN apt-get update && apt-get install -y cowsay lolcat && rm -rf /var/lib/apt/lists/*	—   delete caches
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
    
You can rename a container by changing its tag:

    docker tag <your_docker_id>/myapp <your_docker_id>/cowapp:ver2
    
In order to upload your container to a different registry you need to build the image with the full registry name:

    docker build -t <full_domain_name>/<your_id>/myapp .
    docker push <full_domain_name>/<your_id>/myapp
    
## 4. Basic Networking

There is an eth0 for the container and a peer veth in the host, with a virtual bridge from host to container. iptables make sure that traffic only flows between containers in the same bridge  (default docker0).

Check existing networks:

    docker network ls
    
'bridge' maps to docker0 (shown when you run ifconfig in the host)

'host' maps container to host (not recommended)

'none' provides no connectivity

Let's create a container that responds to HTTP requests with its own IP address:

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

### 4.1 Linking containers with /etc/hosts (legacy)

This way of linking containers is static and restarting containers will need linked containers to restart as well, so that port numbers are updated in /etc/hosts

Run one container and store its IP address in a host variable:

    docker run -d --name myapp <your_docker_id>/containerip
    TESTING_IP=$(docker inspect --format "{{ .NetworkSettings.IPAddress }}" myapp)
    echo $TESTING_IP

Run an additional container and pass it the IP address of the first container to ping it:

    docker run --rm -it -e TESTING_IP=$TESTING_IP your_docker_id>/containerip /bin/bash
    ping $TESTING_IP -c 2
    
### 4.2 Linking containers with links

Run one container:

    docker run -d --name myapp <your_docker_id>/containerip
    
Run an additional container and create a link/alias to the first one to ping it:

    docker run --rm -it --link myapp1:container1 <your_docker_id>/containerip /bin/bash
        ping container1 -c 2
    
This link option updates /etc/hosts in the new container with an entry for the linked container IP. But if that container restarts then this /etc/hosts is not updated and will keep pointing to the old IP

### 4.3 Linking containers with user networks

User-defined bridges that automatically discover containers and provide DNS (not /etc/hosts at all)

Create a new docker network:

    docker network create mynet

Create two containers connected to that network:
    
    docker run -d --net=mynet --name myapp1 <your_docker_id>/containerip
    docker run -d --net=mynet --name myapp2 <your_docker_id>/containerip
    
Connect to the first container and try pinging the second one using its name:
    
    docker exec -it myapp2 /bin/bash
        ping myapp1 -c 2
        
## 5. Docker Compose

This is a tool to orchestrate a number of containers inside a service, by mean of a YML file.

### 5.1 Example 1 - Connectivity

Let's run two containers and verify connectivity between them using names.

Create file docker-compose.yml:

    myapp:
	  image: <your_docker_id>/containerip
	someclient
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

## 6. Storage

### 6.1 Sharing a directory between host and a container

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
    
### 6.2 Creating a Docker Volume to share between containers

Run a container and create a volume:

    docker run -d --name myapp -v /data ubuntu
    
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
    docker volume ls
