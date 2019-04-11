---
title: "Docker"
teaching: 120
exercises: 10
questions:
- "How can one use Docker?"
- "What is the difference between Docker image and container?"
objectives:
- "Understand the concept of layers of a Docker image."
- "Run Docker container"
- "Build software container image using Docker"
keypoints:
- "To minimize the container image size, do not include development tools and other build artifacts"
- "Docker images are 'layered'"
- "'Layered' images have build time advantages"
- "Multi-stage Docker builds are a powerful technique to minimize image size."
---

## Docker 101

First and foremost, let's check that our Docker Engine is up and running. Open your terminal and
execute:

~~~
$ docker version
~~~
{: .language-bash}

~~~
Client: Docker Engine - Community
 Version:           18.09.2
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        6247962
 Built:             Sun Feb 10 04:12:39 2019
 OS/Arch:           darwin/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.2
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.6
  Git commit:       6247962
  Built:            Sun Feb 10 04:13:06 2019
  OS/Arch:          linux/amd64
  Experimental:     true
~~~
{: .output}

You should see output similar to the one above. `docker version` reports versions of two components
of the Docker Engine: Client and Server. If Docker Engine is not running, `docker version` would
still report the version of the Client, but then notify us that the Server is not running with the
following message:

~~~
Error response from daemon: Bad response from Docker engine
~~~
{: .error}

If you see the above error message, please let the instructors know about it as soon as possible
because it is important to fix it before we go any further.

> ## Docker on Linux
>
> If you're working with Linux, you have to prefix all `docker` commands with `sudo` because Docker
> requires administrative privileges. This means that you haven't followed optional steps of the
> Linux installation guide. If you would like to avoid having to use `sudo` with every Docker
> command, follow these steps:
>
> ~~~
> sudo groupadd docker         # Create 'docker' group
> sudo gpasswd -a $USER docker # Add current user to the group
> sudo service docker restart  # Restart Docker daemon
> exit                         # Log out
> ~~~
> {: .language-bash}
{: .callout}


`docker version` is an example of a "Docker command". Every Docker command begins with the word
`docker` and is followed by a verb or a noun (such as `version`). To get a list of all commands
provided by Docker, execute `docker` without any arguments:

~~~
$ docker
~~~
{: .language-bash}

~~~

Usage:	docker [OPTIONS] COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default "/Users/mbelkin/.docker")
  -D, --debug              Enable debug mode
  -H, --host list          Daemon socket(s) to connect to
  -l, --log-level string   Set the logging level ("debug"|"info"|"warn"|"error"|"fatal") (default "info")
      --tls                Use TLS; implied by --tlsverify
      --tlscacert string   Trust certs signed only by this CA (default "/Users/mbelkin/.docker/ca.pem")
      --tlscert string     Path to TLS certificate file (default "/Users/mbelkin/.docker/cert.pem")
      --tlskey string      Path to TLS key file (default "/Users/mbelkin/.docker/key.pem")
      --tlsverify          Use TLS and verify the remote
  -v, --version            Print version information and quit

Management Commands:
  builder     Manage builds
  checkpoint  Manage checkpoints
  config      Manage Docker configs
  container   Manage containers
  image       Manage images
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  deploy      Deploy a new stack or update an existing stack
  diff        Inspect changes to files or directories on a container's filesystem
  events      Get real time events from the server
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
  history     Show the history of an image
  images      List images
  import      Import the contents from a tarball to create a filesystem image
  info        Display system-wide information
  inspect     Return low-level information on Docker objects
  kill        Kill one or more running containers
  load        Load an image from a tar archive or STDIN
  login       Log in to a Docker registry
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  search      Search the Docker Hub for images
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes

Run 'docker COMMAND --help' for more information on a command.
~~~
{: .output}

Don't be frightened by the staggering number of commands available. All you have to remember at this
point is that if you forget the name of the command, you can quickly find it by simply executing
`docker` in your command line.


## Hello, Docker!

"Hello, World!" is the simplest program used in programming classes to introduce novices to
the programming languages. Here is an example of such a "program"
~~~
$ docker run busybox echo Hello, World!
~~~
{: .language-bash}
~~~
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
fc1a6b909f82: Pull complete
Digest: sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd
Status: Downloaded newer image for busybox:latest
Hello, World!
~~~
{: .output}

Congratulations! You have just executed containerized application!

Let's break down what has just happened here.
We used `docker run` command to execute a simple `echo Hello, World!` command in the `busybox`
container. But we didn't have this container on our system before! So, what Docker did is it looked for
the `busybox`
image locally, failed to find it, contacted central Docker repository of images called
"Docker Hub" (<https://hub.docker.com>) to see if `busybox` image exists there, received a positive response, downloaded
(pulled) the image, and then executed the `echo` command inside of the container.

So, what are these "image", "container", "Docker Hub" that we mentioned above? And where did that
`:latest` in `Status: Downloaded newer image for busybox:latest` come from?

1. An "image" is a static archive of one or more applications and all auxiliary tools required by
   the application(s).
2. A container is a process created by executing an application within the image. It exists only for
   as long as the application contained in the image runs.
3. Docker Hub is an online repository of images that are created by software developers and
   companies (NVIDIA, Canonical, etc) that would like other Docker users to be able to use their
   software in Docker containers.
4. `latest` is called "tag". Tags are used to identify different versions of the same image.
   We did not specify which `tag` we'd like to use when we executed `docker run` command, so
   Docker used the default value `latest`.

Let's now execute the same command again.

~~~
$ docker run busybox echo Hello, World!
~~~
{: .language-bash}

~~~
Hello, World!
~~~
{: .output}

This time, Docker was able to find the image locally so it did not have to pull it from Docker Hub!

Now, let's try using a different image:
~~~
$ docker run hello-world
~~~
{: .language-bash}
~~~
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete
Digest: sha256:2557e3c07ed1e38f26e389462d03ed943586f744621577a99efb77324b0fe535
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
~~~
{: .output}

Docker was not able to find the `hello-world` image locally and pulled it from the Docker Hub.
This time we did not specify the command that we wanted to execute within the image but it still
produced some output! This is an example of the default command that is executed when we
`docker run` an image.  The default command is `/bin/sh`.

> ## Practice
>
> Execute the following images:
>
> - hello-seattle
> - hola-mundo
>
> > ## Solution
> > ~~~
> > docker run hello-seattle
> > docker run hola-mundo
> > ~~~
> {: .solution}
{: .challenge}


## Management commands

When Docker pulled images from Docker Hub, it stored them on our computer. To get a list of images
that we currently have on the machine, let's remind ourselves of the "Management commands" provided
by Docker:

~~~
docker
~~~
{: .language-bash}
~~~
...

Management Commands:
  builder     Manage builds
  checkpoint  Manage checkpoints
  config      Manage Docker configs
  container   Manage containers
  image       Manage images
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes

...
~~~
{: .output}

Note the `image` command and execute:

~~~
docker image ls
~~~
{: .language-bash}

You should see a list of images that we've just pulled.
Now, let's try deleting some of them:

~~~
docker image rm hello-seattle
~~~
{: .language-bash}

~~~
Error response from daemon: conflict: unable to remove repository reference "hello-seattle" (must force) - container 61bb4bb62164 is using its referenced image 515d5e66f68a
~~~
{: .error}

The reason it failed to delete the image is simple: when the container finished executing what it
was supposed to execute, it was stopped. The container, however, still continues to "use" image even
if it is used. Let's have a look at the list of containers that we have on the system:

~~~
docker container ls
~~~
{: .language-bash}

Now, using `docker container ls` we don't see stopped containers. To see them, we need to add `-a`
(or `--all` flag):

~~~
docker container ls -a
~~~
{: .language-bash}

To remove `hello-seattle` image, we need to remove the container that is using it.
To delete a container, we can use `docker container rm` command followed by a NAME or CONTAINER_ID:

~~~
docker container rm frosty_brown
docker image rm hello-seattle
~~~
{: .language-bash}

Name of the container is a random (and unique) name assigned by Docker to every container on
creating. We can specify the name we'd like to assign to the container when we run it using the
`--name` flag of the `run` command:

~~~
docker run --name my_container hello-seattle
docker container rm my_container
docker image rm hello-seattle
~~~
{: .language-bash}

> ## Pruning containers
>
> Instead of deleting stopped containers individually, we can delete all stopped containers using
> `container prune` command:
> ~~~
> docker container prune
> ~~~
> {: .language-bash}
> ~~~
> WARNING! This will remove all stopped containers.
> Are you sure you want to continue? [y/N]
> ~~~
> {: .output}
{: .callout}

## Docker images

Let's now explore the images that we can execute. Open up your browser and navigate to
<https://hub.docker.com>. At the top of the page type in "Ubuntu" in the search box and hit
<kbd>Return</kbd>. Now click on the top most item that says "Ubuntu" -- you should end up on
<https://hub.docker.com/_/ubuntu>. The "Description" page has a section titled
"Supported tags and respective `Dockerfile` links" that lists all the tags that exist for this
image. Let's now try using images with different tags.

~~~
docker run --name ubuntu16 ubuntu:16.04  cat /etc/lsb-release
~~~
{: .language-bash}
~~~
Unable to find image 'ubuntu:16.04' locally
16.04: Pulling from library/ubuntu
34667c7e4631: Already exists
d18d76a881a4: Already exists
119c7358fbfc: Already exists
2aaf13f3eff0: Already exists
Digest: sha256:58d0da8bc2f434983c6ca4713b08be00ff5586eb5cdff47bcde4b2e88fd40f88
Status: Downloaded newer image for ubuntu:16.04
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.6 LTS"
~~~
{: .output}

~~~
docker run --name ubuntu18 ubuntu:18.04  cat /etc/lsb-release
~~~
{: .language-bash}
~~~
Unable to find image 'ubuntu:18.04' locally
18.04: Pulling from library/ubuntu
898c46f3b1a1: Pull complete
63366dfa0a50: Pull complete
041d4cd74a92: Pull complete
6e1bee0f8701: Pull complete
Digest: sha256:017eef0b616011647b269b5c65826e2e2ebddbe5d1f8c1e56b3599fb14fabec8
Status: Downloaded newer image for ubuntu:18.04
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=18.04
DISTRIB_CODENAME=bionic
DISTRIB_DESCRIPTION="Ubuntu 18.04.2 LTS"
~~~
{: .output}

Note the different `DISTRIB_RELEASE` lines in the output above. All we had to change to run a
different distribution is change `16.04` to `18.04`.

> ## Clean up
> Remove any stopped containers that have been just created
>
> > ## Solution
> > ~~~
> > docker container prune
> > ~~~
> {: .solution}
{: .challenge}


## Working with images interactively

To explore images interactively, we need to attach our terminal to the one in the container. This
can be achieved by adding `-it` flag to the `run` command:

~~~
docker run -it ubuntu:16.04
~~~
{: .language-bash}

Now you can work with the image interactively. For example, navigate to the `/etc` directory and
check the content of `lsb-release` file:

~~~
cd /etc
cat lsb-release
~~~
{: .language-bash}

Exit from the container and delete stopped containers.

Now, let's do something useful with containers.  For example, let's install and run NAMD, a parallel
molecular dynamics code designed for high-performance simulations of large biomolecular systems.
Navigate to <https://www.ks.uiuc.edu/Research/namd/> in your browser. Click on "Download NAMD
Binaries" link in "Availability" section and there click on `Linux-x86_64-multicore` under `Version
2.13 (2018-11-09) Platforms`. Enter username and password (you can make them up if you haven't
downloaded it previously) and click on "Continue with registration or download". On the next screen
you'll be asked to provide you basic information -- enter that information and click "Register". On
the next screen, agree to the terms of License and download will begin automatically.

Now, let's discuss how we can move this file into the container.

1. Our first option is to mount directory that contains this file to a directory in Docker
   container. To do that, we can use `--volume` flag of the `run` command:
   ~~~
   docker run -it --name test --volume /Users/mbelkin/Downloads:/usr/local/src/Downloads ubuntu:16.04
   ~~~
   {: .language-bash}
   Not the contents of directory `/usr/local/src/Downloads` in the container is the same as that of
   the `/Users/mbelkin/Downloads` on the host computer. Be careful! If you delete a file in a Docker
   container you will delete it from the host system as well!

2. Alternatively, instead of bind-mounting a directory into the image, we can copy one file only!
   Let's start a Docker container with interactive prompt as we did before. To copy the file into
   the running container, open a new terminal window and execute:

   ~~~
   docker container cp /Path/to/NAMD_2.13_Linux-x86_64-multicore.tar.gz test:/usr/local/src/
   ~~~
   {: .language-bash}

   Now switch back to the first terminal window and check that the file was indeed created:

   ~~~
   ls /usr/local/src
   ~~~
   {: .language-bash}

   ~~~
   NAMD_2.13_Linux-x86_64-multicore.tar.gz
   ~~~
   {: .output}

Let's extract NAMD archive:

~~~
tar xvf NAMD_2.13_Linux-x86_64-multicore.tar.gz
~~~
{: .language-bash}

Now, we need to download some tests that we can use to check NAMD.
To do that from within a container, we need to install `wget` or `curl`:

~~~
apt-get update
apt-get install -y wget
wget http://www.ks.uiuc.edu/Research/namd/utilities/apoa1.tar.gz
tar xvf apoa1.tar.gz
~~~
{: .language-bash}

And now we are ready to run the test!
~~~
mkdir /usr/tmp
./NAMD_2.13_Linux-x86_64-multicore/namd2 apoa1/apoa1.namd
~~~
{: .language-bash}

While the container is running, switch to a new terminal and execute

~~~
docker container pause test
~~~
{: .language-bash}

Switch back to the original terminal. The container has been paused!

To unfreeze, use the `unpause` command:

~~~
docker container unpause test
~~~
{: .language-bash}

Note that container resumed execution of NAMD tests. While they're running, let's execute the
command in that container:

~~~
docker container exec test ps -a
~~~
{: .language-bash}

~~~
  PID TTY          TIME CMD
 2616 pts/0    00:03:05 namd2
~~~
{: .output}

Stop NAMD simulations in the container (switch to the running container and execute
<kbd>CTRL</kbd> + <kbd>C</kbd>. Now, let's save the state of the current container as an image with
`container commit` command:

~~~
docker container commit test ubuntu-with-namd
docker image ls
~~~
{: .language-bash}

Now we can use this newly created image:

~~~
docker run -it --name new-image ubuntu-with-namd
~~~
{: .language-bash}


## Docker files

In the previous section, we created an image from a running container using `container commit`
command. Let's discuss a way how we can create images non-interactively. For that purpose we will
use a Dockerfile. A [Dockerfile](https://docs.docker.com/engine/reference/builder/) is a plain text
file that specifies the instructions to create your container image. A Dockerfile that would
replicate the image that we have just created interactively is below:

~~~
FROM ubuntu:16.04

COPY /Path/to/NAMD_2.13_Linux-x86_64-multicore.tar.gz /usr/local/src/
RUN cd /usr/local/src && tar xvf NAMD_2.13_Linux-x86_64-multicore.tar.gz

RUN apt-get update && \
    apt-get install -y wget
RUN cd /usr/local/src && \
    wget http://www.ks.uiuc.edu/Research/namd/utilities/apoa1.tar.gz && \
    tar xvf apoa1.tar.gz
RUN mkdir /usr/tmp

ENTRYPOINT ["/usr/local/src/NAMD_2.13_Linux-x86_64-multicore/namd2"]
CMD ["/usr/local/src/apoa1/apoa1.namd"]
~~~
{: .output}

To build an image, execute:
~~~
docker build -t ubuntu-with-namd2 -f Dockerfile .
~~~
{: .language-bash}

A quick note of the `docker build` command line.  The `-t` option
specifies the name and tag of the resulting container image. By
default, Docker uses `latest` as the tag unless one is specified.  The
`-f` option specifies the Dockerfile to build the container from.  And
finally, the `.` is the path to use as the build context, i.e., the
sandbox where files from the host are accessible during the
container image build.

The output `Successfully tagged ubuntu-with-namd2:latest` indicates that the
image was built successfully.

Note that each instruction from the Dockerfile is shown as a "Step".
As it builds the container image, Docker tells you which step it is on
and gives the intermediate hash of the resulting layer.

### Image Layering

One of the most important concepts when building container images is
*layering*.  Docker builds container images according to the Open
Container Initiative (OCI) [image
specification](https://github.com/opencontainers/image-spec).  OCI
container images are composed of a series of layers. (If you look
closely at the output of building the first container image above, you
will see that the `ubuntu:16.04` container image itself actually
consists of multiple layers.) The layers are applied sequentially, one
on top of another, to form the container image that you ultimately see
when running a container.

Let's execute:
~~~
docker image history ubuntu-with-namd2
~~~
{: .language-bash}

~~~
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
cc9cab75d4be        10 minutes ago      /bin/sh -c #(nop)  CMD ["/usr/local/src/apoa…   0B
dd04b3ff2d3e        10 minutes ago      /bin/sh -c #(nop)  ENTRYPOINT ["/usr/local/s…   0B
f737da9f32bd        10 minutes ago      /bin/sh -c mkdir /usr/tmp                       0B
b4db1347361b        10 minutes ago      /bin/sh -c cd /usr/local/src &&     wget htt…   23.6MB
a479dd3811c2        10 minutes ago      /bin/sh -c apt-get update &&     apt-get ins…   31.9MB
1b0c05fe7775        11 minutes ago      /bin/sh -c cd /usr/local/src && tar xvf NAMD…   48MB
ddf51c2bc1fd        11 minutes ago      /bin/sh -c #(nop) COPY file:001f14fbe5d059fa…   15.1MB
9361ce633ff1        4 weeks ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>           4 weeks ago         /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B
<missing>           4 weeks ago         /bin/sh -c rm -rf /var/lib/apt/lists/*          0B
<missing>           4 weeks ago         /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   745B
<missing>           4 weeks ago         /bin/sh -c #(nop) ADD file:c02de920036d851cc…   118MB
~~~
{: .output}

The layers are listed in "reverse"
order; the container image you see when running the container is
generated by starting from the last layer shown, applying the second
to the last layer on top of it, then the third from the last on top of
that, and so on. In case of conflicts, a subsequent layer will
overwrite content from previous layers.

The first column shows the layer hash. You can correlate the layer
hashes shown here with the docker build output above.

The second column shows when the layer was created.

The third column shows an abbreviated version of the Dockerfile
instruction used to build the corresponding layer. To see the full
instruction, use docker history --no-trunc.

The fourth column shows the size of the layer.

Note that every "IMAGE" above is, in fact a layer that is stored in the image. The gotcha there is
that the SIZE of the layer is never negative. Even if you delete a file from the image, it is
recorded as a negative mask. Let's change our Dockerfile to remove to source files that we left in
the `/usr/local/src` directory:

~~~
FROM ubuntu:16.04

COPY /Path/to/NAMD_2.13_Linux-x86_64-multicore.tar.gz /usr/local/src/
RUN cd /usr/local/src && tar xvf NAMD_2.13_Linux-x86_64-multicore.tar.gz

RUN apt-get update && \
    apt-get install -y wget
RUN cd /usr/local/src && \
    wget http://www.ks.uiuc.edu/Research/namd/utilities/apoa1.tar.gz && \
    tar xvf apoa1.tar.gz
RUN mkdir /usr/tmp
RUN rm -f /usr/local/src/*.tar.gz

ENTRYPOINT ["/usr/local/src/NAMD_2.13_Linux-x86_64-multicore/namd2"]
CMD ["/usr/local/src/apoa1/apoa1.namd"]
~~~
{: .output}

And build the image:

~~~
docker build -t ubuntu-with-namd3 -f Dockerfile .
~~~
{: .language-bash}

Have you noticed that the image was built significantly faster. This is so because Docker was able
to use cached layers that it previously created. Let's not examine the sizes of the images:

~~~
docker image ls
~~~
{: .language-bash}

~~~
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
ubuntu-with-namd3       latest              5030098f4801        2 minutes ago       236MB
ubuntu-with-namd2       latest              cc9cab75d4be        18 minutes ago      236MB
~~~
{: .output}

The images have the same size even though downloaded files have been deleted and together they add
up to 15 MB! But even worse than that, let's see what happens when file's timestamp is updated.
Change `RUN rm -f /usr/local/src/*.tar.gz` to `RUN touch /usr/local/src/apoa1.tar.gz` and build the
image again:

~~~
docker build -t ubuntu-with-namd4 -f Dockerfile .
~~~
{: .language-bash}

and check the layers of the image:

~~~
docker image history ubuntu-with-namd4
~~~
{: .language-bash}

~~~
1f584deb7a31        3 minutes ago       /bin/sh -c touch /usr/local/src/apoa1.tar.gz    2.89MB
~~~
{: .output}

Docker assumed that the file was changed and recorded its entire contents!
The OCI image specification employs file level deduplication to handle conflicts. When
a build instruction creates or modifies a file, the entire file is saved in the corresponding layer.
Consider the case of a large, 1 GB file. If a subsequent layer modifies a single byte in that file,
the file will account for 1 GB in the container image.

A best practice arising from file level deduplication of layers is to put all actions modifying the
same set of files in the same Dockerfile instruction. For example, remove any temporary files in the
same instruction in which they are created.

Strike a balance between using lots of individual Dockerfile instructions versus using a single
instruction. Lots of individual instructions may produce unnecessarily large container images when
touching the same files, but using too few instructions will eliminate the advantages of the build
cache to speed up your container builds.

A best practice is to bundle all related items into a single layer,
but to put unrelated items in separate layers. For example, install the compiler in one layer and
build your source code in another layer (but cleanup any temporary object files in the same layer
where they are created).

## Hello World!

Let's put these techniques into practice by constructing a container
image for the classic "Hello World!" program.

~~~
#include <stdio.h>

int main() {
  printf("Hello world!\n");
  return 0;
}
~~~
{: .source}

~~~
FROM ubuntu:16.04

# Add the instruction to install gcc here
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
    build-essential \
    gcc && \
    rm -rf /var/lib/apt/lists/*

COPY sources/hello.c /var/tmp/hello.c
RUN gcc -o /var/tmp/hello /var/tmp/hello.c
~~~
{: .output}

This Dockerfile is equivalent to the "Hello World!" Singularity
definition file. As was the case before, the Hello World program
itself is less than 10 kilobytes, yet the Hello World container
image is 292 megabytes, 2.5 time the size of the base image.

[Docker multi-stage
builds](https://docs.docker.com/develop/develop-images/multistage-build/)
are a way to control the size of container images. In the same
Dockerfile, you can define a second stage that is a completely
separate container image and copy just the binary and any runtime
dependencies from preceding stages into the image. The output of a
multi-stage build is a single container image corresponding to the
last stage of the Dockerfile. The multi-stage Hello World Dockerfile
shows how a second FROM instruction starts a second stage, but where
artifacts from the preceding stage can still be accessed (COPY
--from).

~~~
# The "build" stage of the multi-stage Dockerfile
FROM ubuntu:16.04 AS build

RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
    build-essential \
    gcc && \
    rm -rf /var/lib/apt/lists/*

COPY sources/hello.c /var/tmp/hello.c
RUN gcc -o /var/tmp/hello /var/tmp/hello.c

# The "runtime" stage of the multi-stage Dockerfile
# This starts an entirely new container image
FROM ubuntu:16.04

# Copy the hello binary from the build stage
COPY --from=build /var/tmp/hello /var/tmp/hello
~~~
{: .output}

~~~
docker image ls hello-world
~~~
{: .language-bash}

~~~
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         multi-stage         78f118c4f31a        7 seconds ago       117MB
hello-world         single-stage        98c369b8343a        54 seconds ago      292MB
~~~
{: .output}

The container image generated by the multi-stage build adds only the Hello World program to the base
ubuntu:16.04 image, yielding a significant savings in the size of the container. Multi-stage builds
can also be used to avoid redistributing source code or other build artifacts. However, keep in mind
this is a simple case and more complex cases may have additional runtime dependencies that also need
to be copied from one stage to another. HPC Container Maker can help ensure the necessary runtime
dependencies are available in the second stage.


{% include links.md %}
