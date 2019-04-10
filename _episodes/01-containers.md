---
title: "Software Containers"
teaching: 20
exercises: 0
questions:
- "What are software containers?"
- "How are containers different from Virtual Machines?"
- "What is the container runtime landscape?"
objectives:
- "Learn the basic ideas and concepts behind software containers."
- "Learn about the difference between containers and Virtual machines."
- "Learn about the most popular container solutions."
keypoints:
- "Containers are \"ligh-weight virtual machines.\""
- "Not all container solutions are suited for HPC."
---

## Software dependencies, Reproducibility, and all that jazz

Before we dive deep into the lesson, let's first discuss *why* we even need software containers or
anything like that? Can't we just `make install` everything from source? An honest answer to this
question depends on what we are actually trying to achieve. Without going into too much detail,
issues usually arise when there is a need to:

1. work with multiple operating systems (Windows, macOS, Linux),
2. install conflicting dependencies of an application,
3. make sure that one can repeat the work and obtain the same results
  - updated dependencies / software
  - human factor

The problem related to multiple operating systems is easy to understand and is very common. Imagine
that a program works in Windows, so it knows how to use, say, "circles". In Linux, on the other
hand, there are only rectangles, whereas in macOS - only triangles. And if you move that application
written as a circle and designed to work with other circles into an environment made of rectangles
or triangles -- it will simply not function at all!

![Applications in different operating systems. The rightmost figure
shows Linux application in Windows environment.](../fig/app-different-os-01.png)

The second problem -- conflicting dependencies -- is a bit trickier to avoid as responsible
developers never design their applications with conflicting dependencies. What sometimes happens,
however, is that an application depends on two completely different packages that on _some_
platforms bring in conflicting, circular, diamond and other "bad" dependencies. Certainly, this does
not happen on the platform on which the application is developed, making troubleshooting such issues
more difficult. This problem is sometimes referred to as "[Dependency hell][dep-hell-wiki]".

![Examples of "Dependency Hell": Circular and Diamond dependencies.
In both cases application can not be built.](../fig/dependency-hell-01.png)

Finally, the last item on the list -- "reproducibility" -- is a two-faced problem. Such problems
usually happen when you least expect them. For example, when you (or someone else) repeat the
simulations, calculations, or analyses that were performed a while back and get different results.
This might happen due to a human error (wrong argument, application name, etc.) or due to a
"minor" change in a dependency tree, such as a newer or "patched" version of a program or library.

But can't we solve all of these problems using just virtual machines (VMs)?

## Virtual Machines

Virtual machines are full-fledged emulators of a real computer and deserve a dedicated lesson on
their own. The actual "real" hardware that runs a VM is called a "host" and the emulated machine is
called a "guest". All guest machines are managed by a single hypervisor that ensures that guests can
not affect each other and the host system. There are two types of VMs: Type-1 ("bare metal") and
Type-2 ("hosted") and the main difference between the two is where hypervisor runs. The line between
VM types is very blurry because Type-2 hypervisors such as KVM (Kernel-based Virtual Machine) and
bhyve effectively transform host OS into a Type-1 hypervisor.

![Two types of Virtual Machines.
Type 1: Hypervisor runs on a bare metal and one of the guest
operating systems is "assigned" to be a host.
Type 2: Hypervisor runs inside of a host OS.](../fig/vm-types-01.png)

In "bare metal" VMs, hypervisors run on the hardware and provision resources to all operating
systems that are then launched within them. One of these operating systems is granted special
privileges and is called "host" but other than that, it has exactly the same access to the
underlying hardware as other OSes.

In "hosted" VMs, the system acts as normal and one operating system (called Host Operating System)
has direct access to the hardware. Hypervisor is installed as a program inside of that Host OS and,
therefore, any VM has to be created from within that Host OS. Despite that, overhead associated with
provisioning access to hardware for VMs is minimal.

Regardless of their type, VMs solve the problem of running software under different operating
systems: one can run Linux on a Windows host and vice a versa. However, VMs completely ignore the
latter two of the discussed problems. Moreover, they are not very resource efficient as they emulate
the entire computer with all of its hardware as well as full-blown operating system. And this is
where containers shine!


## Raise of containers

Software containers emerged as "lightweight virtual machines". This definition is not accurate
but it is good enough to understand their main difference with classical VMs.
Container solutions use such Linux kernel features
as Namespaces, CGroups (Control Groups), UnionFS, chroot to run applications in isolated
environments. Here is a schematic illustration of practically any software container solution:

![Container solutions on Linux, Windows, and macOS.
Note that container solutions can run natively on Linux because they only rely on the Linux
kernel. On Windows and Mac container solutions use VM to have access to Linux kernel. Despite common
misconception, macOS run on its own kernel called Darwin.
](../fig/containers-01.png)

The main component of any software container solution is the "Container Engine". Software containers
on Linux differ from their counterparts on Windows and macOS in that they can use the kernel of the
Host operating system. On Windows and Mac, on the other hand, they need a Virtual Machine that
provides them with with access to Linux kernel and the features they require. This figure also
highlights the fact, that all containers will inevitably have the same Linux kernel and can not
emulate Windows environment.

## Container solutions and Docker

Docker is one of the most widely used software container solutions. But containers solutions such as
FreeBSD Jails, Solaris Zones, [LXC][lxc], [LPAR][lpar], and [lmctfy][lmctfy] existed long before
Docker. So, why did Docker become so popular?

- Standardization (container format, API, CLI)
- Support for all major platforms

Standardization led to lower adoption barriers and support for all major platforms allowed Docker to
quickly grow its community in under 4 years. Moreover, Docker provided a complete toolchain for
working with software containers. In the next episode we will learn about the main
concepts and commands of Docker.

[dep-hell-wiki]: https://en.wikipedia.org/wiki/Dependency_hell
[lmctfy]: https://github.com/google/lmctfy
[lpar]: https://en.wikipedia.org/wiki/Logical_partition
[lxc]: https://github.com/lxc/lxc

{% include links.md %}

