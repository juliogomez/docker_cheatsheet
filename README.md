# Docker Cheatsheet for demos

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
	
Run a Nginx web server in TCP port 80, open a shell, install vim and edit the default homepage to see it updates real-time
	
    docker run -d -p 80:80 --name webserver nginx
    docker exec -it webserver /bin/bash
    apt-get update
    apt-get install vim
    cd /usr/share/nginx/html
    vim index.html
    
Run Firefox in a container, by opening a display server, finding out your IP address, and allowing connections from your local machine IP

    open -a XQuartz
    ip=$(ifconfig en0 | grep inet | awk '$1=="inet" {print $2}')
    xhost + $ip										—	
    docker run -d --name firefox -e DISPLAY=$ip:0 -v /tmp/.X11-unix:/tmp/.X11-unix jess/firefox

Stop a container, sending SIGTERM+SIGKILL:

    docker stop <container_id>
    
Delete a container, by deleting the R/W container layers:

    docker rm <container_id>
    
Stop and delete the container:

    docker rm -f <container_id>
    
Delete a container image, by deleting the R/O container layers:

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
     