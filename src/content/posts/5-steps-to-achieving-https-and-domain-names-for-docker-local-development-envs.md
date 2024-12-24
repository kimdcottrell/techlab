---
title: "5 simple steps to achieving https and domain names for Docker localdev environments"
date: 2023-08-30T21:26:05Z
lastmod: 2024-12-24T13:50:31Z
draft: false
tags: ['docker', 'linux', 'devops', 'container', 'traefik', 'reverse proxy', 'https', 'local development', 'localdev', 'containerization', '443', 'domain name']
summary: "Sometimes, Docker's usual `http://1.2.3.4:8080` syntax is just fine for localdev. Othertimes, it's not. Let's explore when you'll need to change things up, why you need to do so, and how to easily achieve the next step up: https-enabled domain names for your local environments."
---

# HTTPs and domain names for local development environments

Sometimes, Docker's usual `http://1.2.3.4:8080` syntax is just fine for localdev. Othertimes, it's not. Let's explore when you'll need to change things up, why you need to do so, and how to easily achieve the next step up. 

The project that inspired this article is available here: https://github.com/kimdcottrell/localdev-proxy 

# Updates to this article

- Updating article code so the traefik image pulls from 3.2 instead of 2.10
- The reference repo has been updated to use `traefik:v3.2`

# Why and when you need to go down the `https://mysite.local.dev` route

Let's take a step back. If you're just writing a file using HTML, CSS, and JS, why wouldn't you just load that file up in your browser using the `file://` protocol and doing all your development that way?

Put simply: Unless you're making a webpage simply for yourself on your local machine, your endgame is to serve your code to the public via a website. A website is served either via `http://` or `https://`, and different Javscript functionalities in your browser, and in your file, will be accessible only when accessing your webpage via those `http*://` protocols. 

Using docker and a webserver container is a step above that. You'll at least get your pages served via the `http://` protocol. However, if you're in the professional webdev space, you'll _quickly_ discover Docker's `http://1.2.3.4:8080` syntax is just not good enough. A number of vendor API's require that you're accessing them under at least a domain name, if not a domain name served via https, aka on port 443. And don't even get me started on how your browser will react if you [ever dare to typo](https://stackoverflow.com/questions/73875589/disable-website-redirection-to-https-on-chrome) `https://` before a local domain name that is only served on port 80...

If you plan on using Docker for local development on websites that interact with things like payment processors or ecommerce vendors, you _require_ a reverse proxy. Local development for websites that interact with payment processors or ecommerce vendors commonly require a domain name and https. Using a reverse proxy will make these things possible:

1. access to your local development environment webserver via a domain name in your browser, e.g. `http://mysite.local.dev`
2. serving pages locally via https, e.g. `https://mysite.local.dev`

# What's an easy way to setup a reverse proxy for local development? 

Let's solve for one problem at a time, in 5 simple steps. 

All the code from this article stems from the development of this repo: https://github.com/kimdcottrell/localdev-proxy

We'll be using [Traefik Proxy](https://doc.traefik.io/traefik/) for this. There are others, such as [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy), though I personally prefer Traefik for the inherent Englishness it brings to the table as well as how it stays within the bounds of yaml that I'm used to after working with Docker so much. 

## Step 1: Setting up the Traefik container stack for port 80 connections

Within a separate, new `docker-compose.yml` file outside of your application's, you'll need to grab the traefik container and specify an external network within Docker's pervue at a minimum.

```yaml
services:
  traefik:
    image: traefik:v3.2
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./traefik.yml:/etc/traefik/traefik.yml
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      # Explicitly tell Traefik to expose this container
      - traefik.enable=true
      # The domain the service will respond to
      - traefik.http.routers.reverseproxy.rule=Host(`admin.traefik`)
      # Allow request only from the predefined entry point named "web"
      - traefik.http.routers.reverseproxy.entrypoints=web
      # Traefik's own application is served on port 8080, so point the loadbalancer to there
      - traefik.http.services.reverseproxy.loadbalancer.server.port=8080
    networks:
      - proxy

networks:
  proxy:
    name: proxy
```

On `docker compose up` (**do not run this command right now, it won't work**), this will mount the Docket socket, letting Traefik learn about all the running containers on your host machine. If this was being done on a production environment, I'd flag that as a security risk, and one that can be dealt with [like so](/posts/how-to-use-traefik-proxy-without-docker-socket-exposed-http-filter-edition/). However, that's a bit out of scope for this. 

It will, of course, also start the traefik container with it's config file available at `traefik.yml`. Tho it will also start a network within Docker's pervue called `proxy`. That network is what our application's webserver will connect to in order to allow Traefik to work its magic. 

We'll need to make that `traefik.yml` file though. It should contain the following:

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
    endpoint: "unix:///var/run/docker.sock"
    watch: true
    exposedbydefault: false
    network: proxy 
```

Careful though! We're not quite done. Edit your local machine's `/etc/hosts` file to include this line:

```
127.0.0.1  admin.traefik
```

Finally, with that in place, we're good to start the traefik container. Run a `docker compose up` and in your browser, type in `http://admin.traefik`. You should see a pretty little admin page. Or you'll see your browser trying to be helpful and error out as it attempts to redirect you to `https://admin.traefik`. In which case, don't panic, just test that you're seeing some html in a curl via `curl -H 'Host: admin.traefik' 127.0.0.1`. 

## Step 2: Hooking in your application

In your application's `docker-compose.yml`, include the following to your webserver container(s). Be sure to correlate the loadbalancer port setting with that which is running from inside the webserver container - aka what is essentially in it's Dockerfile under the `EXPOSE` setting. 

**NOTE: the `...`'s are simply to mean "your other, pre-existing configuration settings".**

```yaml
# Changeable substrings: `webserverlocaldev`, `webserver`, `webserver.local.dev`, `8001`
services:
  webserver:
    ...
    labels:
      # Explicitly tell Traefik to expose this container
      - traefik.enable=true
      # The domain the service will respond to, and what is in your /etc/hosts
      - traefik.http.routers.webserverlocaldev.rule=Host(`webserver.local.dev`)
      # Allow request only from the predefined entry point named "web"
      - traefik.http.routers.webserverlocaldev.entrypoints=web # this is working with the port 80 entrypoint in the traefik config (a different docker-compose.yml)
      # What is essentially in this container's Dockerfile or image's Dockerfile under the `EXPOSE` setting
      - traefik.http.services.webserverlocaldev.loadbalancer.server.port=8001 # this can be anything, but mirror the change back to the Dockerfile via EXPOSE
    networks:
      - proxy
networks:
  proxy:
    external: true
```

Changeable substrings: `webserverlocaldev`, `webserver`, `webserver.local.dev`, `8001`

Everything else pretty much needs to stay as-is. 

Also, be sure to edit your local machine's `/etc/hosts` file to include a line similar to this:

```
127.0.0.1  webserver.local.dev
```

If all goes well, you should be able to simply spin up your application's container stack via `docker compose up` and visit `http://webserver.local.dev` in your browser and see it. 


## Step 3: Tell traefik about your https connections and certifications

Going back to the directory where your traefik container lives, you'll want to create the following tree:

```bash
.
├── certs # empty dir for now
├── docker-compose.yml
├── dynamic_conf.yml # empty file for now
└── traefik.yml
```

Certifications for https are... Complicated. Luckily, instead of praying to the [openssl gods](https://stackoverflow.com/questions/10175812/how-to-generate-a-self-signed-ssl-certificate-using-openssl#41366949) that you've nailed all the syntax requirements, [mkcert](https://github.com/FiloSottile/mkcert) exists. And for localdev purposes, it is the bees knees. 

Simply go to that repo and follow the install directions for your operating system. Then, run the following:

```bash
cd certs
mkcert -install
mkcert -key-file wildcard-local-dev-key.pem -cert-file wildcard-local-dev-cert.pem *.local.dev
```

On a sucessful run of `mkcert`, you should see the following appear in your traefik's app dir's tree:

```
.
├── certs
│   ├── wildcard-local-dev-key.pem
│   └── wildcard-local-dev-cert.pem
```

With the certs now created, we need to:

1. In the `docker-compose.yml`, mount the dir so traefik's container has those certs available in the filesystem, also include the new ports we want added and traefik's dynamic configuration:

```yaml
  traefik:
    image: traefik:v3.2
    ...
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./dynamic_conf.yml:/etc/traefik/dynamic_conf.yml
      - ./traefik.yml:/etc/traefik/traefik.yml
      - ./certs:/etc/certs:ro
```

2. In the `dynamic_conf.yml`, tell traefik about the certs:

```
tls:
  certificates:
    - certFile: /etc/certs/wildcard-local-dev-cert.pem
      keyFile: /etc/certs/wildcard-local-dev-key.pem
```

3. In the `traefik.yml`, tell traefik about how we want traffic served on 443 and about the dynamic config file:

```
entryPoints:
...
  web-secure:
    address: ":443"

providers:
...
  file:
    filename: /etc/traefik/dynamic_conf.yml
    watch: true
```

4. Restart the traefik container via `docker compose down && docker compose up` so the changes inside the `docker-compose.yml` get picked up.

## Step 4: Tell traefik about your 443 redirection from your application's docker-compose.yml

Swap back to your application's `docker-compose.yml`. Your labels should be refactored to look like this:

```yaml
# Changeable substrings: `webserverlocaldev`, `webserver`, `webserver.local.dev`, `8001`
services:
  webserver:
    ...
    labels:
      # Explicitly tell Traefik to expose this container
      - traefik.enable=true
      # Tell Traefik you are planning a redirection, and to include the needed middleware
      - traefik.http.middlewares.webserverlocaldev-redirect-web-secure.redirectscheme.scheme=https
      - traefik.http.routers.webserverlocaldev.middlewares=webserverlocaldev-redirect-web-secure
      # The domain the service will respond to, and what is in your /etc/hosts
      - traefik.http.routers.webserverlocaldev-web.rule=Host(`webserver.local.dev`)
      # Allow request only from the predefined entry point named "web"
      - traefik.http.routers.webserverlocaldev-web.entrypoints=web # this is working with the port 80 entrypoint in the traefik config (a different docker-compose.yml)
      # Let's redirect!
      - traefik.http.routers.webserverlocaldev-web-secure.rule=Host(`webserver.local.dev`)
      - traefik.http.routers.webserverlocaldev-web-secure.tls=true
      - traefik.http.routers.webserverlocaldev-web-secure.entrypoints=web-secure
      # What is essentially in this container's Dockerfile or image's Dockerfile under the `EXPOSE` setting
      - traefik.http.services.webserverlocaldev-web-secure.loadbalancer.server.port=8001 # this can be anything, but mirror the change back to the Dockerfile via EXPOSE
```

## Step 5: Restart your application and test

Now after a quick `docker compose down && docker compose up`, you should be able to visit `https://webserver.local.dev` in your browser. If you've made it this far, I'm genuinely proud of you. This has been a LOT of words.

![I have said too much](/5-steps-for-https-and-domain-names-on-local-dev-environments.gif 'I have said too much')

# Minor notes

Some folks out there will use the traefik config and its dynamic config to hold their application's traefik config, instead of putting them in their `docker-compose.yml`. This is not wrong, though in my opinion, it just makes more sense for the application to hold its own traefik configuration settings within its own `docker-compose.yml`. 

What we're doing here is complicated enough, and I personally don't see any need to mire traefik with your various application configurations. Keeping both applications separate allows for a much better debugging experience. Your application's settings will not get picked up by Traefik unless the application has been started. 

But at the end of the day, it's personal preference. Do what makes you happy. And on that note, I hope this help demystify https and domain names in Docker local devs. 




