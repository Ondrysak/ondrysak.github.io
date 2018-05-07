Title: Docker is love!
Date: 2018-05-07 09:01
Modified: 2018-05-07 09:01
Category: Technology
Tags: technology, docker
Slug: Docker
Authors: Ondřej Naňka
Summary: What is Docker, what does it do?


Hello fellaz, if u did not hear about Docker before now its the time to change it!

Virtualization is cool, Docker is cool++. 

## Dockerfile

Docker can build images automatically by reading the instructions from a Dockerfile. A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image. Using docker build users can create an automated build that executes several command-line instructions in succession. See example of Dockerfile to run flask app using gunicorn WSGI below.

```Dockerfile
FROM alpine:3.6
ENV LANG C.UTF-8
ENV FLASK_APP=pyrest
ENV FLASK_DEBUG=true
RUN apk add --no-cache py-gunicorn python curl && \
    python -m ensurepip && \
    rm -r /usr/lib/python*/ensurepip && \
    pip install --upgrade pip setuptools && \
    rm -r /root/.cache
COPY . /app
WORKDIR /app
RUN pip install .
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "pyrest:app"]
```

#FROM 
The FROM instruction initializes a new build stage and sets the Base Image for subsequent instructions. As such, a valid Dockerfile must start with a FROM instruction. The image can be any valid image – it is especially easy to start by pulling an image from the Public Repositories.
#ENV
The ENV instruction sets the environment variable <key> to the value <value>. This value will be in the environment for all subsequent instructions in the build stage and can be replaced inline in many as well.
#COPY
The COPY instruction copies new files or directories from <src> and adds them to the filesystem of the container at the path <dest>.
#WORKDIR
The WORKDIR instruction sets the working directory for any RUN, CMD, ENTRYPOINT, COPY and ADD instructions that follow it in the Dockerfile. If the WORKDIR doesn’t exist, it will be created even if it’s not used in any subsequent Dockerfile instruction.
The WORKDIR instruction can be used multiple times in a Dockerfile. If a relative path is provided, it will be relative to the path of the previous WORKDIR instruction. For example:
#RUN
The RUN instruction will execute any commands in a new layer on top of the current image and commit the results. The resulting committed image will be used for the next step in the Dockerfile.
#CMD

There can only be one CMD instruction in a Dockerfile. If you list more than one CMD then only the last CMD will take effect.

The main purpose of a CMD is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an ENTRYPOINT instruction as well.

## Docker commands

Lets list some basic commands you can play around with, I will talk about them more in depth later.

```bash
docker build
docker images 
docker run
docker stop
docker exec
docker ps
docker rm
docker inspect
docker pull 
docker push
```

#docker build

Is used for building a image

#docker images

The default docker images will show all top level images, their repository and tags, and their size.

Docker images have intermediate layers that increase reusability, decrease disk usage, and speed up docker build by allowing each step to be cached. These intermediate layers are not shown by default.

#docker run

The docker run command first creates a writeable container layer over the specified image, and then starts it using the specified command. That is, docker run is equivalent to the API /containers/create then /containers/(id)/start. A stopped container can be restarted with all its previous changes intact using docker start. See docker ps -a to view a list of all containers.

#docker stop

The main process inside the container will receive SIGTERM, and after a grace period, SIGKILL.

#docker exec

The docker exec command runs a new command in a running container.

The command started using docker exec only runs while the container’s primary process (PID 1) is running, and it is not restarted if the container is restarted.

COMMAND will run in the default directory of the container. If the underlying image has a custom directory specified with the WORKDIR directive in its Dockerfile, this will be used instead.

#docker ps

List containers

#docker rm

Remove one or more containers

#docker inspect 

Docker inspect provides detailed information on constructs controlled by Docker.

By default, docker inspect will render results in a JSON array.

#docker pull

Most of your images will be created on top of a base image from the Docker Hub registry.

Docker Hub contains many pre-built images that you can pull and try without needing to define and configure your own.

To download a particular image, or set of images (i.e., a repository), use docker pull.

#docker push

Use docker push to share your images to the Docker Hub registry or to a self-hosted one.

[Docker docs](https://docs.docker.com/install/overview/)

