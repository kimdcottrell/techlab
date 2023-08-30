---
title: "How to use Traefik Proxy without exposing the Docker socket (HTTP Filter Edition)"
date: 2023-07-21T12:51:11Z
draft: false
tags: ['docker', 'linux', 'devops', 'traefik', 'reverse proxy', 'security', 'localdev', 'local development', 'container']
summary: "Opening up the Docker socket to a container results in the possibility that someone can utilize that container to break into the host machine. This is an example showing how to prevent such a jailbreak - it uses a localdev stack, though the solution for the socket used can also be used in a production environment."
---

# What problem does this solve? 

This post is for anyone who wants to use Traefik as a reverse proxy for their local development environment, but doesn't want to expose their Docker socket to the internet. Now, if you're new to Traefik Proxy, you're probably still accessing your containers like http://127.0.0.1:3421. Traefik allows the assignment of domain names on your local development webserver containers, turning http://127.0.0.1:3421 into http(s)://funlocaldomain.com. It does this by communicating with Docker on your host machine through the Docker socket. 

Letting a container have access to something that has root access to your host machine is A Very Bad Idea (TM). If we were to mount Docker's socket in a remotely accessible environment, under the right conditions, an attacker could do shenanigans like [this](https://blog.quarkslab.com/why-is-exposing-the-docker-socket-a-really-bad-idea.html).

For this specific use case, we technically are sticking to a local-machine-only kind of environment. So nothing is in danger and the following is over-engineered for our use-case. But I am presently unemployed, bored, and this has been a minor thing that has irked me for years. 

So without further ado, let's use Traefik Proxy with a more secure Docker socket setup that filters incoming requests to the Docker Engine API. 

# How does /var/run/docker.sock work?

Under the hood, the Docker socket is merely a Unix socket. Unix sockets are a way for processes to communicate with each other. The Docker socket is a Unix socket that allows processes to communicate with the Docker Engine API. The Docker Engine API is a REST API that allows you to control the Docker daemon.

![I like your funny words magic man](/funny-words-magic-man.gif 'I like your funny words, magic man')

For an example of that word salad above: 

```yaml
# let's start a container in a terminal window
$ docker run -it ubuntu bash

# in a new window, let's run a command that will list all the running containers on our machine via the docker.sock 
$ curl -i -s --unix-socket /var/run/docker.sock -X GET http://localhost/containers/json
HTTP/1.1 200 OK
Api-Version: 1.43
Content-Type: application/json
Docker-Experimental: false
Ostype: linux
Server: Docker/24.0.2 (linux)
Date: Fri, 21 Jul 2023 23:21:46 GMT
Content-Length: 883

[{"Id":"910429d724d845c5c6c9b9ebcd16a6e497af77846eadbd887e03caaa3581d38c","Names":["/distracted_pascal"],"Image":"ubuntu","ImageID":"sha256:5a81c4b8502e4979e75bd8f91343b95b0d695ab67f241dbed0d1530a35bde1eb","Command":"bash","Created":1689981644,"Ports":[],"Labels":{"org.opencontainers.image.ref.name":"ubuntu","org.opencontainers.image.version":"22.04"},"State":"running","Status":"Up About a minute","HostConfig":{"NetworkMode":"default"},"NetworkSettings":{"Networks":{"bridge":{"IPAMConfig":null,"Links":null,"Aliases":null,"NetworkID":"aec05363fd846301f4ced514eaa3912ec0c8fcb6de5de7ff33a0a4764d53a019","EndpointID":"8519151861f405b9ae86bff7ab623086b0563229d0d6678745e29451ca72b915","Gateway":"172.17.0.1","IPAddress":"172.17.0.2","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"MacAddress":"02:42:ac:11:00:02","DriverOpts":null}}},"Mounts":[]}]
```

That `curl` is sending a GET request to `/var/run/docker.sock` to the path `http://it-literally-does-not-matter-what-is-here/containers/json`. That's right: The Docker Engine API is honestly just a funny little REST API [under the hood](https://docs.docker.com/engine/api/v1.43/). We've all been bamboozled. That REST API is also what is powering the `docker` CLI. 

If your container has direct access to that socket, it can do anything that the `docker` CLI can do. _Anything_. This absolutely brilliant and terrifying writeup [here](https://blog.quarkslab.com/why-is-exposing-the-docker-socket-a-really-bad-idea.html) goes into the exact specifics. 

# How do we fix this?

There's a few ways. But for this demonstration, we are going to use a webserver that will sit ontop of the Docker socket and filter incoming requests. We are going to specifically be using [Tecnativa/docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxyhttps://traefik.io/traefik/).

# Notes before we get started

I have been doing my recent localdev shenanigans off an Ubuntu 23.04 box, though I've worked for years on a few different OSX boxes doing similar things. This solutions in this article should be almost 1:1 for OSX, though your milage may vary on Windows. 

The versioning for my local tech stack in this article:

- Ubuntu 23.04
- Docker version 24.0.2
- Docker Compose version v2.18.1

Another thing to note is that I have set Docker to run all containers as my user instead of root by default, via the `userns-remap`. The exact details on what I'm referring to can be found [here](https://docs.docker.com/engine/security/userns-remap/#enable-userns-remap-on-the-daemon). I have been experimenting with trying to make my local environment as close to a prod environment as possible, security restraints and all. Running your containers as less privileged users by default helps deter privilege escalation attacks. However, this does mean that I have to do some extra work to get Traefik Proxy to work, and you will see that in the article.

# Let's do this!

Create something like this dir and these files:

```
./localproxy
├── docker-compose.yml
├── traefik
```

Put this in your `docker-compose.yml`:

```yaml
version: "3.8"

services:
  # This is an optional security container.
  # This will be used to filter ONLY get requests to the Docker Engine API.
  # It stops stuff like https://blog.quarkslab.com/why-is-exposing-the-docker-socket-a-really-bad-idea.html
  socket-proxy:
    image: tecnativa/docker-socket-proxy
    restart: unless-stopped
    privileged: true
    userns_mode: host # this is needed if https://docs.docker.com/engine/security/userns-remap/#enable-userns-remap-on-the-daemon is setup
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      CONTAINERS: 1 # tell the proxy to grant get requests to /containers/* from the Docker API
    networks:
      - default # we only want to be able to access this inside this network's container stack

  traefik:
    image: traefik:v2.10
    restart: unless-stopped
    command: --log.level=DEBUG
    depends_on:
      - socket-proxy
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - ./traefik.yml:/etc/traefik/traefik.yml
    networks:
      - proxy
      - default # run this container on the default network so it can access the socket-proxy container

  whoami:
    image: traefik/whoami
    restart: unless-stopped
    depends_on:
      - socket-proxy
      - traefik
    labels:
      # Explicitly tell Traefik to expose this container
      - traefik.enable=true
      # The domain the service will respond to
      - traefik.http.routers.whoami.rule=Host(`whoami.traefik`)
      # Allow request only from the predefined entry point named "web"
      - traefik.http.routers.whoami.entrypoints=web
    networks:
      - proxy

networks:
  proxy:
```

And put this in your `traefik.yml`:

```yaml
api:
  insecure: true
  dashboard: true
  debug: false

entryPoints:
  web:
    address: ":80"

log:
  format: json
  level: DEBUG

providers:
  docker:
    endpoint: "tcp://socket-proxy:2375"
    watch: true
    exposedbydefault: false
    network: proxy
```

Now, if we were to spin that up right now, we wouldn't actually have that custom domain name working. We could use something like `dnsmasq` here, but let's keep it simple. 

Let's add in that `whoami.traefik` domain to our `/etc/hosts`:

```yaml
127.0.0.1 whoami.traefik
```

Now we can spin up our stack:

```bash
cd ./localproxy
docker compose up
```

And if we go to `http://whoami.traefik` in our browser, we should see something like this:

```yaml
Hostname: 0950b838b55c
IP: 127.0.0.1
IP: 172.26.0.3
RemoteAddr: 172.26.0.2:59812
GET / HTTP/1.1
Host: whoami.traefik
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.5
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 172.25.0.1
X-Forwarded-Host: whoami.traefik
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 0d57a4c14707
X-Real-Ip: 172.25.0.1
```

And if we go to `http://localhost:8080` in our browser, we should get rerouted to the [Traefik dashboard](https://doc.traefik.io/traefik/operations/dashboard/).

# Explaining the setup

There are two major differences to the usual Traefik proxy+Docker container setup here:

- the `socket-proxy` container in the `docker-compose.yml`
- the `providers.docker.endpoint` in the `traefik.yml`

The `socket-proxy` container is our webserver that is sitting on top of the Docker socket. It is filtering out all requests to the Docker API except for `GET` requests to `/containers/*`. That will allow Traefik to get the list of containers and their labels, but not allow it to do anything else. This is a security measure to prevent privilege escalation attacks.

That `socket-proxy` container needs to be how Traefik accesses the Docker API, and that is what the `providers.docker.endpoint` is for. It is telling Traefik to use the `socket-proxy` container as the Docker API endpoint.

Everything else is par for the course. We have a `traefik` container that is running Traefik, and a `whoami` container that is running the [Traefik whoami example](https://doc.traefik.io/traefik/getting-started/quick-start/#step-3-run-a-container).

# Adding a custom domain

In the `docker-compose.yml`, we have this:

```yaml
  whoami:
    labels:
      # The domain the service will respond to
      - traefik.http.routers.whoami.rule=Host(`whoami.traefik`)
```

And it's that line that allows http://whoami.traefik to work when 127.0.0.1 is set as the host. Traefik will watch for requests to that domain on port 80, and handle them accordingly. 

## Debugging incase that custom domain doesn't work

If we did not have `/etc/hosts` working, or if we were having problems with the setup, we could test this out by using the `Host` header in `curl`:

```bash
$ curl -H Host:whoami.traefik http://127.0.0.1
Hostname: c651019dda4e
IP: 127.0.0.1
IP: 172.28.0.3
RemoteAddr: 172.28.0.2:50072
GET / HTTP/1.1
Host: whoami.traefik
User-Agent: curl/7.88.1
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.27.0.1
X-Forwarded-Host: whoami.traefik
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: d34b0d47a2ba
X-Real-Ip: 172.27.0.1
```

For more debugging info, you can check the Traefik dashboard, or the raw data from Traefik available at http://localhost:8080/api/rawdata.

And ofc, you can always check the logs of the `traefik` container:

```bash
docker compose logs traefik
localproxy-traefik-1  | time="2023-07-22T00:04:50Z" level=info msg="Configuration loaded from file: /etc/traefik/traefik.yml"
```

# Conclusion

Do you need to do this? No. This was a combination of boredom and a several years long, extremely minor grievance about a utility on my local machine running with too high of permissions.

Although, if you're working on a remote environment and you NEED to lockdown the Docker socket, similar steps to those in this article are a good way to do it.