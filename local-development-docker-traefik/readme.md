## The Problem

Local development with docker is nothing new.
Run a local container export some ports and here we go.
But I ran into the problem of working with multiple projects at the same time of conflicting ports on my docker host. And it is hard to remember which project is running on which port. Local domains would be a nice solution.
I will walk you through my local development setup with docker and traefik to solve this problem.

## Concept

We will install a reverse proxy on our local machine to add a domain to our projects. We will do this by using Traefik (https://traefik.io/traefik).
Traefik calls itself a cloud native application proxy.
It is ideal for using in cloud context like Kubernetes or docker. Traefik itself is also a simple docker container. And this will be the only container to expose a port to our docker host. The containers of the different projects and the traefik container will be in the same docker network. Traefik will forward the requests from the client to the corresponding container.

![Concept skatch](https://dev-to-uploads.s3.amazonaws.com/i/oxq2hx461j78ndvzqq38.jpg)

## Requirements

- docker
- docker-compose
- your IDE of choice

## Setup

Our first step will be to create a docker network.
We will call it "web". We are creating this network so that different docker-compose stacks can connect to each other.

```
docker network create web
```

Now we are starting our traefik container.
We could do this by running a simple docker command, but in this case ware a using a small docker-compose file to configure our container.

```
version: '3'

networks:
  web:
    external: true

services:
  traefik:
    image: traefik:v2.3
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    restart: always
    ports:
      - "80:80"
      - "8080:8080" # The Web UI (enabled by --api)
    networks:
      - web
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
Save this file in a directory and start the container by typing
```
docker-compose up -d
```
After the container started successfully we can access the traefik dashboard via http://localhost:8080.

Now we are starting a small web project. For example this is only a small website. I will only show the docker-compose.yml file in this post. You can find the complete folder structure and traefik setup here: https://github.com/flemssound/local-dev-docker-traefik

```
version: '3.3'

networks:
  web:
    external: true

services:
  application:
    image: nginx
    networks:
      - web
    # Here we define our settings for traefik how to proxy our service.
    labels:
      # This is enableing treafik to proxy this service
      - "traefik.enable=true"
      # Here we have to define the URL
      - "traefik.http.routers.myproject.rule=Host(`myproject.localhost`)"
      # Here we are defining wich entrypoint should be used by clients to access this service
      - "traefik.http.routers.myproject.entrypoints=web"
      # Here we define in wich network treafik can find this service
      - "traefik.docker.network=web"
      # This is the port that traefik should proxy
      - "traefik.http.services.myproject.loadbalancer.server.port=80"
    volumes:
      - ./html:/usr/share/nginx/html
    restart: always
```
Now we can access our website via http://myproject.localhost.
Everything in the HTML folder is mounted to the public folder into the nginx container.
Instead of exposing the nginx port directly to our host we proxy it through traefik.
We can also see it in the traefik dashboard.
![Traefik dashboard](https://dev-to-uploads.s3.amazonaws.com/i/pmhz3aw76g41b8j1s29e.png)

To create another project copy the myproject folder, adjust the docker-compose.yml and start it up. Now we have a second project running, for example under mysecondproject.localhost also on port 80, and we don't have to worry about conflicting ports in our projects and can access them by their name.

## References
- Traefik: https://traefik.io/traefik
- Nginx container: https://hub.docker.com/_/nginx
- Example project and setup: https://github.com/flemssound/local-dev-docker-traefik
