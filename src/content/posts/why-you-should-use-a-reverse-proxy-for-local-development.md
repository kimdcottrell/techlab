---
title: "Why you should use a reverse proxy for local development"
date: 2023-07-30T17:47:24Z
draft: true
tags: ['docker', 'linux', 'devops', 'container', 'traefik', 'reverse proxy', 'https']
summary: ""
---

# Why you should use a reverse proxy for local development?

Let's take a step back. If you're just writing a file using HTML, CSS, and JS, why wouldn't you just load that file up in your browser using the `file://` protocol and doing all your development that way?

Put simply: Unless you're making a webpage simply for yourself on your local machine, your endgame is to make a website. A website is served either via `http://` or `https://`, and different Javscript functionalities in your browser, and in your file, will be accessible only when accessing your webpage via those `http*://` protocols. 

Using docker and a webserver container is a step above that. You'll at least get your pages served via the `http://` protocol. However, if you're in the professional webdev space, you'll _quickly_ discover Docker's `http://1.2.3.4:8080` syntax is just not good enough. A number of vendor API's require that you're accessing them under at least a domain name, if not a domain name served via https, aka on port 443. 

If you plan on using Docker for local development on websites that interact with things like payment processors or ecommerce vendors, you _require_ a reverse proxy. Local development for websites that interact with payment processors or ecommerce vendors commonly require a domain name and https. Using a reverse proxy will make these things possible:

1. access to your local development environment webserver via a domain name in your browser, e.g. `http://mysite.local.dev`
2. serving pages locally via https, e.g. `https://mysite.local.dev`

# What's an easy way to setup a reverse proxy for local development? 



