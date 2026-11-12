# Assignment 4: Docker Compose and Minikube Installation

## Q1. NGINX static website with Docker Compose

Follow [Compose Q1: NGINX static website](../docker/05-compose.md#2-nginx-static-website-with-a-bind-mount-assignment-4-q1). The `compose.yaml` runs NGINX on host port 8080 and mounts `website/` read-only into the NGINX document root.

```bash
mkdir -p ~/compose-nginx/website
cd ~/compose-nginx
echo '<h1>Hello from Docker Compose</h1>' > website/index.html
```

`compose.yaml`:

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./website:/usr/share/nginx/html:ro
```

```bash
docker compose up -d
docker compose ps
```

Open `http://localhost:8080`. Stop with `docker compose down`.

## Q2. Run a Python script as a one-shot Compose service

Complete files and explanation: [Compose Q2: Python script container](../docker/05-compose.md#3-run-a-one-shot-python-script-with-compose-assignment-4-q2).

`app.py`:

```python
print("Hello from Python Container")
```

`Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

`compose.yaml`:

```yaml
services:
  python-job:
    build: .
```

```bash
docker compose up --build
```

The script is expected to print and exit. Stop/remove with `docker compose down`.

## Q3. MySQL with persistent storage using Compose

Use [Compose Q3: MySQL named volume](../docker/05-compose.md#4-mysql-with-persistent-named-volume-assignment-4-q3). It includes `.env`, `.gitignore`, `compose.yaml`, SQL for the `test` table, and a restart/recreate verification. Do not commit `.env`.

```bash
docker compose up -d
docker compose ps
docker compose exec db mysql -uroot -p
```

Create and insert into `test`, exit, then:

```bash
docker compose down
docker compose up -d
docker compose exec db mysql -uroot -p -D college -e 'SELECT * FROM test;'
```

## Q4. Three NGINX containers using one shared volume

Use [Compose Q4: shared volume with separate document roots](../docker/05-compose.md#5-three-nginx-containers-one-shared-volume-three-different-pages-assignment-4-q4).

It uses one named volume attached to all three NGINX containers at `/usr/share/nginx/html`, and separate NGINX configuration files so each container serves a different subdirectory.

```bash
docker compose up -d
docker compose ps
curl http://localhost:8081
curl http://localhost:8082
curl http://localhost:8083
```

Expected: Container 1, Container 2, and Container 3 respectively. The simple approach of mounting the same volume as the default document root in every container would show the same page on all ports.

## Q5. Install Minikube on Ubuntu

Docker must already be installed and running. Minikube's Docker driver uses the local Docker installation. See the full [Kubernetes setup guide](../kubernetes/01-minikube-setup.md).

Install Minikube on x86-64 Linux:

```bash
cd ~/Downloads
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

Allow your login user to access Docker (then start a new login session if needed):

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker ps
```

Start and verify the local Kubernetes cluster:

```bash
minikube start --driver=docker
minikube status
minikube kubectl -- get nodes
```

Add the optional `kubectl` shortcut:

```bash
echo 'alias kubectl="minikube kubectl --"' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```
