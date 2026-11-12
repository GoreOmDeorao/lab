# Docker: Installation, Images, and Container Lifecycle

Practical reference for Ubuntu/Linux. Commands are grouped in the order you normally run them. Source coverage: Docker slides 9-25 and 59-71; `Docker notes.txt`; Assignment 2, Q1-Q6.

## 1. Install Docker Engine on Ubuntu

The lecture directs you to Docker's official APT repository installation. Use the current official steps for your Ubuntu release rather than copying old package commands from an outdated blog:

- https://docs.docker.com/engine/install/ubuntu/

After installing Docker Engine, verify the daemon and CLI:

```bash
sudo systemctl enable --now docker       # Start Docker now and at boot
sudo systemctl status docker --no-pager  # Confirm the daemon is active
docker --version                         # Print the Docker CLI version
sudo docker run --rm hello-world         # Confirm the daemon can start a test container
```

Optional: allow your login user to use Docker without typing `sudo`:

```bash
sudo groupadd docker 2>/dev/null || true # Create group if it does not already exist
sudo usermod -aG docker "$USER"          # Add your user to the Docker group
newgrp docker                             # Start a shell with the new group membership
# If the new shell does not refresh permissions, log out and log back in.
docker ps                                 # Verify access to the daemon
```

**Security:** membership in the `docker` group is effectively root-level access to the host. Do not add untrusted users. Never make `/var/run/docker.sock` world-writable with `chmod 777`.

Alternative shown in the Docker slides (Snap):

```bash
sudo snap install docker
sudo docker --version
```

Prefer one installation method. Do not install Docker Engine from APT and Snap at the same time unless you know how to manage the two installations.

## 2. Common ready-to-run images from the slides

```bash
docker run -it ubuntu:latest bash                 # Interactive Ubuntu shell
docker run -d -p 8080:80 nginx:latest              # NGINX web server
docker run -d -e MYSQL_ROOT_PASSWORD=labpass mysql:8 # MySQL server (lab password only)
docker run -it python:3.12 python                 # Python interpreter
# MongoDB example from the slide:
docker run -d -p 27017:27017 mongo:latest
```

Use a dedicated volume and a non-public lab password for databases. Do not publish database ports to untrusted networks.

## 2. Pull, list, inspect, and remove images

```bash
docker pull nginx:latest          # Download the NGINX image from Docker Hub
docker image ls                   # List images stored locally
docker image inspect nginx:latest # Show image metadata, layers, and configuration
docker image rm nginx:latest      # Remove the image if no container still references it
```

`docker run IMAGE` automatically pulls an image when it is not available locally. An image cannot normally be removed while a container (including a stopped container) still references it. Remove the referencing container first.

## 3. Run an NGINX container and publish a port

```bash
docker run --name test -d -p 8080:80 nginx:latest
# --name test gives the container a predictable name.
# -d runs it in the background.
# -p 8080:80 maps host port 8080 to container port 80.
docker ps
```

Open `http://localhost:8080`. If port 8080 is already in use, choose another host port, for example `-p 8088:80`, and browse to `http://localhost:8088`.

Enter the running container:

```bash
docker exec -it test /bin/bash  # Open an interactive Bash shell inside the container
ls                              # Run a Linux command inside the container
exit                            # Leave the container shell; the container keeps running
```

Stop and remove it:

```bash
docker stop test      # Gracefully stop the container
docker rm test        # Remove the stopped container
docker ps -a          # List all containers, including stopped ones
```

## 4. Container run modes

One-shot process: the container stops when its main process finishes.

```bash
docker run --rm busybox echo "Hello from BusyBox"
```

Interactive Ubuntu shell:

```bash
docker run -it --name ubuntu-lab ubuntu:latest bash
# Inside the container:
pwd
ls
exit
```

Restart and attach to the stopped container:

```bash
docker ps -a                         # Find ubuntu-lab in the list
docker start -ai ubuntu-lab          # Start it and attach to its terminal
# Type exit again to stop its interactive main process.
```

Detached interactive BusyBox shell:

```bash
docker run -dit --name first-busybox busybox
# -d: detached; -i: keep stdin open; -t: allocate a terminal
docker exec -it first-busybox sh     # Open a shell in the already-running container
exit
docker stop first-busybox
docker rm first-busybox
```

## 5. List, name, rename, inspect, and read logs

```bash
docker ps                                      # Running containers only
docker ps -a                                   # All containers
docker container ls                            # Another form of docker ps
docker images                                  # Local image list
docker inspect test                            # Full JSON details for a container
docker inspect nginx:latest                    # Full JSON details for an image
docker inspect -f '{{.NetworkSettings.IPAddress}}' test # Print the container IP
docker rename test web-server                  # Rename an existing container
docker logs web-server                         # Print captured stdout/stderr
docker logs -f web-server                      # Follow logs live
docker logs --tail 10 web-server               # Print only the last 10 lines
docker logs -t web-server                      # Include timestamps
```

Create a named container at start instead of accepting a random name:

```bash
docker run -dit --name myubuntu ubuntu:latest bash
```

## 6. Stop, remove, and prune containers/images

```bash
docker stop myubuntu        # Stop a running container
docker rm myubuntu          # Remove a stopped container
docker rm -f myubuntu       # Force-stop and remove a named container
```

Prune all stopped containers (not just one):

```bash
docker container prune     # Destructive: removes every stopped container
```

Remove unused images:

```bash
docker image prune          # Remove dangling images
docker image prune -a       # Destructive: remove all images not used by containers
```

The lecture includes these broad cleanup commands. They affect every container on the machine, so inspect the list first:

```bash
docker ps -a
# Destructive: stops every container, including unrelated labs.
docker stop $(docker ps -aq)
# Destructive: removes every container, including stopped containers.
docker rm $(docker ps -aq)
```

Safer cleanup for only the lab containers is to name them and remove those names explicitly.

## 7. Tag an image and publish it to Docker Hub

```bash
docker pull nginx:latest
docker tag nginx:latest mynginx:v1       # Add another tag to the same local image
docker tag nginx:latest mynginx:latest  # Add a second tag
docker image ls
```

Remove one tag:

```bash
docker image rm mynginx:v1
```

Removing one tag does not necessarily delete the image layers if another tag still references the image.

Push an image you own. Replace the username with your Docker Hub account:

```bash
DOCKERHUB_USER="your-dockerhub-username"
docker login -u "$DOCKERHUB_USER" # Enter credentials/token when prompted
docker tag nginx:latest "$DOCKERHUB_USER/mynginx:v1"
docker push "$DOCKERHUB_USER/mynginx:v1"
docker logout
```

Verify the repository on https://hub.docker.com/. Do not put passwords or access tokens in shell history or Git files.

## 8. Copy an application from GitHub and build it

The slide deck's example repository:

```bash
cd ~
git clone https://github.com/newdelthis/webappdemo.git
cd webappdemo
cat Dockerfile                    # Inspect the image build instructions
docker build -t webappdemo:1.0 .   # Build from the Dockerfile in the current directory
docker image ls
```

The slide maps host port 9999 to container port 8080, which assumes the application listens on port 8080 inside the container:

```bash
docker run -d --name web-1 -p 9999:8080 webappdemo:1.0
```

Open `http://localhost:9999`. Remove the same container name used above:

```bash
docker rm -f web-1
docker image rm webappdemo:1.0
```

## 9. Save and load an image as a tar file

```bash
docker pull ubuntu:latest
docker save ubuntu:latest -o ubuntu.tar  # Export image and layers to a tar archive
ls -lh ubuntu.tar
```

Copy `ubuntu.tar` to the second machine, then:

```bash
docker load -i ubuntu.tar               # Import image into the local Docker image store
docker image ls
```

## 10. Source note: Git, GitHub, Docker, and Docker Hub

- Git records and manages source-code history.
- GitHub hosts Git repositories.
- A Dockerfile plus application code and dependencies are used to build an image.
- A container is a running instance of an image.
- Docker Hub stores and distributes container images.

Useful command chain:

```bash
git clone <REPOSITORY_URL>
cd <REPOSITORY_DIRECTORY>
docker build -t <IMAGE_NAME>:<TAG> .
docker run --rm <IMAGE_NAME>:<TAG>
docker tag <IMAGE_NAME>:<TAG> <DOCKERHUB_USER>/<IMAGE_NAME>:<TAG>
docker push <DOCKERHUB_USER>/<IMAGE_NAME>:<TAG>
```

Replace angle-bracket placeholders before running; they are not literal shell arguments.

## 11. Run a command and remove the container automatically

```bash
docker run --rm alpine:latest sh -c "echo 'hello world'"
```

`--rm` removes the container as soon as its one-shot command exits. The image remains locally cached.
