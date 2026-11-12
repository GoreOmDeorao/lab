# Assignment 5: Docker Swarm

All commands and full explanations are in the [Docker Swarm practical guide](../docker/06-swarm.md). Docker Engine must be running. Start with Q1 and keep the same Swarm cluster for Q2-Q5.

## Q1. NGINX service with two replicas, port 8080

```bash
docker swarm init
docker service create --name nginx-service --replicas 2 --publish published=8080,target=80 nginx:latest
docker service ls
docker service ps nginx-service
```

Open `http://localhost:8080`.

## Q2. Python worker service with three replicas

Create `app.py`:

```python
import socket
import time

while True:
    print(f"Running in Swarm... host={socket.gethostname()}", flush=True)
    time.sleep(5)
```

Create `Dockerfile`:

```dockerfile
FROM python:3.12-alpine
WORKDIR /app
COPY app.py .
CMD ["python", "-u", "app.py"]
```

Build and deploy:

```bash
docker build -t python-worker:1.0 .
docker service create --name python-service --replicas 3 python-worker:1.0
docker service ps python-service
docker service logs --follow python-service
```

To see each running task's container ID:

```bash
docker ps --filter label=com.docker.swarm.service.name=python-service
```

Use `docker logs <CONTAINER_ID>` for any listed container. In a single-node Swarm all replicas run on the same machine; use multiple joined nodes to see distribution across machines.

## Q3. Scale NGINX from 2 to 5, then to 1

```bash
docker service scale nginx-service=5
docker service ls
docker service ps nginx-service

docker service scale nginx-service=1
docker service ls
docker service ps nginx-service
```

## Q4. Rolling update from `nginx:1.25` to `nginx:latest`

```bash
docker service update --image nginx:1.25 nginx-service
docker service ps nginx-service

docker service update \
  --image nginx:latest \
  --update-parallelism 1 \
  --update-delay 10s \
  --update-failure-action rollback \
  nginx-service

docker service ps nginx-service
docker service inspect --pretty nginx-service
```

Rollback if needed:

```bash
docker service rollback nginx-service
docker service ps nginx-service
```

## Q5. Remove services and leave Swarm

```bash
docker service ls
docker service rm nginx-service
docker service rm python-service
docker service ls
docker swarm leave --force
```

`docker swarm leave --force` is destructive to this node's Swarm membership. Run it only after removing the lab services and when you no longer need this cluster.

## Q6-Q7. Written explanation questions

These two prompts ask for conceptual descriptions of Swarm usage and limitations, not command-line practical tasks. They are intentionally not expanded in this practical-only handbook. The hands-on examples above demonstrate service deployment, scaling, load-balanced published ports, rolling updates, and cleanup.
