---
title: Setup
---


## Docker Hub account

To work with Docker we need an account on [Docker Hub][dockerhub]. If you don't have an account with
Docker Hub yet, you can easily sign-up for one [here][dockerhub-signup].

## Docker CE (Community Edition)

Please install Docker for your platform following the instructions for your platform:

- [macOS][docker-mac]
- [Windows][docker-windows]
- Linux:
  - [CentOS][docker-centos]
  - [Debian][docker-debian]
  - [Fedora][docker-fedora]
  - [Ubuntu][docker-ubuntu]
  - [Binaries][docker-binaries]

Please consider following optional post-installation steps on Linux. If you don't follow them,
you will have to prefix all of the `docker` commands with `sudo`.

## Singularity

Please install Singularity following the instructions for your platform from their
[User Guide][singularity-user-guide]:

- [macOS][singularity-mac]
- [Windows][singularity-windows]
- [Linux][singularity-linux]

## Python

HPC Container Maker is a Python package and, as such, requires a working version of Python. The
latest version of Python can be found on the [official website][python-downloads].

[dockerhub]: https://hub.docker.com/
[dockerhub-signup]: https://hub.docker.com/signup
[docker-mac]: https://docs.docker.com/docker-for-mac/install/
[docker-windows]: https://docs.docker.com/docker-for-windows/install/
[docker-centos]: https://docs.docker.com/install/linux/docker-ce/centos/
[docker-debian]: https://docs.docker.com/install/linux/docker-ce/debian/
[docker-fedora]: https://docs.docker.com/install/linux/docker-ce/fedora/
[docker-ubuntu]: https://docs.docker.com/install/linux/docker-ce/ubuntu/
[docker-binaries]: https://docs.docker.com/install/linux/docker-ce/binaries/
[docker-postinstall]: https://docs.docker.com/install/linux/linux-postinstall/
[singularity-user-guide]: https://www.sylabs.io/guides/3.1/user-guide/
[singularity-mac]: https://www.sylabs.io/guides/3.1/user-guide/installation.html#mac
[singularity-windows]: https://www.sylabs.io/guides/3.1/user-guide/installation.html#windows
[singularity-linux]: https://www.sylabs.io/guides/3.1/user-guide/installation.html#install-on-linux
[python-downloads]: https://www.python.org/downloads/

{% include links.md %}
