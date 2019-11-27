# Lab 9 demo: Docker basics

## Intro

First, we will need to install Docker daemon and client tools. There are
multiple ways how to do it, but we will use the simplest possible approach for
the demo:

    sudo apt install docker.io

Note the package name, `docker.io` -- this name is inspired by previous Docker
website name and chosen not to conflict with another package in Debian and
Ubuntu repositories called `docker`; the latter has nothing to do with
containers:

    Package: docker
    Description: System tray for KDE3/GNOME2 docklet applications

    Package: docker.io
    Description: Linux container runtime

Also note that if you decide to install the package from the Docker own
repository as described
[here](https://docs.docker.com/install/linux/docker-ce/ubuntu/) the DEB package
name will be different again: `docker-ce`.

For this demo a bit older Docker version from Ubuntu `universe` repository is
just fine.

Next, let's make sure that Docker container runtime itself is up and running:

    systemctl status docker

... and the Docker client is working:

    docker --version
    docker images


First Docker container
----------------------

Docker team has prepared one very simple container image to verify the Docker
installation. Let's download and run it:

    docker run hello-world

It should print something like

    Unable to find image 'hello-world:latest' locally
    latest: Pulling from library/hello-world
    1b930d010525: Pull complete
    Digest: sha256:4df8ca8a7e309c256d60d7971ea14c27672fc0d10c5f303856d7bc48f8cc17ff
    Status: Downloaded newer image for hello-world:latest

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    ...

Read the output carefully. It explains in details what Docker just did.

Docker Hub is a public Docker registry, in simple words, it is the "package
repository" for Docker where package is the container image.

The action above has left some artifacts on the Docker host, namely the
downloaded container image:

    docker images

Now if you run `docker run hello-world` again it will not re-download the
container but use the local version instead:

    docker run hello-world


Second Docker container
-----------------------

We cannot do much with this `hello-world` app. So let's run another container:

    docker run -it alpine /bin/sh

 - `alpine` here stands for image name; this is an image for container with
    [Alpine Linux](https://alpinelinux.org/), a minimalist Linux distribution
    quite popular for containerized applications;
 - `-it` means `--interactive --tty` which instructs Docker daemon to start the
   interactive session with the container and allocate the pseudo-terminal for
   that; check `docker run --help` for more details;
 - `/bin/sh` is the command we asked Docker to run inside the container

As a result, terminal prompt should change to something like

    / #

This is the shell inside the container! Let's look around there:

    cat /etc/issue
    > Welcome to Alpine Linux 3.10
    > Kernel \r on an \m (\l)

    uname -a
    > Linux 9f0db391d1fd 4.15.0-66-generic #75-Ubuntu SMP Tue Oct 1 05:24:09 UTC 2019 x86_64 Linux

The first command will print the system identification -- the system running in
the container as seen by applications is Alpine Linux v3.10.

The second command will print the Linux kernel version, and you can easily
notice that the kernel (Ubuntu) does not quite match the userspace (Alpine).

This is how operating system virtualization (or containerization) work: both
host and guest systems share the same Linux kernel but their userspaces are
isolated from each other. Let's make some damage in the container:

    # Make sure to run this in the container, *not* on the Docker host!
    rmdir /opt
    exit

Note that `/opt` directory is still present on the host; it was only removed
from the container:

    ls -la /opt

Also note that the Linux kernel version on the host system is the same as was in
the container:

    uname -a


Containerized service example
-----------------------------

Docker containers were designed to run services. To be more precise, every
container is expected to run exactly one process. In previous example the
process was `/bin/sh` and it was running in interactive mode. Once we terminated
the session the process was terminated as well. You can list the terminated
Docker processes with

    docker ps -a

Running processes can be listed with

    docker ps

Note that there are none. So let's start on of the processes in Alpine Linux
container and leave it running in daemon mode. This process could be a `top`
utility:

  docker run -d alpine /usr/bin/top

`-d` here means `--detach`: Docker will launch the container and leave it
running without keeping any interactive session open

The process should now appear in the list of running Docker containers:

    docker ps

You can attach to the running container using its id:

    docker attach 4e411b27a685

You should see the `top` process running. Note that it is the only running
process in this container; although it is technically possible to launch more
than one process in the same container it is still a bit witchcraft, and Docker
discourages these attempts.

You can detach from the process by pressing `Ctrl+C`. The process will be
terminated, and so the container that hosted it.


Some real services
------------------

Now once we know the basics of the Docker container anatomy we can try running
some real services, containerized. One possible candidate is Apache web server:

    docker run -d httpd
    docker ps

It will take some time to download, but in the end you should see the HTTPd
container running and listening on port 80/tcp. But is you check the local
processes listening on port 80 you will find none:

    netstat -lnpt

This is another example of process isolation. By default Docker containers
will not bind to host system ports automatically, this needs to be allowed
explicitly. Let's destroy the running HTTPd contaimer and launch it again, this
time providing the port configuration:

    docker stop bb5f196693ea
    docker run -d -p8081:80 httpd

`-p8081:80` here means `--publish 8081:80` -- this exposes container's port 80
and binds it to host's port 8081.

It should now be possible to communicate with the Apache HTTPd process running
in the container via port 8081 on the Docker host:

    netstat -lnpt

and

    curl http://localhost:8081

should return the page served by Apache.

Let's get it further and deploy the entire Wordpress in Docker container:

    docker run -d -p8082:80 wordpress

That's it! Wordpress should be listening on port 8082 now. If you access this
port with you browser you should see the Wordpress welcome page.


Final notes
-----------

This is a very basic demo on how to handle Docker containers. We did not create
any container images, yet, but only used available, ready made images from
Docker Hub.

Images we used:
 - https://hub.docker.com/_/hello-world
 - https://hub.docker.com/_/alpine
 - https://hub.docker.com/_/httpd
 - https://hub.docker.com/_/wordpress

**SECURITY NOTE ABOUT DOCKERHUB**

Do not blindly trust images from Docker Hub!

If downloading an image, make sure it is an official Docker image!

See also: https://docs.docker.com/docker-hub/official_images

Sometimes it is hard to tell if the image is released buy Docker team or the
product team. Check out these two and carefully inspect the pages:

 - https://hub.docker.com/_/wordpress
 - https://hub.docker.com/r/grafana/grafana

Only one of them is official Docker image as it is built by Docker team.
Another, although also called 'official', is not provided by Docker but by the
product developers. Both teams interpret the word 'official' differently; you
should understand this difference if working with Docker Hub.

**Only use Docker Hub images from the authors whom you trust!**
