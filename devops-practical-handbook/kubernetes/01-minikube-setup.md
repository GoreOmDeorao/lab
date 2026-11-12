# Kubernetes: Minikube Setup on Ubuntu

Source coverage: Kubernetes slides 18 and 42-43; Assignment 4, Q5. The setup below uses Minikube with Docker as the driver.

## 1. Prerequisites

- Ubuntu/Linux x86-64 or ARM64 system.
- Docker Engine installed and running.
- At least 2 CPUs, 2 GB free memory, and 20 GB free disk space are Minikube's basic published requirements.

Verify Docker:

```bash
docker version
docker ps
```

If your login user needs Docker-group access:

```bash
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker "$USER"
newgrp docker
docker ps
```

Docker-group membership effectively grants root-equivalent host access. Only do this for a trusted user.

## 2. Install Minikube on Linux (x86-64)

```bash
cd ~/Downloads
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

Verify:

```bash
minikube version
```

## 3. Start the cluster with the Docker driver

```bash
minikube start --driver=docker
minikube status
minikube kubectl -- get nodes
```

The first start downloads Kubernetes images and can take several minutes. A node in `Ready` state indicates the local cluster is ready to accept Pods.

## 4. Use kubectl through Minikube

Minikube can run a compatible kubectl binary for you:

```bash
minikube kubectl -- get pods -A
minikube kubectl -- get nodes
```

Add the alias used by the lecture so you can type `kubectl` directly:

```bash
echo 'alias kubectl="minikube kubectl --"' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
kubectl get pods -A
```

If you already installed `kubectl` separately, you can use that executable too. Check which one runs with `type -a kubectl`.

## 5. Start/stop/delete Minikube

```bash
minikube status              # Show component status
minikube stop                # Stop the local cluster without deleting it
minikube start --driver=docker # Start the existing cluster again
minikube delete              # DESTRUCTIVE: delete the cluster and its local Kubernetes data
```

## 6. Useful environment commands

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
minikube ip
minikube dashboard
```

## 7. Build a local image inside Minikube

For a local image that is not pushed to Docker Hub, build it into Minikube's image store:

```bash
minikube image build -t my-local-app:v1 .
minikube image ls | grep my-local-app
```

Then configure the Kubernetes Deployment to use that exact tag and set `imagePullPolicy: IfNotPresent` (or `Never` when you want to prohibit pulling). A tag mismatch or building only in the host Docker image store can lead to `ImagePullBackOff`.

## 8. Access Services in Minikube

```bash
minikube service <SERVICE_NAME> --url
```

For a `LoadBalancer` Service, keep this running in a separate terminal:

```bash
minikube tunnel
```

Then inspect its external address:

```bash
kubectl get svc
```

On some Linux hosts, the node IP and NodePort are directly reachable. `minikube service ... --url` is a convenient cross-platform method.

## 9. Official references

- Minikube installation: https://minikube.sigs.k8s.io/docs/start/
- Minikube Docker driver: https://minikube.sigs.k8s.io/docs/drivers/docker/
- Kubernetes kubectl documentation: https://kubernetes.io/docs/reference/kubectl/
