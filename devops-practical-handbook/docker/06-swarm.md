# Docker Swarm: Services, Scaling, Rolling Updates

Source coverage: Docker slides 96-101; Assignment 5, Q1-Q5.

## 1. Initialize a single-node Swarm

Check current Swarm state first:

```bash
docker info --format '{{.Swarm.LocalNodeState}}'
```

If the output is `inactive`, initialize Swarm:

```bash
docker swarm init
```

If Docker reports multiple network interfaces and requires an advertise address, find the machine's IP and specify the reachable interface address:

```bash
ip -brief address
docker swarm init --advertise-addr <MANAGER_IP>
```

Replace `<MANAGER_IP>` with the machine's actual IP address. List the node(s):

```bash
docker node ls
```

A single-machine Swarm is useful for this lab but does not demonstrate fault tolerance across machines. A multi-node Swarm needs worker nodes joined using the token and command shown by `docker swarm join-token worker` on the manager.

## 2. Assignment 5 Q1: Deploy two NGINX replicas on host port 8080

```bash
docker service create \
  --name nginx-service \
  --replicas 2 \
  --publish published=8080,target=80 \
  nginx:latest
```

Meaning:

- `service create` asks Swarm to maintain a service, not just one standalone container.
- `--name` sets the service name.
- `--replicas 2` asks Swarm to keep two tasks running.
- `--publish published=8080,target=80` routes host port 8080 to container port 80.
- `nginx:latest` is the image used by each task.

Verify:

```bash
docker service ls
docker service ps nginx-service
docker ps
```

Open `http://localhost:8080`. Swarm's routing mesh can accept a published service port on a Swarm node and route it to a task.

## 3. Assignment 5 Q2: Python worker service (3 replicas)

Create a folder and `app.py`:

```bash
mkdir -p ~/swarm-python-worker
cd ~/swarm-python-worker
```

`app.py`:

```python
import socket
import time

while True:
    print(f"Running in Swarm... host={socket.gethostname()}", flush=True)
    time.sleep(5)
```

`flush=True` makes each message appear immediately in container logs rather than waiting for buffered output.

Create `Dockerfile`:

```dockerfile
FROM python:3.12-alpine
WORKDIR /app
COPY app.py .
CMD ["python", "-u", "app.py"]
```

Build the worker image:

```bash
docker build -t python-worker:1.0 .
docker image ls python-worker
```

Create the Swarm service with three replicas:

```bash
docker service create \
  --name python-service \
  --replicas 3 \
  python-worker:1.0
```

Verify tasks and inspect service logs:

```bash
docker service ls
docker service ps python-service
docker service logs --follow python-service
```

To inspect logs from the individual containers on this single-node lab:

```bash
docker ps --filter label=com.docker.swarm.service.name=python-service
```

Copy one container ID from the result and run:

```bash
docker logs <CONTAINER_ID>
```

Repeat with another container ID. The `host=` value printed by the Python script helps distinguish which task/container produced the line.

**Multi-node note:** a locally built image exists only on this machine. If your Swarm has multiple worker nodes, push the image to a registry and deploy using the registry-qualified image name so every node can pull it.

## 4. Assignment 5 Q3: Scale an NGINX service up and down

Use the `nginx-service` created in Q1. If it does not exist, create it first using Q1.

Increase the replica count to five:

```bash
docker service scale nginx-service=5
docker service ls
docker service ps nginx-service
```

Decrease to one:

```bash
docker service scale nginx-service=1
docker service ls
docker service ps nginx-service
```

Swarm removes extra tasks when the desired replica count decreases. The service definition remains; only the number of running tasks changes.

## 5. Assignment 5 Q4: Rolling update one task at a time with 10-second delay

First make sure the service uses `nginx:1.25`:

```bash
docker service update --image nginx:1.25 nginx-service
docker service ps nginx-service
```

Update it to `nginx:latest`, one task at a time, with a 10-second delay:

```bash
docker service update \
  --image nginx:latest \
  --update-parallelism 1 \
  --update-delay 10s \
  --update-failure-action rollback \
  nginx-service
```

Check progress:

```bash
docker service ls
docker service ps nginx-service
docker service inspect --pretty nginx-service
```

If the update is faulty and you need to return to the previous service specification:

```bash
docker service rollback nginx-service
docker service ps nginx-service
```

A rolling update changes tasks gradually rather than replacing every task simultaneously. The 10-second delay gives time between task updates.

## 6. Assignment 5 Q5: Remove services and leave Swarm

List services first:

```bash
docker service ls
```

Remove the two lab services:

```bash
docker service rm nginx-service
docker service rm python-service
```

Verify:

```bash
docker service ls
```

Leave Swarm on this manager node:

```bash
docker swarm leave --force
```

`--force` is required to force a manager to leave the Swarm. Only run this on your lab node when you no longer need the Swarm cluster.

## 7. Useful Swarm checks

```bash
docker info | grep -i swarm
docker node ls
docker service ls
docker service ps <SERVICE_NAME>
docker service logs <SERVICE_NAME>
docker service scale <SERVICE_NAME>=<REPLICA_COUNT>
docker service update --image <IMAGE:TAG> <SERVICE_NAME>
docker service rollback <SERVICE_NAME>
docker service rm <SERVICE_NAME>
```
