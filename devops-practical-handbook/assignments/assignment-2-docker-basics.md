# Assignment 2: Docker Basics

These solutions match Assignment 2, Q1-Q7. Run one question at a time. Docker Engine must already be installed and running; see [Docker installation](../docker/01-container-basics.md#1-install-docker-engine-on-ubuntu).

## Q1. Pull and run an NGINX image

```bash
docker pull nginx:latest                 # Download NGINX image from Docker Hub
docker run -d --name nginx-a2 -p 8080:80 nginx:latest
# -d: background; --name: stable container name; -p: host 8080 -> container 80
docker ps                                # Verify that it is running
```

Open `http://localhost:8080`. Then stop and remove:

```bash
docker stop nginx-a2
docker rm nginx-a2
docker ps -a                              # Verify it no longer appears
```

## Q2. Image commands with Alpine

```bash
docker pull alpine:latest
docker image ls                           # List all local images
docker image inspect alpine:latest       # Detailed image metadata
docker image rm alpine:latest            # Remove Alpine if no container uses it
```

Try this only after starting a container from Alpine:

```bash
docker run -dit --name alpine-test alpine:latest sleep infinity
docker image rm alpine:latest             # Expected to fail while container references image
docker rm -f alpine-test
docker image rm alpine:latest             # Now image can be removed if no other container uses it
```

**Why the first removal fails:** Docker protects an image that is still referenced by a container, even if that container is stopped. Remove the container first.

## Q3. Interactive Ubuntu container and restart

```bash
docker run -it --name ubuntu-a2 ubuntu:latest bash
```

Inside the container, run:

```bash
pwd
ls
exit
```

Back on the host:

```bash
docker ps -a                              # The exited container is still listed
docker start -ai ubuntu-a2               # Restart and attach to its terminal
# Type exit to stop it again.
docker ps -a
```

## Q4. Name and rename a container

```bash
docker run -d --name web_server -p 8081:80 nginx:latest
docker ps --filter name=web_server
docker stop web_server
docker rename web_server my_server
docker ps -a --filter name=my_server
docker rm my_server
```

The name changes from `web_server` to `my_server`; the container ID stays the same.

## Q5. Tag an image

```bash
docker pull nginx:latest
docker tag nginx:latest mynginx:v1
docker tag nginx:latest mynginx:latest
docker image ls mynginx
```

Remove one tag and inspect the remaining tags:

```bash
docker image rm mynginx:v1
docker image ls mynginx
```

Tags are references to an image. Removing one tag does not remove the underlying image if another tag still points to it.

## Q6. Push an image to Docker Hub

Set the Docker Hub account name:

```bash
DOCKERHUB_USER="your-dockerhub-username"
docker login -u "$DOCKERHUB_USER"          # Enter password/token when prompted
docker tag nginx:latest "$DOCKERHUB_USER/mynginx:v1"
docker push "$DOCKERHUB_USER/mynginx:v1"
docker logout
```

Verify the repository on https://hub.docker.com/. Do not write access tokens or passwords into this repository.

## Q7. Port mapping and Docker networks

```bash
docker run -d --name nginx-network -p 8080:80 nginx:latest
```

Open `http://localhost:8080`.

- `8080` is the port on the Linux host.
- `80` is the port inside the NGINX container.
- Requests to host port 8080 are forwarded to container port 80.

List and inspect networks:

```bash
docker network ls
docker network inspect bridge
```

Cleanup:

```bash
docker rm -f nginx-network
```

More detail and custom bridge name-based communication: [Docker networking guide](../docker/02-networking.md).
