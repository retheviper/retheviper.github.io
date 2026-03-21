---
title: "Building a Home Server"
date: 2024-09-16
categories: 
  - nginx
image: "../../images/nginx.webp"
tags:
  - kotlin
  - ktor
  - compose
  - docker
  - nginx
---
This time, I created a home server. Since it has just been created, there may be various improvements and issues to be solved, but now that the structure is set up, I will briefly explain the structure.

## Why create your own server?

Recently, there are many cloud services that can be used for free, and you can easily publish web applications using services such as Firebase and Vercel. So why not just use the existing services? I have thought about this myself, but I will briefly explain why I decided to create my own server.

## Performance

VM and serverless services provided in many clouds often have limitations on CPU, memory, storage, etc. Also, in the case of services such as AWS and GCP, the CPU architecture used is often quite old-fashioned, such as Broadwell. In comparison, the machine I used to build my own server this time is an M2 Pro Mac mini, so I can expect higher performance compared to IPC. The memory is also equipped with 32GB of memory, so there are no memory limitations.

## Cost

Even if it is a free service, there is basically no cost, but if you add additional functions such as WAF for security or use a database, it often ends up costing money. And the more you use it, the less it costs. For the time being, I only want to use it for hobbies and personal functions, so I wanted to reduce my expenses as much as possible. If you build your own server, you may have to pay for electricity, but since the Mac mini consumes less power, it doesn't really matter, and if you put it to sleep when you're not using it, you can reduce your electricity bill.

## Features

In the case of serverless or VM, the supported runtime may be fixed or the DB options may be limited. For example, I am currently using a free instance of Oracle Cloud, but since PostgreSQL cannot be used with the free instance, I start the DB in a VM. When using an existing platform like this, you may need to adapt the functionality provided by the service rather than the functionality you want to use. This time, I wanted to be able to freely build a service using the technology I wanted without being subject to such restrictions.

## Others

I mentioned various other reasons, but the biggest one was that I was using two Macs and wanted to make use of the remaining resources. I use a MacBook Pro when I'm on the go, and I use a Mac mini all the time, so I wanted to use it as a server and be able to access it from outside.

## Infrastructure Configuration

The overall infrastructure looks like this:

![Infrastructure configuration](home-network-infrastructure.png)

## ONU

At home, I use NURO Hikari, so I need an ONU to connect to the internet. ONU is an abbreviation for Optical Network Unit, which is a device that connects to the Internet using optical lines. It also has basic router functions.

In my case, I use a separate Wi-Fi router in addition to the ONU, so I only configured port forwarding on the ONU. By forwarding TCP ports 80 and 443, requests from outside can reach the network behind the ONU.

## Router

Next is the router. As I explained earlier, the ONU also has the functions of a router, so there is no need to use it, but I use the router to use the free DDNS function provided by the router. ONU alone does not have a fixed IP, so by using the router's DDNS and linking it to the purchased domain, you can always access the same domain. Since you have a fixed domain, it will be easier to obtain an SSL certificate. Again, fort forward ports 80 and 443.

## Server

The server uses the Mac mini mentioned above. Inside, Nginx and Ktor applications built as Docker containers are running. Nginx is acting as a reverse proxy and forwarding requests to the Ktor application. Ktor applications can distribute static files, so we distribute web applications created with Compose Web (WASM).

Nginx and Ktor are configured on the same Docker network, so Nginx can access the Ktor application by container name. Nginx receives requests on ports 80 and 443 and forwards them to the Ktor application. Since I use Certbot to obtain an SSL certificate, the site is available over HTTPS. HTTP requests on port 80 are redirected to HTTPS on port 443 with a 301 redirect.

These configurations are defined in Docker Compose as follows.

```yaml
services:
  nginx:
    build:
      context: .
      dockerfile: Dockerfile_nginx
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - certs:/etc/letsencrypt
      - certs-data:/var/www/certbot
    networks:
      - home_network

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certs:/etc/letsencrypt
      - certs-data:/var/www/certbot
    networks:
      - home_network

  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: app:latest
    container_name: app
    ports:
      - "8888:8888"
    networks:
      - home_network
    depends_on:
      - nginx

networks:
  home_network:
    driver: bridge

volumes:
  certs:
  certs-data:
```

Also, to prevent unauthorized access, Nginx is built with the following Dockerfile.

```Dockerfile
# Use the latest Nginx as the base image
FROM nginx:latest

# Install required packages (git, wget, dnsutils)
RUN apt-get update && apt-get install -y \
    git \
    dnsutils \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Clone the Nginx Ultimate Bad Bot Blocker repository
RUN git clone https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker.git /opt/nginx-ultimate-bad-bot-blocker

# Create the required directories and copy the bot configuration files
RUN mkdir -p /etc/nginx/bots.d /usr/local/sbin \
    && cp /opt/nginx-ultimate-bad-bot-blocker/bots.d/* /etc/nginx/bots.d/ \
    && cp /opt/nginx-ultimate-bad-bot-blocker/conf.d/globalblacklist.conf /etc/nginx/conf.d/

# Download and run the `install-ngxblocker` script
RUN wget https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/install-ngxblocker -O /usr/local/sbin/install-ngxblocker \
    && chmod +x /usr/local/sbin/install-ngxblocker \
    && /usr/local/sbin/install-ngxblocker -x

# Download and run the `setup-ngxblocker` script
RUN wget https://raw.githubusercontent.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker/master/setup-ngxblocker -O /usr/local/sbin/setup-ngxblocker \
    && chmod +x /usr/local/sbin/setup-ngxblocker \
    && /usr/local/sbin/setup-ngxblocker -x

# Add custom Nginx configuration files (`nginx.conf` and `default.conf`)
COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf

# Volume for certificates (when using Let's Encrypt)
VOLUME ["/etc/letsencrypt", "/var/www/certbot"]

# Test the Nginx configuration and start it
CMD ["nginx", "-g", "daemon off;"]
```

## Issues

Generally speaking, there were no problems with the infrastructure configuration, but I felt there were some issues with the application part. Compose Web (Wasm) is mainly used in Frontend.

## Routing Is Not Supported

This may be a characteristic of applications built with Wasm, but Compose Web basically generates static SPA-like content, and since it is still in alpha, there are still many unsupported areas. In particular, because routing is not yet supported, page transitions are an issue. There is no official solution yet, but [a routing library compatible with Compose Wasm](https://mvnrepository.com/artifact/app.softwork/routing-compose-wasm-js) was recently released, and there are several similar libraries, so I would like to try them.

## Safari not supported

Wasm may not work depending on the browser. For Compose Web, we know that it doesn't work in Safari, so we need to think about how it works in Safari. The reason is that it is not yet compatible with GC. I don't think many users use Safari even when using a Mac, but many users use Safari on iOS, so this is quite a big issue. However, it is said that [WasmGC will be supported in Safari soon](https://www.publickey1.jp/blog/24/safariwasmgcsafari_technology_preview_202wasmgc.html), so we have no choice but to wait until then.

## CJK not supported

So-called CJK characters, such as Japanese, Chinese, and Korean, are not fully supported yet. In Compose Web, CJK characters still are not fully supported, so entering Japanese results in garbled text. This seems to be a problem related to fonts or rendering, and according to [this discussion](https://github.com/JetBrains/compose-multiplatform/issues/3967), some of it has been improved in the latest Compose versions. Still, depending on the Kotlin version, it may not yet be practical, so more adjustments seem necessary.

## Finally

This time I mainly talked about infrastructure configuration, but one of my goals was to use Compose to create an application that can be completed with just Kotlin, so if possible, I would like to write about application development using Compose in the future. In the case of Compose Web, I've had some experience with it since the JS days, but recently I've been focusing more on Wasm, so there are still a lot of things that aren't covered, but I'm looking forward to its future development. Personally, I think the most appealing thing is that the server and client code can be shared through the common package, so I would like to develop it while thinking about various architectures.

See you soon!
