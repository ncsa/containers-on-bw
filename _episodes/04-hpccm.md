---
title: "HPCCM: HPC Container Maker"
teaching: 50
exercises: 10
questions:
- "How can one build and run HPC applications in a container?"
objectives:
- "How to use HPC Container Maker to make building container images of HPC applications easier"
keypoints:
- "HPC Container Maker (HPCCM) is an open source tool that makes it easier to build container images for HPC applications"
- "HPCCM building blocks uplevel container specification"
- "HPCCM recipes are Python"
- "HPCCM can generate both Dockerfiles and Singularity definition files"
---

This episode describes how to use [HPC Container
Maker](https://github.com/NVIDIA/hpc-container-maker), a tool to
simplify the process of creating container specification files for
High Performance Computing.

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

## Getting Started
To illustrate this, let's start with a simple example of a container
image that includes CUDA and OpenMPI.

~~~
Stage0 += baseimage(image='nvidia/cuda:9.2-devel-centos7')
Stage0 += openmpi(infiniband=False)
~~~
{: .language-python}

~~~
$ hpccm --recipe openmpi.py
~~~
{: .language-bash}

~~~
FROM nvidia/cuda:9.2-devel-centos7

# OpenMPI version 3.1.2
RUN yum install -y \
        bzip2 \
        file \
        hwloc \
        make \
        numactl-devel \
        openssh-clients \
        perl \
        tar \
        wget && \
    rm -rf /var/cache/yum/*
RUN mkdir -p /var/tmp && wget -q -nc --no-check-certificate -P /var/tmp https://www.open-mpi.org/software/ompi/v3.1/downloads/openmpi-3.1.2.tar.bz2 && \
    mkdir -p /var/tmp && tar -x -f /var/tmp/openmpi-3.1.2.tar.bz2 -C /var/tmp -j && \
    cd /var/tmp/openmpi-3.1.2 &&   ./configure --prefix=/usr/local/openmpi --disable-getpwuid --enable-orterun-prefix-by-default --with-cuda --without-verbs && \
    make -j$(nproc) && \
    make -j$(nproc) install && \
    rm -rf /var/tmp/openmpi-3.1.2.tar.bz2 /var/tmp/openmpi-3.1.2
ENV LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH \
    PATH=/usr/local/openmpi/bin:$PATH
~~~
{: .output}

When this simple two line recipe is processed by HPCCM, the
optimized Dockerfile is generated. Notice that the Dockerfile best
practices described earlier, such as combining related steps into a
single layer and removing temporary files in the same layer they are
generated are automatically employed.

A Singularity definition file can be generated from the exact same
recipe just by specifying the --format command line option.

~~~
$ hpccm --recipe hpccm/openmpi.py --format singularity
~~~
{: .language-bash}

~~~
BootStrap: docker
From: nvidia/cuda:9.2-devel-centos7
%post
    . /.singularity.d/env/10-docker.sh

# OpenMPI version 3.1.2
%post
    yum install -y \
        bzip2 \
        file \
        hwloc \
        make \
        numactl-devel \
        openssh-clients \
        perl \
        tar \
        wget
    rm -rf /var/cache/yum/*
%post
    cd /
    mkdir -p /var/tmp && wget -q -nc --no-check-certificate -P /var/tmp https://www.open-mpi.org/software/ompi/v3.1/downloads/openmpi-3.1.2.tar.bz2
    mkdir -p /var/tmp && tar -x -f /var/tmp/openmpi-3.1.2.tar.bz2 -C /var/tmp -j
    cd /var/tmp/openmpi-3.1.2 &&   ./configure --prefix=/usr/local/openmpi --disable-getpwuid --enable-orterun-prefix-by-default --with-cuda --without-verbs
    make -j$(nproc)
    make -j$(nproc) install
    rm -rf /var/tmp/openmpi-3.1.2.tar.bz2 /var/tmp/openmpi-3.1.2
%environment
    export LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH
    export PATH=/usr/local/openmpi/bin:$PATH
%post
    export LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH
    export PATH=/usr/local/openmpi/bin:$PATH
~~~
{: .output}

HPCCM building blocks are also configurable. The defaults are
suitable for many use cases, but you may need to more precisely
tailor the container image. For example, the [OpenMPI building
block](https://github.com/NVIDIA/hpc-container-maker/blob/master/docs/building_blocks.md#openmpi)
has several configuration options.

For example, this recipe installs OpenMPI in /opt, disables the
Fortran interface and InfiniBand support, and specifies to use
version 2.1.2. Also note that the base image is based on Ubuntu
rather than CentOS, as in the previous recipe; the building block
automatically detected the Linux distribution type and uses apt-get
rather than yum to install its dependencies.

~~~
Stage0 += baseimage(image='nvidia/cuda:9.2-devel-ubuntu16.04')
Stage0 += openmpi(configure_opts=['--disable-getpwuid',
                                  '--enable-orterun-prefix-by-default',
                                  '--disable-fortran'],
                  infiniband=False, prefix='/opt/openmpi', version='2.1.2')
~~~
{: .language-python}

~~~
FROM nvidia/cuda:9.2-devel-ubuntu16.04

# OpenMPI version 2.1.2
RUN apt-get update -y && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        bzip2 \
        file \
        hwloc \
        libnuma-dev \
        make \
        openssh-client \
        perl \
        tar \
        wget && \
    rm -rf /var/lib/apt/lists/*
RUN mkdir -p /var/tmp && wget -q -nc --no-check-certificate -P /var/tmp https://www.open-mpi.org/software/ompi/v2.1/downloads/openmpi-2.1.2.tar.bz2 && \
    mkdir -p /var/tmp && tar -x -f /var/tmp/openmpi-2.1.2.tar.bz2 -C /var/tmp -j && \
    cd /var/tmp/openmpi-2.1.2 &&   ./configure --prefix=/opt/openmpi --disable-getpwuid --enable-orterun-prefix-by-default --disable-fortran --with-cuda --without-verbs && \
    make -j$(nproc) && \
    make -j$(nproc) install && \
    rm -rf /var/tmp/openmpi-2.1.2.tar.bz2 /var/tmp/openmpi-2.1.2
ENV LD_LIBRARY_PATH=/opt/openmpi/lib:$LD_LIBRARY_PATH \
    PATH=/opt/openmpi/bin:$PATH
~~~
{: .output}

## Using MPI

The [MPI Bandwidth sample
program](https://computing.llnl.gov/tutorials/mpi/samples/C/mpi_bandwidth.c)
from the Lawrence Livermore National Laboratory (LLNL) will be used
as a proxy application to illustrate how to use HPCCM recipes to
create application containers and how MPI can be used with
Singularity.

For system with COTS (commercial off the shelf) network fabrics such
as Ethernet or InfiniBand, you would use the MPI library embedded in
the container. The OpenMPI library detects the best available
network fabric and automatically uses it, so the same container can
be deployed on most COTS systems regardless of the network fabric.

The MPI Bandwidth for COTS clusters includes the mlnx_ofed building
block to install the Mellanox OpenFabrics Enterprise Distribution
(OFED) user space software and a IB-enabled version of OpenMPI.

~~~
Stage0 += baseimage(image='centos:7')
Stage0 += gnu(fortran=False)
Stage0 += mlnx_ofed()
Stage0 += openmpi(cuda=False, version='3.0.0')

# MPI Bandwidth
Stage0 += shell(commands=[
    'wget -q -nc --no-check-certificate -P /var/tmp https://computing.llnl.gov/tutorials/mpi/samples/C/mpi_bandwidth.c',
    'mpicc -o /usr/local/bin/mpi_bandwidth /var/tmp/mpi_bandwidth.c'])

Stage1 += baseimage(image='centos:7')
Stage1 += Stage0.runtime()

# MPI Bandwidth
Stage1 += copy(_from='0', src='/usr/local/bin/mpi_bandwidth',
               dest='/usr/local/bin/mpi_bandwidth')
~~~
{: .language-python}

For systems with non-COTS network fabrics that require a vendor MPI
library in order to realize full performance, such as the Blue
Waters system at NCSA, a different container approach is
required. MPI libraries derived from MPICH are [ABI
compatible](https://www.mpich.org/abi/). In other words, the MPI
library used to build an application may be "swapped" out with
another MPI library at runtime.

The outline of the recipe is the same - include compiler and MPI
building blocks, and build the MPI Bandwidth application - but some
of the specifics differ.  A "vanilla" version of MPICH is used to
build the MPI Bandwidth application since it will just be swapped
out with an optimized MPI library at runtime. The
[dl-intercept](https://github.com/olcf/dl-intercept) library is also
included in the container to make the swapping process easier.  OFED
can be omitted since system specific fabric libraries will also be
mapped into the container at runtime.

~~~
from hpccm.templates.CMakeBuild import CMakeBuild
from hpccm.templates.git import git

Stage0 += baseimage(image='centos:7')
Stage0 += gnu()
Stage0 += mpich()

# dl-intercept
Stage0 += boost()
Stage0 += cmake(eula=True)
Stage0 += packages(ospackages=['ca-certificates', 'git', 'libstdc++-static'])
cm = CMakeBuild()
Stage0 += shell(commands=[
  git().clone_step(repository='https://github.com/olcf/dl-intercept',
                   branch='v1.0.2', path='/var/tmp'),
  cm.configure_step(directory='/var/tmp/dl-intercept',
                    opts=['-DCMAKE_INSTALL_PREFIX=/usr/local',
                          '-DCMAKE_BUILD_TYPE=RELEASE',
                          '-DBOOST_ROOT=/usr/local/boost']),
  cm.build_step(),
  cm.build_step(target='install')])

# MPI Bandwidth
Stage0 += shell(commands=[
    'wget -q -nc --no-check-certificate -P /var/tmp https://computing.llnl.gov/tutorials/mpi/samples/C/mpi_bandwidth.c',
    'mpicc -o /usr/local/bin/mpi_bandwidth /var/tmp/mpi_bandwidth.c'])

Stage1 += baseimage(image='centos:7')
Stage1 += Stage0.runtime(exclude=['boost'])

# dl-intercept
Stage1 += copy(_from='0', dest='/usr/local/lib/libdl-intercept.so',
               src='/usr/local/lib/libdl-intercept.so')
Stage1 += environment(
  variables={'LD_AUDIT': '/usr/local/lib/libdl-intercept.so'})

# MPI Bandwidth
Stage1 += copy(_from='0', src='/usr/local/bin/mpi_bandwidth',
               dest='/usr/local/bin/mpi_bandwidth')
~~~
{: .language-python}

Build the container image using Docker and then convert it into a
Singularity image.

~~~
$ hpccm --recipe mpi_bandwidth.py > Dockerfile
$ docker build -t mpi_bandwidth -f Dockerfile .
$ sudo docker run -t --rm --cap-add SYS_ADMIN -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/output singularityware/docker2singularity mpi_bandwidth
~~~
{: .language-bash}

To inject a MPI library from the host into a Singularity container,
3 things are necessary. First, the directory containing the host MPI
library must be mounted into the container.  Second, any network
fabric library dependencies or other MPI library dependencies must
also be injected into the container.  And finally, the environment
needs to be configured for the application to use the injected MPI
library rather than the version inside the container.  All 3 of
these items can be accomplished by setting a few environment
variables.

The SINGULARITY_BINDPATH environment variable mounts the specified
paths inside the container at the same location. Corresponding mount
points do not need to preexist inside the container.  For instance,
if your host software environment is installed in /opt/sw, you would
set SINGULARITY_BINDPATH=/opt/sw to mount it inside the container.

The SINGULARITY_CONTAINLIBS environment variable injects the
specified libraries into the container.  The libraries are injected
in /.singularity.d/libs.  The location is largely immaterial as this
directory is automatically added to LD_LIBRARY_PATH by Singularity.
Typically only libraries from system paths such as /lib64 need to be
injected this way if the entire host software directory is mounted
using SINGULARITY_BINDPATH.  You may need to extend
SINGULARITY_BINDPATH to include configuration or other files that
are installed in system locations such as /etc.

The RTLD_SUBSTITUTIONS environment variable, recognized by the
dl-intercept library, performs the library swap. Environment
variables with the SINGULARITYENV_ prefix are set by Singularity
only inside the container.

To inject the Intel MPI Library from the host into the container on
an InfiniBand cluster where the host software environment is
installed in /opt/sw, the environment would be similar to the
following.

~~~
$ export SINGULARITY_BINDPATH="/opt/sw,/etc/libibverbs.d,/etc/dat.conf"
$ export SINGULARITY_CONTAINLIBS="/lib64/libnuma.so.1,/lib64/libibverbs.so.1,/lib64/libdat2.so.2,/lib64/libnl-3.so.200,/lib64/libnl-route-3.so.200,/lib64/libmlx4-rdmav2.so,/lib64/libmlx5-rdmav2.so,/lib64/librxe-rdmav2.so,/lib64/libdaploucm.so.2,/lib64/libdaploscm.so.2,/lib64/libdaplofa.so.2"
$ export SINGULARITYENV_RTLD_SUBSTITUTIONS="libmpi.so.12:/opt/sw/intel/compilers_and_libraries_2018.1.163/linux/mpi/intel64/lib/libmpi.so.12"
~~~
{: .language-bash}

The system administrator could prepopulate these environment settings
in an environment module or in /etc/singularity/singularity.conf to
simplify the process.

With those environment variables set, you can run the containerized
version of MPI Bandwidth using the MPI library injected from the
host using a typical mpirun + Singularity command line.

~~~
$ mpirun -n 2 -f hostfile ... singularity exec mpi_bandwidth.simg /usr/local/bin/mpi_bandwidth
~~~
{: .language-bash}

## Using GPUs

The [CUDA STREAM](https://github.com/bcumming/cuda-stream) benchmark
will be used a proxy application to illustrate how to use a HPCCM
recipe to create application containers and how GPUs can used with
Singularity.

Using GPUs with Singularity is much simpler than injecting MPI from
the host.  Just use the --nv option with Singularity and Singularity
will automatically handle injecting the GPU libraries from the host
inside the container.

Building containers to support GPUs is simple as well.  Just use the
CUDA containers provided free of charge by NVIDIA on Docker Hub.

The CUDA development container already includes the necessary
compilers, so just install a few packages necessary to clone a git
repository and the recipe is ready to build CUDA STREAM.

~~~
from hpccm.templates.git import git

Stage0 += baseimage(image='nvidia/cuda:9.1-devel-centos7')

# CUDA STREAM
Stage0 += packages(ospackages=['ca-certificates', 'git'])
Stage0 += shell(commands=[
  git().clone_step(repository='https://github.com/bcumming/cuda-stream.git',
                   path='/var/tmp'),
  'cd /var/tmp/cuda-stream',
  'nvcc -std=c++11 -ccbin=g++ -gencode arch=compute_35,code=\\"sm_35,compute_35\\" -o stream stream.cu'])

Stage1 += baseimage(image='nvidia/cuda:9.1-base-centos7')

Stage1 += copy(_from='0', dest='/usr/local/bin/stream',
               src='/var/tmp/cuda-stream/stream')

Stage1 += runscript(commands=['/usr/local/bin/stream'])
~~~
{: .language-python}

Build the container image using Docker and then convert it into a
Singularity image.

~~~
$ hpccm --recipe cuda_stream.py > Dockerfile
$ docker build -t cuda_stream -f Dockerfile .
$ sudo docker run -t --rm --cap-add SYS_ADMIN -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/output singularityware/docker2singularity cuda_stream
~~~
{: .language-bash}

Just add --nv to enable GPU support in Singularity.

~~~
$ singularity run --nv cuda_stream.simg
~~~
{: .language-bash}

## Using MPI and GPUs

The [CloverLeaf](http://uk-mac.github.io/CloverLeaf/) mini-app will
be used as a proxy application to illustrate how to use a HPCCM
recipe to create a container supporting GPUs and injecting MPI from
the host with Singularity. Several implementations of CloverLeaf
are available; the MPI+OpenACC version will be used here.

As a Fortran code using OpenACC, the PGI compiler will be used to
compile it.  As in the MPI Bandwidth example, a "vanilla" MPICH and
the dl-intercept library will be used to facilitate injecting a host
MPI library at runtime. The HPCCM recipe is similar to the MPI
Bandwidth case, but with MPICH built using the PGI compiler. As in
the CUDA STREAM case, the CUDA Docker Hub base images will be used
for GPU support.

~~~
from hpccm.templates.CMakeBuild import CMakeBuild
from hpccm.templates.git import git
from hpccm.templates.sed import sed

Stage0 += baseimage(image='nvidia/cuda:9.1-devel-centos7')

# Compilers - will use PGI but need GNU as well
Stage0 += gnu()
compiler = pgi(eula=True)
Stage0 += compiler

# MPICH - build using the PGI compiler
Stage0 += mpich(toolchain=compiler.toolchain)

# dl-intercept
Stage0 += boost()
Stage0 += cmake(eula=True)
Stage0 += packages(ospackages=['ca-certificates', 'git', 'libstdc++-static'])
cm = CMakeBuild()
Stage0 += shell(commands=[
  git().clone_step(repository='https://github.com/olcf/dl-intercept',
                   branch='v1.0.2', path='/var/tmp'),
  'cd /var/tmp/dl-intercept',
  cm.configure_step(directory='/var/tmp/dl-intercept',
                    opts=['-DCMAKE_INSTALL_PREFIX=/usr/local',
                          '-DCMAKE_BUILD_TYPE=RELEASE',
                          '-DBOOST_ROOT=/usr/local/boost']),
  cm.build_step(),
  cm.build_step(target='install')])

# CloverLeaf OpenACC
Stage0 += shell(commands=[
  git().clone_step(repository='https://github.com/UK-MAC/CloverLeaf_OpenACC',
                   branch='master', path='/var/tmp'),
  # Build for all compute capabilities for broadest GPU support
  sed().sed_step(file='/var/tmp/CloverLeaf_OpenACC/Makefile',
                 patterns=[r's/cc35/ccall/g']),
  'cd /var/tmp/CloverLeaf_OpenACC',
  'COMPILER=PGI make'])

Stage1 += baseimage(image='nvidia/cuda:9.1-base-centos7')
Stage1 += Stage0.runtime(exclude=['boost'])

# dl-intercept
Stage1 += copy(_from='0', dest='/usr/local/lib/libdl-intercept.so',
               src='/usr/local/lib/libdl-intercept.so')
Stage1 += environment(
  variables={'LD_AUDIT': '/usr/local/lib/libdl-intercept.so'})

# CloverLeaf
Stage1 += copy(_from='0', dest='/opt/CloverLeaf/OpenACC/clover_leaf',
               src='/var/tmp/CloverLeaf_OpenACC/clover_leaf')
Stage1 += copy(_from='0', dest='/opt/CloverLeaf/OpenACC/InputDecks',
               src='/var/tmp/CloverLeaf_OpenACC/InputDecks')
Stage1 += shell(commands=['ln -s /opt/CloverLeaf/OpenACC/InputDecks/clover_bm16_short.in /opt/CloverLeaf/OpenACC/clover.in'])
~~~
{: .language-python}

Building the container image follows the same workflow as the
previous cases: generate a Dockerfile from the HPCCM recipe, build a
Docker container image, and then convert it to a Singularity
container image.

~~~
$ hpccm --recipe cloverleaf.py > Dockerfile
$ docker build -t cloverleaf -f Dockerfile .
$ sudo docker run -t --rm --cap-add SYS_ADMIN -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/output singularityware/docker2singularity cloverleaf
~~~
{: .language-bash}

To inject a host MPI library, set SINGULARITY_BINDPATH,
SINGULARITY_CONTAINLIBS, and SINGULARITYENV_RTLD_SUBSTITIONS as
described in the section for MPI Bandwidth.  There is one additional
complication as this is a Fortran code, unlike the C MPI Bandwidth.
The MPICH ABI compatibility is partial since Fortran does not
mandate symbol names, so you likely will see the following error
message.

~~~
/opt/CloverLeaf/OpenACC/clover_leaf: symbol lookup error: /opt/CloverLeaf/OpenACC/clover_leaf: undefined symbol: mpi_constants_
~~~
{: .output}

One workaround for this case is to ensure that the host MPI library
was built with the same compiler as was used to build the
containerized application (in this case, the PGI compiler).

Another workaround is to load a shim library that maps symbols in
the application to the symbols in the host MPI library.  The Intel
MPI Library provides a ["binding
kit"](https://software.intel.com/en-us/mpi-developer-guide-linux-compilers-support)
for this purpose. The shim library can be injected by setting
SINGULARITYENV_LD_PRELOAD to point to the shim library.  Details
will vary depending on your host MPI library.

One final note, not specifically related to containers, is that you
should ensure that each MPI rank has exclusive access to a GPU. This
is typically accomplished with a shell script that associates a GPU
with a MPI rank using the local MPI rank index.

~~~
#!/bin/sh

# Bind GPU to corresponding MPI rank

if [ -n "$MPI_LOCALRANKID" ]; then
  # Intel MPI
  RANK=$MPI_LOCALRANKID
elif [ -n "$MV2_COMM_WORLD_LOCAL_RANK" ]; then
  # MVAPICH
  RANK=$MV2_COMM_WORLD_LOCAL_RANK
else
  echo "unable to determine local mpi rank index"
fi

if [ -n "$RANK" ]; then
  export CUDA_VISIBLE_DEVICES=$RANK
fi

exec $*
~~~
{: .language-bash}

With the Singularity environment configured to inject MPI from the
host, you can run the containerized and GPU accelerated CloverLeaf.
CloverLeaf has very simple input and output and assumes that the
input file is in the current directory and also writes its output
file to the current directory.  So first copy the input file to
current working directory.

~~~
$ singularity exec cloverleaf.simg cp /opt/CloverLeaf/OpenACC/clover.in .
$ mpirun -n 4 --ppn 2 -f hostfile ... singularity exec --nv cloverleaf.simg gpubind.sh /opt/CloverLeaf/OpenACC/clover_leaf
~~~
{: .language-bash}

{% include links.md %}
