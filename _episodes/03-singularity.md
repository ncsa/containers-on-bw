---
title: "Containerizing HPC applications"
teaching: 120
exercises: 10
questions:
- "How can one build and run HPC applications in a container?"
objectives:
- "Build container images using Singularity"
- "Build container images using Docker"
- "Explain the key differences between 'flat' Singularity images and 'layered' Docker images"
- "How to use HPC Container Maker to make building container images of HPC applications easier"
- "Run containers that use GPUs (single node)"
- "Run containers that use MPI and inject the MPI library from the host"
- "Run containers that use GPUs and MPI (multiple nodes)"
keypoints:
- "Singularity images are 'flat'"
- "'Flat' images have runtime advantages"
- "To minimize the container image size, do not include development tools and other build artifacts"
- "Docker images are 'layered'"
- "'Layered' images have build time advantages"
- "Multi-stage Docker builds are a powerful technique to minimize image size by only redistributing the application and it's runtime dependencies"
- "Docker images can be easily converted to Singularity images, i.e., build with Docker and run with Singularity to get the best of both"
- "HPC Container Maker (HPCCM) is an open source tool that makes it easier to build container images for HPC applications"
- "HPCCM building blocks uplevel container specification"
- "HPCCM recipes are Python"
- "HPCCM can generate both Dockerfiles and Singularity definition files"
- "Singularity GPU support is enabled by --nv"
- "If the image will be deployed on COTS systems, use the MPI library in the container image"
- "If the image will be deployed on a system that requires a vendor MPI library, inject the MPI library from the host into the container"
- "MPI injection is based on MPICH ABI compatibility"
- "The SINGULARITY_BINDPATH, SINGULARITY_CONTAINLIBS, and SINGULARITYENV_RTLD_SUBSTITUIONS (dl-intercept) are used to inject the host MPI library into the container"
- "Special attention may be necessary for Fortran applications when injecting the host MPI library into the container"
---

Build applications in containers for later use on Blue Waters or other supercomputers.

1. MPI (MPI bandwidth / benchmark)
    - not optimized for the system whete it will be used
2. GPU (1 node GPU app)
    -  CUDA version in the container vs. that on Blue Waters
3. MPI + GPU
    - Singularity: `--nv` + some trickery (bind-mount folders) for MPI (kinda "just works")
    - Shifter: tricks for GPU + the same trickery for MPI
4. (Python / R / Haskell)-based software stacks
    - `apt-get` / `yum` / `pip` / `haskell`

This episode covers building container images with Docker and
Singularity. It also describes how to use [HPC Container
Maker](https://github.com/NVIDIA/hpc-container-maker), a tool to
simplify the process of creating container specification files for
High Performance Computing.  Among the topics discussed are container
specification files, the basics of building container images, and
techniques for managing the size of container images.

The episode assumes you are already familiar with basic Linux shell
commands.

## Building Container Images with Singularity

This part of the episode covers how to [build container images with
Singularity](https://www.sylabs.io/guides/latest/user-guide/build_a_container.html).
Administrative privileges are required to build Singularity container
images.  In contrast to Docker, running Singularity containers does
not require administrative privileges.  By default, Singularity uses a
`setuid` helper program when elevated privileges are needed.

### Building Your First Singularity Image

A [Singularity definition
file](https://www.sylabs.io/guides/latest/user-guide/quick_start.html#singularity-definition-files)
is a plain text file that specifies the instructions to create your
container image. By convention this file is named `Singularity.def`,
but any name may be used. The definition file syntax resembles the
syntax of RPM spec files.

For this first image, we'll use a very simple definition
file. Singularity will build your container image based on the
`ubuntu:16.04` container image from Docker Hub. It will try to find it
locally first, then will go the default repository (Docker Hub) to
download the image. After that is a `%post` instruction that tells the
container builder to run the shell command `date > /build-info.txt`
and save the result as part of the container image.

~~~
BootStrap: docker
From: ubuntu:16.04

%post
    date > /build-info.txt
~~~
{: .output}

~~~
$ sudo singularity build first-image.simg Singularity.def
~~~
{: .language-bash}

~~~
Using container recipe deffile: Singularity.def
Sanitizing environment
Adding base Singularity environment to container
Docker image path: index.docker.io/library/ubuntu:16.04
Cache folder set to /root/.singularity/docker
[4/4] |===================================| 100.0% 
Exploding layer: sha256:7b722c1070cdf5188f1f9e43b8413157f8dfb2b4fe84db3c03cb492379a42fcc.tar.gz
Exploding layer: sha256:5fbf74db61f1459176d8647ba8f53f8e6cf933a2e56f73f0e8da81213117b7e9.tar.gz
Exploding layer: sha256:ed41cb72e5c918bdbd78e68f02930a3f1cf1d6079402b0a5b19de8508e67b766.tar.gz
Exploding layer: sha256:7ea47a67709ebea8efed59fbda703dbd00a0d2cae7e2808959744bfa30bfc0e9.tar.gz
Exploding layer: sha256:c6a9ef4b9995d615851d7786fbc2fe72f72321bee1a87d66919b881a0336525a.tar.gz
Running post scriptlet
+ date
Finalizing Singularity container
Calculating final size for metadata...
Skipping checks
Building Singularity image...
Singularity container built: first-image.simg
Cleaning up...
~~~
{: .output}

A quick note of the `singularity build` command line. The first
argument is the filename of the resulting container image. By
convention, Singularity 2.x container image files have the `.simg`
extension, while Singularity 3.x container images have the `.sif`
extension. The second argument is the path to the Singularity
definition file. 

The output `Singularity container built:first-image.simg` indicates
that the image was built successfully.

Let's check out the newly built image.

~~~
$ singularity exec first-image.simg cat /build-info.txt
~~~
{: .language-bash}

~~~
Wed Jan 23 22:02:30 UTC 2019
~~~
{: .output}

The date shown should be just a short time ago when you built the
image. The date in this file corresponds to when the container image
was built, not when it is run.

Note that the container image is a single file. Normal file operations
such as chmod, mv, scp, etc. can be applied to the image.

~~~
$ ls -lh first-image.simg
~~~
{: .language-bash}

~~~
-rwxr-xr-x. 1 root root 36M Jan 23 22:02 first-image.simg
~~~
{: output}

> ### Hello World!
> Let's put these techniques into practice by constructing a container
> image for the classic "Hello World!" program.
> 
> ~~~
> #include <stdio.h>
>
> int main() {
>   printf("Hello world!\n");
> }
> ~~~
> {: language-c}
> 
> The Ubuntu base container on Docker Hub does not include development
> tools in order to help minimize the size of the image. As an
> exercise, modify the Singularity definition file to install the GNU
> C compiler and standard C headers. For Ubuntu, the command to
> install packages is `apt-get`. The packages are named `gcc` and
> `build-essential`.
>
> ~~~
> BootStrap: docker
> From: ubuntu:16.04
>
> %files
>     hello.c /var/tmp/hello.c
>
> %post
>     # Add the command to install gcc here
>
>     gcc -o /usr/local/bin/hello /var/tmp/hello.c
> ~~~
> {: language-bash}
> > ## Solution
> > ~~~
> > BootStrap: docker
> > From: ubuntu:16.04
> >
> > %files
> >     hello.c /var/tmp/hello.c
> >
> > %post
> >     # Add the command to install gcc here
> >     apt-get update -y
> >     apt-get install -y --no-install-recommends \
> >         build-essential \
> >         gcc
> >     rm -rf /var/lib/apt/lists/*
> >
> >     gcc -o /usr/local/bin/hello /var/tmp/hello.c
> > ~~~
> > {: .language-bash}
> >
> > Note that the apt package cache is removed to minimize the image size.
> {: .solution}
>
> Verify your solution by running the Hello World program inside the
> container.
>
> ~~~
> $ singularity exec hello-world.simg /usr/local/bin/hello
> ~~~
> {: .language-bash}
>
> ~~~
> Hello World!
> ~~~
> {: .output}
>
> Let's take a closer look at the Hello World container image.
>
> ~~~
> ls -lh hello-world.simg
> ~~~
> {: .language-bash}
>
> ~~~
> -rwxr-xr-x. 1 root root 93M Jan 23 22:21 hello-world.simg
> ~~~
> {: .output}
>
> The image first-image.simg, which was essentially just the base
> Ubuntu container image, is 36 megabytes. The Hello World program
> itself is less than 10 kilobytes, yet the Hello World container
> image is 93 megabytes, 2.5 times the size of the base image. The
> compiler accounts for over half of the total container size! But
> all we really care about is the Hello World program, there is no
> need to redistribute the compiler (or our source code) to users of
> the container image.
>
> You could reduce the size of the Singularity container image by
> removing the source code and compiler after the Hello World program
> has been built. Doing so would reduce the container image size back
> to 36 megabytes. However, more complex programs with runtime
> dependencies will require more sophisticated steps to remove
> unnecessary components while maintaining the needed runtime
> dependencies.  HPC Container Maker can handle runtime
> dependencies automatically.
>
> The Singularity "flat" image format is optimal for running a
> container, but the unlike Docker, Singularity does not have the
> concepts of image layers, caches, or multi-stage containers.  The
> process of building "layered" images is more flexible and can
> produce significantly smaller container images.  Luckily,
> Singularity can easily convert layered Docker images into flat
> Singularity images.
{: .challenge}

## Building Container Images with Docker

The next part of the episode covers how to build container images with
Docker.  Administrative privileges are required to use Docker.

### Building Your First Docker Image

A [Dockerfile](https://docs.docker.com/engine/reference/builder/) is a
plain text file that specifies the instructions to create your
container image.  For this first image, we'll use a very simple
Dockerfile.  Docker will build your container image based on the
`ubuntu:16.04` container image from Docker Hub.  It will try to find
it locally first, then will go the default repository (Docker Hub) to
download the image.  After that is a `RUN` instruction that tells the
container builder to run the shell command `date > /build-info.txt`
and save the result as part of the container image.

~~~
FROM ubuntu:16.04

RUN date > /build-info.txt
~~~
{: .output}

~~~
$ sudo docker build -t first-image -f Dockerfile .
~~~
{: .language-bash}

~~~
Sending build context to Docker daemon  134.4MB
Step 1/2 : FROM ubuntu:16.04
16.04: Pulling from library/ubuntu

2c1070cd: Pulling fs layer 
74db61f1: Pulling fs layer 
cb72e5c9: Pulling fs layer 
Digest: sha256:e4a134999bea4abb4a27bc437e6118fdddfb172e1b9d683129b74d254af51675
Status: Downloaded newer image for ubuntu:16.04
 ---> 7e87e2b3bf7a
Step 2/2 : RUN date > /build-info.txt
 ---> Running in 6e35a0f7b22b
Removing intermediate container 6e35a0f7b22b
 ---> 27cf56005b9e
Successfully built 27cf56005b9e
Successfully tagged first-image:latest
~~~
{: .output}

A quick note of the `docker build` command line.  The `-t` option
specifies the name and tag of the resulting container image. By
default, Docker uses `latest` as the tag unless one is specified.  The
`-f` option specifies the Dockerfile to build the container from.  And
finally, the `.` is the path to use as the build context, i.e., the
sandbox where files from the host are accessible during the
container image build.

The output `Successfully tagged first-image:latest` indicates that the
image was built successfully.

Note that each instruction from the Dockerfile is shown as a "Step".
As it builds the container image, Docker tells you which step it is on
and gives the intermediate hash of the resulting layer.

Let's check out the newly built image.

~~~
$ docker run --rm -it first-image cat /build-info.txt
~~~
{: .language-bash}

~~~
Wed Jan 23 22:45:53 UTC 2019
~~~
{: .output}

The date shown should be just a short time ago when you built the
image. The date in this file corresponds to when the container image
was built, not when it is run.

### Image Layering

One of the most important concepts when building container images is
*layering*.  Docker builds container images according to the [Open
Container Initiative (OCI) image
specification](https://github.com/opencontainers/image-spec).  OCI
container images are composed of a series of layers. (If you look
closely at the output of building the first container image above, you
will see that the `ubuntu:16.04` container image itself actually
consists of multiple layers.) The layers are applied sequentially, one
on top of another, to form the container image that you ultimately see
when running a container.

To help illustrate layering, let's extend the previous Dockerfile to
add a second `RUN` instruction that appends the Linux kernel version
of the system where the container was built to `/build-info.txt`."

~~~
FROM ubuntu:16.04

RUN date > /build-info.txt
RUN uname -r >> /build-info.txt
~~~
{: .output}

~~~
$ docker build -t second-image -f Dockerfile .
~~~
{: .language-bash}

~~~
Sending build context to Docker daemon  134.4MB
Step 1/3 : FROM ubuntu:16.04
 ---> 7e87e2b3bf7a
Step 2/3 : RUN date > /build-info.txt
 ---> Using cache
 ---> 27cf56005b9e
Step 3/3 : RUN uname -r >> /build-info.txt
 ---> Running in b9250b16b473
Removing intermediate container b9250b16b473
 ---> ad5acf7716ae
Successfully built ad5acf7716ae
Successfully tagged second-image:latest
~~~
{: .output}

First, note that first 2 steps were cached. Docker recognizes that the
first 2 instructions have previously been processed, so the
corresponding layers do not need to be regenerated. This is possible
due to layering. The layer cache can significantly speed up building
container images. Recall that the layers are applied sequentially; so
the entire history of instructions up to that point must be identical
for the cached layer to be used.

The third step which we just added to the Dockerfile is not in the
cache, so it needs to be performed and a new layer is generated.

Let's verify that the kernel version is included in the build info
file.

~~~
$ sudo docker run --rm -it second-image cat /build-info.txt
~~~
{: .language-bash}

~~~
Wed Jan 23 22:45:53 UTC 2019
4.4.115-1.el7.elrepo.x86_64
~~~
{: .output}

Docker provides a method to take a closer look at the layers composing
a container image.

~~~
$ sudo docker history second-image
~~~
{: .language-bash}

~~~
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
ad5acf7716ae        8 seconds ago       /bin/sh -c uname -r >> /build-info.txt          57B                 
27cf56005b9e        8 minutes ago       /bin/sh -c date > /build-info.txt               29B                 
7e87e2b3bf7a        24 hours ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           24 hours ago        /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B                  
<missing>           24 hours ago        /bin/sh -c rm -rf /var/lib/apt/lists/*          0B                  
<missing>           24 hours ago        /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   745B                
<missing>           24 hours ago        /bin/sh -c #(nop) ADD file:916a45030dee881bb…   117MB          
~~~
{: .output}

Your image consists of 7 layers. The layers are listed in "reverse"
order; the container image you see when running the container is
generated by starting from the last layer shown, applying the second
to the last layer on top of it, then the third from the last on top of
that, and so on. In case of conflicts, a subsequent layer will
overwrite content from previous layers.

The first column shows the layer hash. You can correlate the layer
hashes shown here with the docker build output above.

The second column shows when the layer was created. You created the
top 2 layers just a few minutes ago, while the other layers correspond
to the ubuntu:16.04 base image and were created longer ago.

The third column shows an abbreviated version of the Dockerfile
instruction used to build the corresponding layer. To see the full
instruction, use docker history --no-trunc. The instructions for the
top 2 layers match what was specified in the Dockerfile.

The fourth column shows the size of the layer. Why is the layer that
appended the kernel version (`uname -r ...`) almost twice as large the
layer that saved the date?

The OCI image specification employs file level deduplication to handle
conflicts. When a build instruction creates or modifies a file, the
entire file is saved in the corresponding layer. So when the kernel
version was appended to the build info file, that layer did not
capture just the difference, but rather the whole modified file. In
this particular case, the file is tiny and the amount of duplicated
data is minimal. But consider the case of a large, 1 GB file. If a
subsequent layer modifies a single byte in that file, the file will
account for 2 GB in the container image, even though the file will
appear to be "only" 1 GB when running the container.

A best practice arising from file level deduplication of layers is to
put all actions modifying the same set of files in the same Dockerfile
instruction. For example, remove any temporary files in the same
instruction in which they are created.

Let's modify the Dockerfile so that the date and kernel version are
written to the build info file in the same instruction. In the bash
shell, commands can be concatenated with &&. (You may have noticed
long RUN commands connected with && in other Dockerfiles; this is
why.)

~~~
FROM ubuntu:16.04

RUN date > /build-info.txt && uname -r >> /build-info.txt
~~~
{: .output}

~~~
$ sudo docker history third-image
~~~
{: .language-bash}

~~~
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
fa5646fd610e        2 seconds ago       /bin/sh -c date > /build-info.txt && uname -…   57B                 
7e87e2b3bf7a        24 hours ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           24 hours ago        /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B                  
<missing>           24 hours ago        /bin/sh -c rm -rf /var/lib/apt/lists/*          0B                  
<missing>           24 hours ago        /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   745B                
<missing>           24 hours ago        /bin/sh -c #(nop) ADD file:916a45030dee881bb…   117MB       
~~~
{: .output}

Notice there is now a single layer for the build info file and the
extraneous layer with the duplicated data has been eliminated.

Strike a balance between using lots of individual Dockerfile
instructions versus using a single instruction. Lots of individual
instructions may produce unnecessarily large container images when
touching the same files, but using too few instructions will eliminate
the advantages of the build cache to speed up your container builds.

A best practice is to bundle all related items into a single layer,
but to put unrelated items in separate layers. For example, install
the compiler in one layer and build your source code in another layer
(but cleanup any temporary object files in the same layer where they
are created).

> ### Hello World!
> Let's put these techniques into practice by constructing a container
> image for the classic "Hello World!" program.
> 
> ~~~
> #include <stdio.h>
>
> int main() {
>   printf("Hello world!\n");
> }
> ~~~
> {: language-c}
> 
> ~~~
> FROM ubuntu:16.04
>
> # Add the instruction to install gcc here
> RUN apt-get update -y && \
>     apt-get install -y --no-install-recommends \
>     build-essential \
>     gcc && \
>     rm -rf /var/lib/apt/lists/*
>
> COPY sources/hello.c /var/tmp/hello.c
> RUN gcc -o /var/tmp/hello /var/tmp/hello.c
> ~~~
> {: language-bash}
>
> This Dockerfile is equivalent to the "Hello World!" Singularity
> definition file. As was the case before, the Hello World program
> itself is less than 10 kilobytes, yet the Hello World container
> image is 292 megabytes, 2.5 time the size of the base image.
>
> [Docker multi-stage
> builds](https://docs.docker.com/develop/develop-images/multistage-build/)
> are a way to control the size of container images. In the same
> Dockerfile, you can define a second stage that is a completely
> separate container image and copy just the binary and any runtime
> dependencies from preceding stages into the image. The output of a
> multi-stage build is a single container image corresponding to the
> last stage of the Dockerfile. The multi-stage Hello World Dockerfile
> shows how a second FROM instruction starts a second stage, but where
> artifacts from the preceding stage can still be accessed (COPY
> --from).
>
> ~~~
> # The "build" stage of the multi-stage Dockerfile
> FROM ubuntu:16.04 AS build
>
> RUN apt-get update -y && \
>     apt-get install -y --no-install-recommends \
>     build-essential \
>     gcc && \
>     rm -rf /var/lib/apt/lists/*
>
> COPY sources/hello.c /var/tmp/hello.c
> RUN gcc -o /var/tmp/hello /var/tmp/hello.c
>
> # The "runtime" stage of the multi-stage Dockerfile
> # This starts an entirely new container image
> FROM ubuntu:16.04
>
> # Copy the hello binary from the build stage
> COPY --from=build /var/tmp/hello /var/tmp/hello
> ~~~
> {: .output}
>
> ~~~
> $ sudo docker images hello-world
> ~~~
> {: .language-bash}
>
> ~~~
> REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
> hello-world         multi-stage         78f118c4f31a        7 seconds ago       117MB
> hello-world         single-stage        98c369b8343a        54 seconds ago      292MB
> ~~~
> {: .output}
>
> The container image generated by the multi-stage build adds only the
> Hello World program to the base ubuntu:16.04 image, yielding a
> significant savings in the size of the container. Multi-stage builds
> can also be used to avoid redistributing source code or other build
> artifacts. However, keep in mind this is a simple case and more
> complex cases may have additional runtime dependencies that also
> need to be copied from one stage to another. HPC Container Maker can
> help ensure the necessary runtime dependencies are available in the
> second stage.

It is possible to benefit from the Docker container build capabilities
such as the layer cache and multi-stage builds, and the Singularity
runtime capabilities such a single file "flat" container image and
rootless containers.  Build your containers images using Docker, and
then convert the image to a Singularity image using the
`docker2singularity` converter.  This workflow will be used for the
rest of the episode.

~~~
$ sudo docker run -t --rm --cap-add SYS_ADMIN -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/output singularityware/docker2singularity <container>
~~~
{: .language-bash}

## HPC Container Maker

[HPC Container Maker
(HPCCM)](https://github.com/NVIDIA/hpc-container-maker) simplifies the
process of creating container specification files. It specifically
addresses the challenges of generating HPC container images.

HPC Container Maker generates Dockerfiles or Singularity definition
files from a high level Python recipe. HPCCM recipes have some
distinct advantages over "native" container specification formats.

1. A library of HPC building blocks that separate the choice of what
   to include in a container image from the details of how it's
   done. The building blocks transparently provide the latest
   component and container best practices.

2. Python provides increased flexibility over static container
   specification formats. Python-based recipes can branch, validate
   user input, etc. - the same recipe can generate multiple container
   specifications.

3. Generate either Dockerfiles or Singularity definition files from
   the same recipe.

HPCCM is based on the concept of [building
blocks](https://github.com/NVIDIA/hpc-container-maker/blob/master/docs/building_blocks.md). For
instance, there is an [OpenMPI building
block](https://github.com/NVIDIA/hpc-container-maker/blob/master/docs/building_blocks.md#openmpi). The
building blocks encapsulate the best practices of building HPC
software components with the best practices of building container
images to generate optimal container image specifications. This lets
you easily take advantage of all the existing knowledge of how to best
install a component like OpenMPI inside a container image.

Container images are specified as a HPCCM recipe, which is then
converted by a command line tool into a Dockerfile or a Singularity
definition file. A HPCCM recipe is a Python script, usually a really
simple Python script. But you do have the full power of Python
available to you so you can do things like validate input, branch
inside the recipe based on the type of build desired, or even search
the web to download the latest version of a software package.

> ### Getting Started
> To illustrate this, let's start with a simple example of a container
> image that includes CUDA and OpenMPI.
>
> ~~~
> Stage0 += baseimage(image='nvidia/cuda:9.2-devel-centos7')
> Stage0 += openmpi(infiniband=False)
> ~~~
> {: .language-python}
>
> ~~~
> $ hpccm --recipe openmpi.py
> ~~~
> {: .language-bash}
>
> ~~~
> FROM nvidia/cuda:9.2-devel-centos7
>
> # OpenMPI version 3.1.2
> RUN yum install -y \
>         bzip2 \
>         file \
>         hwloc \
>         make \
>         numactl-devel \
>         openssh-clients \
>         perl \
>         tar \
>         wget && \
>     rm -rf /var/cache/yum/*
> RUN mkdir -p /var/tmp && wget -q -nc --no-check-certificate -P /var/tmp https://www.open-mpi.org/software/ompi/v3.1/downloads/openmpi-3.1.2.tar.bz2 && \
>     mkdir -p /var/tmp && tar -x -f /var/tmp/openmpi-3.1.2.tar.bz2 -C /var/tmp -j && \
>     cd /var/tmp/openmpi-3.1.2 &&   ./configure --prefix=/usr/local/openmpi --disable-getpwuid --enable-orterun-prefix-by-default --with-cuda --without-verbs && \
>     make -j$(nproc) && \
>     make -j$(nproc) install && \
>     rm -rf /var/tmp/openmpi-3.1.2.tar.bz2 /var/tmp/openmpi-3.1.2
> ENV LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH \
>     PATH=/usr/local/openmpi/bin:$PATH
> ~~~
> {: .output}
>
> When this simple two line recipe is processed by HPCCM, the
> optimized Dockerfile is generated. Notice that the Dockerfile best
> practices described earlier, such as combining related steps into a
> single layer and removing temporary files in the same layer they are
> generated are automatically employed.
>
> A Singularity definition file can be generated from the exact same
> recipe just by specifying the --format command line option.
>
> ~~~
> $ hpccm --recipe hpccm/openmpi.py --format singularity 
> ~~~
> {: .language-bash}
>
> ~~~
> BootStrap: docker
> From: nvidia/cuda:9.2-devel-centos7
> %post
>     . /.singularity.d/env/10-docker.sh
>
> # OpenMPI version 3.1.2
> %post
>     yum install -y \
>         bzip2 \
>         file \
>         hwloc \
>         make \
>         numactl-devel \
>         openssh-clients \
>         perl \
>         tar \
>         wget
>     rm -rf /var/cache/yum/*
> %post
>     cd /
>     mkdir -p /var/tmp && wget -q -nc --no-check-certificate -P /var/tmp https://www.open-mpi.org/software/ompi/v3.1/downloads/openmpi-3.1.2.tar.bz2
>     mkdir -p /var/tmp && tar -x -f /var/tmp/openmpi-3.1.2.tar.bz2 -C /var/tmp -j
>     cd /var/tmp/openmpi-3.1.2 &&   ./configure --prefix=/usr/local/openmpi --disable-getpwuid --enable-orterun-prefix-by-default --with-cuda --without-verbs
>     make -j$(nproc)
>     make -j$(nproc) install
>     rm -rf /var/tmp/openmpi-3.1.2.tar.bz2 /var/tmp/openmpi-3.1.2
> %environment
>     export LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH
>     export PATH=/usr/local/openmpi/bin:$PATH
> %post
>     export LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH
>     export PATH=/usr/local/openmpi/bin:$PATH
> ~~~
> {: .output}
>
> HPCCM building blocks are also configurable. The defaults are
> suitable for many use cases, but you may need to more precisely
> tailor the container image. For example, the [OpenMPI building
> block](https://github.com/NVIDIA/hpc-container-maker/blob/master/docs/building_blocks.md#openmpi)
> has several configuration options.
>
> For example, this recipe installs OpenMPI in /opt, disables the
> Fortran interface and InfiniBand support, and specifies to use
> version 2.1.2. Also note that the base image is based on Ubuntu
> rather than CentOS, as in the previous recipe; the building block
> automatically detected the Linux distribution type and uses apt-get
> rather than yum to install its dependencies.
>
> ~~~
> Stage0 += baseimage(image='nvidia/cuda:9.2-devel-ubuntu16.04')
> Stage0 += openmpi(configure_opts=['--disable-getpwuid',
>                                   '--enable-orterun-prefix-by-default',
>                                   '--disable-fortran'],
>                   infiniband=False, prefix='/opt/openmpi', version='2.1.2')
> ~~~
> {: .language-python}
>
> ~~~
> FROM nvidia/cuda:9.2-devel-ubuntu16.04
>
> # OpenMPI version 2.1.2
> RUN apt-get update -y && \
>     DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
>         bzip2 \
>         file \
>         hwloc \
>         libnuma-dev \
>         make \
>         openssh-client \
>         perl \
>         tar \
>         wget && \
>     rm -rf /var/lib/apt/lists/*
> RUN mkdir -p /var/tmp && wget -q -nc --no-check-certificate -P /var/tmp https://www.open-mpi.org/software/ompi/v2.1/downloads/openmpi-2.1.2.tar.bz2 && \
>     mkdir -p /var/tmp && tar -x -f /var/tmp/openmpi-2.1.2.tar.bz2 -C /var/tmp -j && \
>     cd /var/tmp/openmpi-2.1.2 &&   ./configure --prefix=/opt/openmpi --disable-getpwuid --enable-orterun-prefix-by-default --disable-fortran --with-cuda --without-verbs && \
>     make -j$(nproc) && \
>     make -j$(nproc) install && \
>     rm -rf /var/tmp/openmpi-2.1.2.tar.bz2 /var/tmp/openmpi-2.1.2
> ENV LD_LIBRARY_PATH=/opt/openmpi/lib:$LD_LIBRARY_PATH \
>     PATH=/opt/openmpi/bin:$PATH
> ~~~
> {: .output}

> ## Using MPI
>
> The [MPI Bandwidth sample
> program](https://computing.llnl.gov/tutorials/mpi/samples/C/mpi_bandwidth.c)
> from the Lawrence Livermore National Laboratory (LLNL) will be used
> as a proxy application to illustrate how to use HPCCM recipes to
> create application containers and how MPI can be used with
> Singularity.
>
> For system with COTS (commercial off the shelf) network fabrics such
> as Ethernet or InfiniBand, you would use the MPI library embedded in
> the container. The OpenMPI library detects the best available
> network fabric and automatically uses it, so the same container can
> be deployed on most COTS systems regardless of the network fabric.
>
> The MPI Bandwidth for COTS clusters includes the mlnx_ofed building
> block to install the Mellanox OpenFabrics Enterprise Distribution
> (OFED) user space software and a IB-enabled version of OpenMPI.
>
> ~~~
> Stage0 += baseimage(image='centos:7')
> Stage0 += gnu(fortran=False)
> Stage0 += mlnx_ofed()
> Stage0 += openmpi(cuda=False, version='3.0.0')
>
> # MPI Bandwidth
> Stage0 += shell(commands=[
>     'wget -q -nc --no-check-certificate -P /var/tmp https://computing.llnl.gov/tutorials/mpi/samples/C/mpi_bandwidth.c',
>     'mpicc -o /usr/local/bin/mpi_bandwidth /var/tmp/mpi_bandwidth.c'])
>
> Stage1 += baseimage(image='centos:7')
> Stage1 += Stage0.runtime()
>
> # MPI Bandwidth
> Stage1 += copy(_from='0', src='/usr/local/bin/mpi_bandwidth',
>                dest='/usr/local/bin/mpi_bandwidth')
> ~~~
> {: .language-python}
>
> For systems with non-COTS network fabrics that require a vendor MPI
> library in order to realize full performance, such as the Blue
> Waters system at NCSA, a different container approach is
> required. MPI libraries derived from MPICH are [ABI
> compatible](https://www.mpich.org/abi/). In other words, the MPI
> library used to build an application may be "swapped" out with
> another MPI library at runtime.
>
> The outline of the recipe is the same - include compiler and MPI
> building blocks, and build the MPI Bandwidth application - but some
> of the specifics differ.  A "vanilla" version of MPICH is used to
> build the MPI Bandwidth application since it will just be swapped
> out with an optimized MPI library at runtime. The
> [dl-intercept](https://github.com/olcf/dl-intercept) library is also
> included in the container to make the swapping process easier.  OFED
> can be omitted since system specific fabric libraries will also be
> mapped into the container at runtime.
>
> ~~~
> from hpccm.templates.CMakeBuild import CMakeBuild
> from hpccm.templates.git import git
>
> Stage0 += baseimage(image='centos:7')
> Stage0 += gnu()
> Stage0 += mpich()
>
> # dl-intercept
> Stage0 += boost()
> Stage0 += cmake(eula=True)
> Stage0 += packages(ospackages=['ca-certificates', 'git', 'libstdc++-static'])
> cm = CMakeBuild()
> Stage0 += shell(commands=[
>   git().clone_step(repository='https://github.com/olcf/dl-intercept',
>                    branch='v1.0.2', path='/var/tmp'),
>   cm.configure_step(directory='/var/tmp/dl-intercept',
>                     opts=['-DCMAKE_INSTALL_PREFIX=/usr/local',
>                           '-DCMAKE_BUILD_TYPE=RELEASE',
>                           '-DBOOST_ROOT=/usr/local/boost']),
>   cm.build_step(),
>   cm.build_step(target='install')])
>
> # MPI Bandwidth
> Stage0 += shell(commands=[
>     'wget -q -nc --no-check-certificate -P /var/tmp https://computing.llnl.gov/tutorials/mpi/samples/C/mpi_bandwidth.c',
>     'mpicc -o /usr/local/bin/mpi_bandwidth /var/tmp/mpi_bandwidth.c'])
>
> Stage1 += baseimage(image='centos:7')
> Stage1 += Stage0.runtime(exclude=['boost'])
> 
> # dl-intercept
> Stage1 += copy(_from='0', dest='/usr/local/lib/libdl-intercept.so',
>                src='/usr/local/lib/libdl-intercept.so')
> Stage1 += environment(
>   variables={'LD_AUDIT': '/usr/local/lib/libdl-intercept.so'})
>
> # MPI Bandwidth
> Stage1 += copy(_from='0', src='/usr/local/bin/mpi_bandwidth',
>                dest='/usr/local/bin/mpi_bandwidth')
> ~~~
> {: .language-python}
>
> Build the container image using Docker and then convert it into a
> Singularity image.
>
> ~~~
> $ hpccm --recipe mpi_bandwidth.py > Dockerfile
> $ docker build -t mpi_bandwidth -f Dockerfile .
> $ sudo docker run -t --rm --cap-add SYS_ADMIN -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/output singularityware/docker2singularity mpi_bandwidth
> ~~~
> {: .language-bash}
>
> To inject a MPI library from the host into a Singularity container,
> 3 things are necessary. First, the directory containing the host MPI
> library must be mounted into the container.  Second, any network
> fabric library dependencies or other MPI library dependencies must
> also be injected into the container.  And finally, the environment
> needs to be configured for the application to use the injected MPI
> library rather than the version inside the container.  All 3 of
> these items can be accomplished by setting a few environment
> variables.
>
> The SINGULARITY_BINDPATH environment variable mounts the specified
> paths inside the container at the same location. Corresponding mount
> points do not need to preexist inside the container.  For instance,
> if your host software environment is installed in /opt/sw, you would
> set SINGULARITY_BINDPATH=/opt/sw to mount it inside the container.
>
> The SINGULARITY_CONTAINLIBS environment variable injects the
> specified libraries into the container.  The libraries are injected
> in /.singularity.d/libs.  The location is largely immaterial as this
> directory is automatically added to LD_LIBRARY_PATH by Singularity.
> Typically only libraries from system paths such as /lib64 need to be
> injected this way if the entire host software directory is mounted
> using SINGULARITY_BINDPATH.  You may need to extend
> SINGULARITY_BINDPATH to include configuration or other files that
> are installed in system locations such as /etc.
>
> The RTLD_SUBSTITUTIONS environment variable, recognized by the
> dl-intercept library, performs the library swap. Environment
> variables with the SINGULARITYENV_ prefix are set by Singularity
> only inside the container.  
>
> To inject the Intel MPI Library from the host into the container on
> an InfiniBand cluster where the host software environment is
> installed in /opt/sw, the environment would be similar to the
> following.
>
> ~~~
> $ export SINGULARITY_BINDPATH="/opt/sw,/etc/libibverbs.d,/etc/dat.conf"
> $ export SINGULARITY_CONTAINLIBS="/lib64/libnuma.so.1,/lib64/libibverbs.so.1,/lib64/libdat2.so.2,/lib64/libnl-3.so.200,/lib64/libnl-route-3.so.200,/lib64/libmlx4-rdmav2.so,/lib64/libmlx5-rdmav2.so,/lib64/librxe-rdmav2.so,/lib64/libdaploucm.so.2,/lib64/libdaploscm.so.2,/lib64/libdaplofa.so.2"
> $ export SINGULARITYENV_RTLD_SUBSTITUTIONS="libmpi.so.12:/opt/sw/intel/compilers_and_libraries_2018.1.163/linux/mpi/intel64/lib/libmpi.so.12"
> ~~~
> {: .language-bash}
>
> The system administrator could prepopulate these environment settings
> in an environment module or in /etc/singularity/singularity.conf to
> simplify the process.
>
> With those environment variables set, you can run the containerized
> version of MPI Bandwidth using the MPI library injected from the
> host using a typical mpirun + Singularity command line.
>
> ~~~
> $ mpirun -n 2 -f hostfile ... singularity exec mpi_bandwidth.simg /usr/local/bin/mpi_bandwidth
> ~~~
> {: .language-bash}

> ## Using GPUs
> 
> The [CUDA STREAM](https://github.com/bcumming/cuda-stream) benchmark
> will be used a proxy application to illustrate how to use a HPCCM
> recipe to create application containers and how GPUs can used with
> Singularity.
>
> Using GPUs with Singularity is much simpler than injecting MPI from
> the host.  Just use the --nv option with Singularity and Singularity
> will automatically handle injecting the GPU libraries from the host
> inside the container.
>
> Building containers to support GPUs is simple as well.  Just use the
> CUDA containers provided free of charge by NVIDIA on Docker Hub.
>
> The CUDA development container already includes the necessary
> compilers, so just install a few packages necessary to clone a git
> repository and the recipe is ready to build CUDA STREAM.
>
> ~~~
> from hpccm.templates.git import git
>
> Stage0 += baseimage(image='nvidia/cuda:9.1-devel-centos7')
>
> # CUDA STREAM
> Stage0 += packages(ospackages=['ca-certificates', 'git'])
> Stage0 += shell(commands=[
>   git().clone_step(repository='https://github.com/bcumming/cuda-stream.git',
>                    path='/var/tmp'),
>   'cd /var/tmp/cuda-stream',
>   'nvcc -std=c++11 -ccbin=g++ -gencode arch=compute_35,code=\\"sm_35,compute_35\\" -o stream stream.cu'])
>
> Stage1 += baseimage(image='nvidia/cuda:9.1-base-centos7')
>
> Stage1 += copy(_from='0', dest='/usr/local/bin/stream',
>                src='/var/tmp/cuda-stream/stream')
>
> Stage1 += runscript(commands=['/usr/local/bin/stream'])
> ~~~
> {: .language-python}
>
> Build the container image using Docker and then convert it into a
> Singularity image.
>
> ~~~
> $ hpccm --recipe cuda_stream.py > Dockerfile
> $ docker build -t cuda_stream -f Dockerfile .
> $ sudo docker run -t --rm --cap-add SYS_ADMIN -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/output singularityware/docker2singularity cuda_stream
> ~~~
> {: .language-bash}
>
> Just add --nv to enable GPU support in Singularity.
>
> ~~~
> $ singularity run --nv cuda_stream.simg
> ~~~
> {: .language-bash}

> ## Using MPI and GPUs
>
> The [CloverLeaf](http://uk-mac.github.io/CloverLeaf/) mini-app will
> be used as a proxy application to illustrate how to use a HPCCM
> recipe to create a container supporting GPUs and injecting MPI from
> the host with Singularity. Several implementations of CloverLeaf
> are available; the MPI+OpenACC version will be used here.
>
> As a Fortran code using OpenACC, the PGI compiler will be used to
> compile it.  As in the MPI Bandwidth example, a "vanilla" MPICH and
> the dl-intercept library will be used to facilitate injecting a host
> MPI library at runtime. The HPCCM recipe is similar to the MPI
> Bandwidth case, but with MPICH built using the PGI compiler. As in
> the CUDA STREAM case, the CUDA Docker Hub base images will be used
> for GPU support.
>
> ~~~
> from hpccm.templates.CMakeBuild import CMakeBuild
> from hpccm.templates.git import git
> from hpccm.templates.sed import sed
>
> Stage0 += baseimage(image='nvidia/cuda:9.1-devel-centos7')
>
> # Compilers - will use PGI but need GNU as well
> Stage0 += gnu()
> compiler = pgi(eula=True)
> Stage0 += compiler
>
> # MPICH - build using the PGI compiler
> Stage0 += mpich(toolchain=compiler.toolchain)
>
> # dl-intercept
> Stage0 += boost()
> Stage0 += cmake(eula=True)
> Stage0 += packages(ospackages=['ca-certificates', 'git', 'libstdc++-static'])
> cm = CMakeBuild()
> Stage0 += shell(commands=[
>   git().clone_step(repository='https://github.com/olcf/dl-intercept',
>                    branch='v1.0.2', path='/var/tmp'),
>   'cd /var/tmp/dl-intercept',
>   cm.configure_step(directory='/var/tmp/dl-intercept',
>                     opts=['-DCMAKE_INSTALL_PREFIX=/usr/local',
>                           '-DCMAKE_BUILD_TYPE=RELEASE',
>                           '-DBOOST_ROOT=/usr/local/boost']),
>   cm.build_step(),
>   cm.build_step(target='install')])
>
> # CloverLeaf OpenACC
> Stage0 += shell(commands=[
>   git().clone_step(repository='https://github.com/UK-MAC/CloverLeaf_OpenACC',
>                    branch='master', path='/var/tmp'),
>   # Build for all compute capabilities for broadest GPU support
>   sed().sed_step(file='/var/tmp/CloverLeaf_OpenACC/Makefile',
>                  patterns=[r's/cc35/ccall/g']),
>   'cd /var/tmp/CloverLeaf_OpenACC',
>   'COMPILER=PGI make'])
>
> Stage1 += baseimage(image='nvidia/cuda:9.1-base-centos7')
> Stage1 += Stage0.runtime(exclude=['boost'])
>
> # dl-intercept
> Stage1 += copy(_from='0', dest='/usr/local/lib/libdl-intercept.so',
>                src='/usr/local/lib/libdl-intercept.so')
> Stage1 += environment(
>   variables={'LD_AUDIT': '/usr/local/lib/libdl-intercept.so'})
> 
> # CloverLeaf
> Stage1 += copy(_from='0', dest='/opt/CloverLeaf/OpenACC/clover_leaf',
>                src='/var/tmp/CloverLeaf_OpenACC/clover_leaf')
> Stage1 += copy(_from='0', dest='/opt/CloverLeaf/OpenACC/InputDecks',
>                src='/var/tmp/CloverLeaf_OpenACC/InputDecks')
> Stage1 += shell(commands=['ln -s /opt/CloverLeaf/OpenACC/InputDecks/clover_bm16_short.in /opt/CloverLeaf/OpenACC/clover.in'])
> ~~~
> {: .language-python}
>
> Building the container image follows the same workflow as the
> previous cases: generate a Dockerfile from the HPCCM recipe, build a
> Docker container image, and then convert it to a Singularity
> container image.
>
> ~~~
> $ hpccm --recipe cloverleaf.py > Dockerfile
> $ docker build -t cloverleaf -f Dockerfile .
> $ sudo docker run -t --rm --cap-add SYS_ADMIN -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/output singularityware/docker2singularity cloverleaf
> ~~~
> {: .language-bash}
>
> To inject a host MPI library, set SINGULARITY_BINDPATH,
> SINGULARITY_CONTAINLIBS, and SINGULARITYENV_RTLD_SUBSTITIONS as
> described in the section for MPI Bandwidth.  There is one additional
> complication as this is a Fortran code, unlike the C MPI Bandwidth.
> The MPICH ABI compatibility is partial since Fortran does not
> mandate symbol names, so you likely will see the following error
> message.
>
> ~~~
> /opt/CloverLeaf/OpenACC/clover_leaf: symbol lookup error: /opt/CloverLeaf/OpenACC/clover_leaf: undefined symbol: mpi_constants_
> ~~~
> {: .output}
>
> One workaround for this case is to ensure that the host MPI library
> was built with the same compiler as was used to build the
> containerized application (in this case, the PGI compiler).
>
> Another workaround is to load a shim library that maps symbols in
> the application to the symbols in the host MPI library.  The Intel
> MPI Library provides a ["binding
> kit"](https://software.intel.com/en-us/mpi-developer-guide-linux-compilers-support)
> for this purpose. The shim library can be injected by setting
> SINGULARITYENV_LD_PRELOAD to point to the shim library.  Details
> will vary depending on your host MPI library.
>
> One final note, not specifically related to containers, is that you
> should ensure that each MPI rank has exclusive access to a GPU. This
> is typically accomplished with a shell script that associates a GPU
> with a MPI rank using the local MPI rank index.
>
> ~~~
> #!/bin/sh
>
> # Bind GPU to corresponding MPI rank
> 
> if [ -n "$MPI_LOCALRANKID" ]; then
>   # Intel MPI
>   RANK=$MPI_LOCALRANKID
> elif [ -n "$MV2_COMM_WORLD_LOCAL_RANK" ]; then
>   # MVAPICH
>   RANK=$MV2_COMM_WORLD_LOCAL_RANK
> else
>   echo "unable to determine local mpi rank index"
> fi
>
> if [ -n "$RANK" ]; then
>   export CUDA_VISIBLE_DEVICES=$RANK
> fi
>
> exec $*
> ~~~
> {: .language-bash}
>
> With the Singularity environment configured to inject MPI from the
> host, you can run the containerized and GPU accelerated CloverLeaf.
> CloverLeaf has very simple input and output and assumes that the
> input file is in the current directory and also writes its output
> file to the current directory.  So first copy the input file to
> current working directory.
>
> ~~~
> $ singularity exec cloverleaf.simg cp /opt/CloverLeaf/OpenACC/clover.in .
> $ mpirun -n 4 --ppn 2 -f hostfile ... singularity exec --nv cloverleaf.simg gpubind.sh /opt/CloverLeaf/OpenACC/clover_leaf
> ~~~
> {: .language-bash}

{% include links.md %}
