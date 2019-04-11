---
title: "Singularity"
teaching: 60
exercises: 10
questions:
- "How can one build and run HPC applications in a container?"
objectives:
- "Build container images using Singularity"
- "'Flat' Singularity images"
keypoints:
- "Singularity images are 'flat'"
- "'Flat' images have runtime advantages"
- "Docker images can be easily converted to Singularity images"
---

## Building Container Images with Singularity

This part of the episode covers how to [build container images with
Singularity](https://www.sylabs.io/guides/latest/user-guide/build_a_container.html).  Administrative
privileges are required to build Singularity container images.  In contrast to Docker, running
Singularity containers does not require administrative privileges.  By default, Singularity uses a
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

A quick note of the `singularity build` command line. The first argument is the filename of the
resulting container image. By convention, Singularity 2.x container image files have the `.simg`
extension, while Singularity 3.x container images have the `.sif` extension. The second argument is
the path to the Singularity definition file.

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
{: .output}

> ## Hello World!
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
> {: .source}
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
> {: .source}
>
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
> > {: .source}
> >
> > Note that the apt package cache is removed to minimize the image size.
> {: .solution}
>
> Verify your solution by running the Hello World program inside the
> container.
>
> > ## Solution
> > ~~~
> > $ singularity exec hello-world.simg /usr/local/bin/hello
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > Hello World!
> > ~~~
> > {: .output}
> {: .solution}
{: .challenge}

Let's take a closer look at the Hello World container image.

~~~
ls -lh hello-world.simg
~~~
{: .language-bash}

~~~
-rwxr-xr-x. 1 root root 93M Jan 23 22:21 hello-world.simg
~~~
{: .output}

The image first-image.simg, which was essentially just the base Ubuntu container image, is 36
megabytes. The Hello World program itself is less than 10 kilobytes, yet the Hello World container
image is 93 megabytes, 2.5 times the size of the base image. The compiler accounts for over half of
the total container size! But all we really care about is the Hello World program, there is no need
to redistribute the compiler (or our source code) to users of the container image.

You could reduce the size of the Singularity container image by
removing the source code and compiler after the Hello World program has been built. Doing so would
reduce the container image size back to 36 megabytes. However, more complex programs with runtime
dependencies will require more sophisticated steps to remove unnecessary components while
maintaining the needed runtime dependencies.  HPC Container Maker can handle runtime dependencies
automatically.

The Singularity "flat" image format is optimal for running a
container, but the unlike Docker, Singularity does not have the concepts of image layers, caches, or
multi-stage containers.  The process of building "layered" images is more flexible and can produce
significantly smaller container images.  Luckily, Singularity can easily convert layered Docker
images into flat Singularity images.

{% include links.md %}
