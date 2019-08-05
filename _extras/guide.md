---
title: "Instructor Notes"
---

## Docker on Linux

On Linux, you have to prefix all `docker` commands with `sudo` because Docker requires
administrative privileges. If you would like to avoid having to type `sudo` with every Docker
command, follow these steps (from [here][docker-postinstall]):

~~~
sudo groupadd docker         # Create 'docker' group
sudo gpasswd -a $USER docker # Add current user to the group
sudo service docker restart  # Restart Docker daemon
exit                         # Log out
~~~
{: .language-bash}

[docker-postinstall]: https://docs.docker.com/install/linux/linux-postinstall/

{% include links.md %}
