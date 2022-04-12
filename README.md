# Traefik Reverse Proxy for Docker on macOS using HTTPS

This is a rundown of how my local web development environment on macOS uses Traefik as a reverse proxy to route traffic to Docker containers over HTTPS. I created this for my own reference since I am not an expert, but hopefully it can help someone else and maybe find some suggested improvements.

* [Features](#features)
* [Dnsmasq Configuration](#dnsmasq-configuration)
* [Traefik Configuration](#traefik-configuration)
* [Project Setup](#project-setup)
  * [Assign Networks and Labels](#assign-networks-and-labels-to-container)
  * [Creating Certs](#creating-certs)
  * [Add CA to our Dockerfile](#add-ca-to-our-dockerfile)
* [Useful Links](#useful-links)

## Features

* Reverse proxy for local Docker containers
* Fixes browser certificate warnings
* Automatically redirects http traffic to https

## Dnsmasq Configuration

We will be using Dnsmasq to make sure all domains using the `.test` TLD are pointed at our local machine and Traefik can handle the requests. We could use `127.0.0.1` as our target and everything will work fine on our host, but when we try to resolve a `.test` domain from inside a container `127.0.0.1` will refer to the current container. To solve this we can create a loopback interface and use that IP.


1. Install Dnsmasq on the host machine.
1. Configure Dnsmasq to point all domains using our `.test` TLD to `10.254.254.254`
1. Run `sudo ifconfig lo0 alias 10.254.254.254` to create an IP address alias (note: this will not persist across a reboot, so [here is a potential solution](https://web.archive.org/web/20200805154725/https://blog.felipe-alfaro.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/))

## Traefik Configuration

1. Create a Docker network named "web" by running `docker network create web`
1. After you've configured your projects and set up your `dynamic_conf.yml` with the appropriate certs, run `docker-compose up -d` to start Traefik
1. The Traefik dashboard should now be visible at http://localhost:8080

## Project Setup

### Assign Networks and Labels to Container

For a container to be accessible to Traefik, we will be adding it to the "web" Docker network we created. We should also assign it to the "default" network if we want to connect to other containers (like a database) in the same project.

We will also need to add labels so that Traefik knows how to route traffic. Use the name of the project as our router name.

These can be added directly to our `docker-compose.yml` file or to a `docker-compose.override.yml` file if preferred.

```yml
services:
  webserver:
    networks:
      - default
      - web
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${ROUTER_NAME}.tls=true"
      - "traefik.http.routers.${ROUTER_NAME}.rule=Host(`projectdomain.test`)"
      - "traefik.http.routers.${ROUTER_NAME}.entrypoints=websecure"
      - "traefik.docker.network=web"
      - "traefik.http.services.${ROUTER_NAME}.loadbalancer.server.port=80"

networks:
  web:
    external: true
```

### Creating Certs

1. Ensure [mkcert](https://github.com/FiloSottile/mkcert) is installed by running `brew install mkcert` as well as `brew install nss` if you use Firefox
1. Create local CA by running `mkcert -install`
1. Change into `certs` directory
1. Run `mkcert mydomain.test` or `mkcert "*.mydomain.test` for a wildcard
1. Create `dynamic_conf.yml` by switching back to the root directory and running `cp dynamic_conf.example.yml dynamic_conf.yml` and then making the necessary changes

```yml
# Dynamic configuration
tls:
  certificates:
    - certFile: "/etc/traefik/certs/mydomain.test.pem"
      keyFile: "/etc/traefik/certs/mydomain.test-key.pem"
    - certFile: "/etc/traefik/certs/_wildcard.mydomain.test.pem"
      keyFile: "/etc/traefik/certs/_wildcard.mydomain.test-key.pem"
```

### Add CA to our Dockerfile

Because our CA is on our host machine, we'll need to add it to any containers that connect to another via a secure URL using one of our generated certificates. The exact Dockerfile changes needed may vary based on the image, but this example seems to work on Ubuntu.

1. Run `mkcert -CAROOT` in terminal to find the location of our CA
1. Copy `rootCA.pem` from this location into our build directory
1. Add the following to our Dockerfile to install the CA into the image

```dockerfile
COPY rootCA.pem /usr/local/share/ca-certificates/rootCA.crt
RUN chmod 644 /usr/local/share/ca-certificates/rootCA.crt && update-ca-certificates;
```

## Useful Links

Thanks to the authors of these articles that helped me scrape this together.

* [Using HTTPS certificates with Traefik and Docker for a development environment](https://www.andrewdixon.co.uk/2020/03/14/using-https-certificates-with-traefik-and-docker-for-a-development-environment/)
* [Use dnsmasq instead of /etc/hosts](https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/)
* [Development setup with Nuxt, Node and Docker](https://medium.com/@marcmintel/development-setup-with-nuxt-node-and-docker-b008a241c11d)
* [HTTP to HTTPS redirects with Traefik](https://jensknipper.de/blog/traefik-http-to-https-redirect/)
* [Local Dev on Docker - Fun with DNS](https://williamhayes.medium.com/local-dev-on-docker-fun-with-dns-85ca7d701f0a)
