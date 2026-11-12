# Kubernetes — Practical Labs

Commands and manifests from Kubernetes slides 18–52. Commands assume Ubuntu, Docker, and Minikube.

## 1. Install Minikube on Ubuntu

Docker must already be installed.

```bash
cd ~/Downloads
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Allow the current user to access Docker. Log out/in or reboot if needed.
sudo usermod -aG docker "$USER"
newgrp docker

minikube start --driver=docker
minikube status
minikube kubectl -- get nodes
```

Create the `kubectl` shortcut used in the slides:

```bash
echo 'alias kubectl="minikube kubectl --"' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

## 2. Run and inspect a Pod imperatively

```bash
kubectl run nginx --image=nginx
kubectl get pods
kubectl describe pod nginx
kubectl logs nginx
kubectl exec nginx -- nginx -v
kubectl get pod nginx -o yaml
kubectl get pod nginx -o wide
kubectl exec nginx -- curl -I http://localhost
kubectl get pods -w
```

Delete the Pod:

```bash
kubectl delete pod nginx
```

## 3. Run and inspect a Deployment

```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl get pods
kubectl describe deployment nginx
kubectl get deployment nginx -o yaml
kubectl get pods -o wide
```

Inspect an individual Pod:

```bash
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
```

Scale to three replicas:

```bash
kubectl scale deployment nginx --replicas=3
kubectl get pods
```

Delete the Deployment and its managed Pods:

```bash
kubectl delete deployment nginx
```

## 4. Demonstrate self-healing

```bash
kubectl create deployment apache --image=httpd --replicas=5
```

In a second terminal, watch Pods:

```bash
kubectl get pods -w
```

In the first terminal, delete one of the Pods:

```bash
kubectl get pods
kubectl delete pod <POD_NAME>
```

Check the replacement and ReplicaSet:

```bash
kubectl get deployment apache
kubectl get replicasets
kubectl get pods -o wide
```

Cleanup:

```bash
kubectl delete deployment apache
```

## 5. Generate starter YAML with kubectl

```bash
kubectl run web --image=nginx:1.26 --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx:1.26 --dry-run=client -o yaml > deployment.yaml
kubectl explain pod.spec.containers
```

When using generated YAML, remove generated `status: {}` and `metadata.creationTimestamp` fields if present. Use spaces, never tabs.

## 6. Create a Pod from `pod.yaml`

Create `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
    - name: web
      image: nginx:1.26
      ports:
        - containerPort: 80
```

Apply and inspect:

```bash
kubectl apply -f pod.yaml
kubectl get pods -o wide
kubectl describe pod web
kubectl get pod web -o yaml
```

Delete it:

```bash
kubectl delete pod web
kubectl get pods
```

## 7. Create a Deployment from `deployment.yaml`

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.26
          ports:
            - containerPort: 80
```

Apply and inspect:

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods
kubectl describe deployment web
```

Delete one Pod and watch the Deployment replace it:

```bash
kubectl get pods
kubectl delete pod <ONE_WEB_POD_NAME>
kubectl get pods -w
```

Scale using the YAML file: edit `replicas:` and apply again.

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

Imperative scaling also works:

```bash
kubectl scale deployment web --replicas=5
kubectl get pods
```

The manifest still says `replicas: 3` unless you edit it. Applying the manifest again returns the Deployment to three replicas.

## 8. Add a NodePort Service

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Apply and verify endpoints:

```bash
kubectl apply -f service.yaml
kubectl get svc web-svc
kubectl get endpoints web-svc
minikube service web-svc --url
```

On Linux, the NodePort can also be reached using the Minikube node IP:

```bash
echo "http://$(minikube ip):30080"
```

## 9. Check Service selectors

The Service selector must match Pod labels. For this example, both use `app: web`.

```bash
kubectl get pods --show-labels
kubectl get pods -l app=web
kubectl get endpoints web-svc
```

Deliberate selector failure test: temporarily change the selector in `service.yaml` to `app: wrong`, then:

```bash
kubectl apply -f service.yaml
kubectl get endpoints web-svc
```

No endpoints means the Service is not selecting the Pods. Restore `app: web` and apply again:

```bash
kubectl apply -f service.yaml
kubectl get endpoints web-svc
```

## 10. Change the Service to ClusterIP

Replace the Service definition with:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

Apply and test from inside the cluster:

```bash
kubectl apply -f service.yaml
kubectl get svc web-svc
kubectl run tmp --rm -it --image=busybox -- wget -qO- http://web-svc
```

## 11. One manifest for Deployment + Service

Create `k8s/deployment.yaml`:

```bash
mkdir -p k8s
nano k8s/deployment.yaml
```

Use this two-object YAML file (the `---` separator splits the Deployment and Service):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cdac-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cdac-web
  template:
    metadata:
      labels:
        app: cdac-web
    spec:
      containers:
        - name: web
          image: cdac-web:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: cdac-web
spec:
  type: NodePort
  selector:
    app: cdac-web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Build the local demo image inside Minikube, then deploy:

```bash
git clone https://github.com/newdelthis/kubernetes-static.git
cd kubernetes-static

eval "$(minikube docker-env)"
# Follow the project's README to build the cdac-web:v1 image.
kubectl apply -f k8s/deployment.yaml
kubectl get deployments
kubectl get pods -o wide
kubectl get svc cdac-web
minikube service cdac-web --url
```

The source requires following the cloned repository's README for the image build; it does not include the Docker build instructions in the slide deck.

## 12. Rolling update to NGINX 1.27

Edit the existing Deployment manifest so it includes the rollout strategy and new image:

```yaml
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 10
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Run the rollout:

```bash
# Terminal 2: watch the Pods
kubectl get pods -w

# Terminal 1:
kubectl apply -f deployment.yaml
kubectl rollout status deployment/web
kubectl get rs
kubectl exec deploy/web -- nginx -v
kubectl rollout history deployment/web
```

## 13. Demonstrate rollback after a bad image update

Change the image in `deployment.yaml` to an invalid tag, for example `nginx:9.99`, and apply:

```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl rollout status deployment/web
```

Stop waiting for the rollout if it remains stuck, then roll back:

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
kubectl get pods
```

Restore the manifest to the valid image before applying it again.

## 14. Rollout using the separate demo repository

```bash
cd ~
git clone https://github.com/newdelthis/rolling_initial.git
cd rolling_initial

kubectl apply -f deployment.yml
kubectl apply -f service.yml
minikube service cdac-service
```

In another terminal:

```bash
kubectl get pods -w
```

Change the image in `deployment.yml` to `newdelthis/rolling:2.0`, then:

```bash
kubectl apply -f deployment.yml
kubectl rollout status deployment/cdac-demo
kubectl get rs
kubectl rollout history deployment/cdac-demo
kubectl rollout undo deployment/cdac-demo
```

## 15. Troubleshooting commands

```bash
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
kubectl logs <POD_NAME> --previous
kubectl get endpoints <SERVICE_NAME>
kubectl get pods --show-labels
kubectl get deploy,rs,svc
kubectl get pod <POD_NAME> -o yaml
```

## 16. Minikube controls

```bash
minikube status
minikube service <SERVICE_NAME> --url
minikube tunnel
minikube stop
```

Source reference: Kubernetes slides 18–52.
