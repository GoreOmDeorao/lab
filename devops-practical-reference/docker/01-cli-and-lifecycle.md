# Docker — CLI and Container Lifecycle

Commands from Docker slides 9–27 and `Docker notes.txt`.

## 1. Install Docker on Ubuntu

The slides direct you to the official APT repository installation method:

1. Follow [Docker Engine installation for Ubuntu](https://docs.docker.com/engine/install/ubuntu/).
2. Run the commands under **Set up Docker's apt repository**.
3. Run the commands under **Install Docker Engine**.
4. Add your login user to the `docker` group and reboot or start a new login session.

```bash
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker "$USER"
sudo reboot
```

Verify after logging back in:

```bash
docker version
docker ps
```

Alternative shown in the slides (Snap):

```bash
sudo snap install docker
sudo docker --version
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker "$(whoami)"
sudo reboot
```

> Docker group membership grants root-equivalent control of the host. Only add trusted users.

## 2. Pull and run NGINX

```bash
docker images
docker pull nginx:latest
docker run --name test -d -p 8080:80 nginx:latest
docker ps
```

Open `http://localhost:8080`.

Open a shell inside the container:

```bash
docker exec -it test /bin/bash
ls
exit
```

Stop and remove it:

```bash
docker stop test
docker rm test
docker ps -a
```

## 3. Container run modes

One-shot command (the container exits when its default process exits):

```bash
docker run busybox
docker ps
```

Interactive container:

```bash
docker run -it ubuntu
# Inside the container:
exit
```

Detached interactive BusyBox container:

```bash
docker run -dit --name first-busybox busybox
docker ps
docker exec -it first-busybox /bin/sh
exit
docker stop first-busybox
docker rm first-busybox
```

Run an NGINX container in the background:

```bash
docker run -d --name ctr1 nginx
docker ps
```

## 4. List, inspect, rename, and view logs

```bash
# Running containers
docker ps

# All containers, including stopped ones
docker ps -a

# Images stored locally
docker images

# Inspect a container or image
docker inspect <CONTAINER_NAME_OR_ID>
docker inspect ubuntu

# Print the IP address of a container
docker inspect -f '{{.NetworkSettings.IPAddress}}' <CONTAINER_NAME_OR_ID>

# Rename a container
docker rename <OLD_NAME> <NEW_NAME>

# View logs
docker logs <CONTAINER_NAME>
docker logs -f <CONTAINER_NAME>
docker logs --tail 10 <CONTAINER_NAME>
docker logs -t <CONTAINER_NAME>
```

## 5. Stop and remove containers

```bash
docker stop <CONTAINER_NAME_OR_ID>
docker rm <CONTAINER_NAME_OR_ID>
```

Stop and remove a named container in one command:

```bash
docker rm -f <CONTAINER_NAME>
```

The source notes include these bulk cleanup commands. They affect **every** container on the Docker host:

```bash
# DESTRUCTIVE: stops all containers
docker stop $(docker ps -a -q)

# DESTRUCTIVE: removes all containers
docker rm $(docker ps -a -q)
```

## 6. Prune unused resources

```bash
# DESTRUCTIVE: removes all stopped containers
docker container prune

# DESTRUCTIVE: removes all images not used by existing containers
docker image prune -a
```

## 7. Remove an image

```bash
docker images
docker rmi <IMAGE_NAME_OR_ID>
```

Source reference: Docker slides 9–27; `Docker notes.txt` sections on image pull, run, exec, ps, stop, rm, and cleanup.
