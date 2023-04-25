---
title: "Fixing Docker permissions for containers on Linux"
date: 2023-04-25T14:34:18Z
draft: false
---

# What does this solve?

Docker on Windows and OSX runs ontop a VM. That VM usually ensures that your local machine's UID and GID and the container UID and GID work fine out of the box.

For Linux, there is no VM layer. Docker instead merely runs by taking advantage of kernel resources. Whilst this results in a much more efficient and speedy performance, this can result in odd behaviors, like you not having permissions inside the container to edit pre-existing files and folders that have been mounted into your container.

# Example code for Docker that we're assuming in this use case

**Dockerfile**

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.20-alpine3.16 AS base

RUN apk --no-cache add bash bash-completion tzdata make hugo \
    && find /tmp -mindepth 1 -maxdepth 1 | xargs rm -rf

WORKDIR /src

SHELL ["bash"]

FROM base as dev

ENTRYPOINT ["hugo", "server", "-DEF", "--watch=true", "--bind=0.0.0.0", "--baseURL=http://0.0.0.0:1313"]
EXPOSE 1313
```

**docker-compose.yml**

```yaml
version: "3.9"
services:
  project:
    build:
      context: ./cicd/containers
      target: dev
    environment:
      UMASK: 0002
    volumes:
      - ./src:/src # we assume that a hugo project has already been created 
    ports:
      - "1313:1313" # the end goal should allow us to use http://localhost:1313 to access our hugo dev site
```

# Let's fix it!

There are many ways to fix this, but this article will specifically go over the most portable one. I wanted a fix that would not prevent someone on a different machine from being able to spin up my Dockerfile. 

We are going to use the [user namespace map](https://docs.docker.com/engine/security/userns-remap/) configuration for the Docker daemon.

You'll want to check what user and group you are currently running as.

```bash
kim@local:~/Personal/techlab/src $ whoami
kim
kim@local:~/Personal/techlab/src $ id -u && id -g
1000
1000
```

Then you'll edit these two files so those id's fit in the range inside the file.

```bash
kim@local:~/Personal/techlab/src $ cat /etc/subuid
kim:1000:65536
kim@local:~/Personal/techlab/src $ cat /etc/subgid
kim:1000:65536
```

Then tell docker that you want to use a specific user namespace for your containers.

```bash
kim@local:~/Personal/techlab/src $ cat /etc/docker/daemon.json 
{
	"userns-remap": "kim"
}
```

Validate the configuration and restart docker.

```bash
kim@local:~/Personal/techlab/src $ dockerd --validate --config-file /etc/docker/daemon.json 
configuration OK
kim@local:~/Personal/techlab/src $ systemctl restart docker
```

Check that docker now has that namespace map that shows the uid and gid (for me, this was 1000.1000, respectively)

```bash
kim@local:~/Personal/techlab/src $ sudo ls -la /var/lib/docker/1000.1000
total 52
drwx--x--- 12 root kim  4096 Apr 25 13:21 .
drwx--x--- 15 root kim  4096 Apr 25 12:25 ..
drwx--x--x  5 root root 4096 Apr 25 12:27 buildkit
drwx--x---  9 root kim  4096 Apr 25 13:23 containers
-rw-------  1 root root   36 Apr 25 12:25 engine-id
drwx------  3 root root 4096 Apr 25 12:25 image
drwxr-x---  3 root root 4096 Apr 25 12:25 network
drwx--x--- 26 root kim  4096 Apr 25 13:23 overlay2
drwx------  4 root root 4096 Apr 25 12:25 plugins
drwx------  2 root root 4096 Apr 25 13:21 runtimes
drwx------  2 root root 4096 Apr 25 12:25 swarm
drwx------  2 root root 4096 Apr 25 13:23 tmp
drwx-----x  2 root root 4096 Apr 25 13:21 volumes
```

And after all that, build your image and double check that the root user inside the machine now successfully can create files and folders in such a way that on your local machine, those files and folders show up as your present user.

```bash
kim@local:~/Personal/techlab/src $ touch from-local-machine
kim@local:~/Personal/techlab/src $ ls -la
total 48
drwxr-s--- 11 kim kim 4096 Apr 25 13:40 .
drwxrwxr-x  6 kim kim 4096 Apr 24 17:46 ..
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 archetypes
-rw-r--r--  1 kim kim  101 Apr 24 17:52 config.toml
drwxr-xr-x  3 kim kim 4096 Apr 25 10:34 content
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 data
-rw-rw-r--  1 kim kim    0 Apr 25 13:40 from-local-machine
-rw-r--r--  1 kim kim    0 Apr 25 10:34 .hugo_build.lock
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 layouts
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 public
drwxr-xr-x  3 kim kim 4096 Apr 25 10:34 resources
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 static
drwxr-xr-x  3 kim kim 4096 Apr 24 17:46 themes

kim@local:~/Personal/techlab/cicd/containers $ docker build --target base -t techlab .
kim@local:~/Personal/techlab/cicd/containers $ docker run -v ~/Personal/techlab/src:/src -it techlab bash
bash-5.1# touch from-inside-container
bash-5.1# ls -la
total 48
drwxr-s---   11 root     root          4096 Apr 25 17:40 .
drwxr-xr-x    1 root     root          4096 Apr 25 17:39 ..
-rw-r--r--    1 root     root             0 Apr 25 14:34 .hugo_build.lock
drwxr-xr-x    2 root     root          4096 Apr 24 21:43 archetypes
-rw-r--r--    1 root     root           101 Apr 24 21:52 config.toml
drwxr-xr-x    3 root     root          4096 Apr 25 14:34 content
drwxr-xr-x    2 root     root          4096 Apr 24 21:43 data
-rw-r--r--    1 root     root             0 Apr 25 17:40 from-inside-container
-rw-rw-r--    1 root     root             0 Apr 25 17:24 from-local-machine
drwxr-xr-x    2 root     root          4096 Apr 24 21:43 layouts
drwxr-xr-x    2 root     root          4096 Apr 24 21:43 public
drwxr-xr-x    3 root     root          4096 Apr 25 14:34 resources
drwxr-xr-x    2 root     root          4096 Apr 24 21:43 static
drwxr-xr-x    3 root     root          4096 Apr 24 21:46 themes
bash-5.1# exit

kim@local:~/Personal/techlab/cicd/containers $ cd ../..
kim@local:~/Personal/techlab $ cd src
kim@local:~/Personal/techlab/src $ ls -la
total 48
drwxr-s--- 11 kim kim 4096 Apr 25 13:40 .
drwxrwxr-x  6 kim kim 4096 Apr 24 17:46 ..
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 archetypes
-rw-r--r--  1 kim kim  101 Apr 24 17:52 config.toml
drwxr-xr-x  3 kim kim 4096 Apr 25 10:34 content
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 data
-rw-r--r--  1 kim kim    0 Apr 25 13:40 from-inside-container
-rw-rw-r--  1 kim kim    0 Apr 25 13:40 from-local-machine
-rw-r--r--  1 kim kim    0 Apr 25 10:34 .hugo_build.lock
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 layouts
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 public
drwxr-xr-x  3 kim kim 4096 Apr 25 10:34 resources
drwxr-xr-x  2 kim kim 4096 Apr 24 17:43 static
drwxr-xr-x  3 kim kim 4096 Apr 24 17:46 themes
```

The output above shows that both `from-*` files are showing up on the local machine and inside the container correctly. 

And you'd do this in lieu of adding a group and user inside the Dockerfile since that approach may backfire as soon as the Dockerfile is spun up on a different linux machine and that user's uid and gid are different. In other words, this approach is more portable across different operating systems - it provides a way for multiple linux machines to play nice, as well as retaining Windows and OSX compatibility.