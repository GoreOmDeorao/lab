# Kubernetes: YAML Manifests and Services

Source coverage: Kubernetes slides 24-43; Assignment 6, Q4-Q7.

## 1. Generate a YAML starting point

```bash
kubectl run web --image=nginx:1.26 --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx:1.26 --dry-run=client -o yaml > deployment.yaml
kubectl explain pod.spec.containers
```

`--dry-run=client` constructs an object locally without creating it in the cluster. `-o yaml` prints the object as YAML and `>` saves it to a file. Generated YAML may contain server-managed fields such as `status` and `creationTimestamp`; remove these before treating it as your source manifest.

## 2. Pod manifest

`pod.yaml`:

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

Apply, inspect, and delete:

```bash
kubectl apply -f pod.yaml
kubectl get pods -o wide
kubectl describe pod web
kubectl get pod web -o yaml
kubectl delete pod web
```

## 3. Deployment manifest

`deployment.yaml`:

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

Important: `spec.selector.matchLabels` must match `spec.template.metadata.labels`. The Deployment uses the selector to identify Pods it owns.

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods
kubectl describe deployment web
```

Delete one Pod and watch the ReplicaSet replace it:

```bash
kubectl get pods
POD_NAME="replace-with-one-web-pod-name"
kubectl delete pod "$POD_NAME"
kubectl get pods -w
```

Scale declaratively by changing `replicas:` in the YAML file and applying it again:

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

Imperative scaling also works:

```bash
kubectl scale deployment web --replicas=5
```

But if the manifest still says `replicas: 3`, a later `kubectl apply -f deployment.yaml` returns the Deployment to three replicas. Keep the file and live cluster state aligned.

## 4. NodePort Service

`service.yaml`:

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

- `port` is the port exposed by the Service inside the cluster.
- `targetPort` is the port on the selected Pod/container.
- `nodePort` is the port opened on each eligible node; the default allowed range is 30000-32767.

Apply and verify:

```bash
kubectl apply -f service.yaml
kubectl get svc web-svc
kubectl get endpoints web-svc
minikube service web-svc --url
```

On Linux you can also form the URL from the node IP:

```bash
echo "http://$(minikube ip):30080"
```

## 5. Service selector troubleshooting

A Service with the wrong selector can exist while sending traffic to no Pods. Check labels and endpoints:

```bash
kubectl get pods --show-labels
kubectl get pods -l app=web
kubectl get endpoints web-svc
```

To reproduce the problem, temporarily edit `service.yaml` and change `selector.app` from `web` to `wrong`, then apply:

```bash
kubectl apply -f service.yaml
kubectl get endpoints web-svc
```

If endpoints show `<none>`, no Pods match the selector. Restore `app: web`, then:

```bash
kubectl apply -f service.yaml
kubectl get endpoints web-svc
```

## 6. ClusterIP Service and in-cluster test

Change `service.yaml` to:

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

Apply and test from a temporary Pod:

```bash
kubectl apply -f service.yaml
kubectl get svc web-svc
kubectl run tmp-curl --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- http://web-svc
```

ClusterIP is reachable within the cluster; it is not intended as a public host port.

## 7. LoadBalancer Service on Minikube

`service-loadbalancer.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f service-loadbalancer.yaml
kubectl get svc web-lb --watch
```

In another terminal, run:

```bash
minikube tunnel
```

Return to the first terminal and inspect:

```bash
kubectl get svc web-lb
```

Minikube's external IP may remain `pending` until the tunnel is active. `minikube tunnel` must remain running while using that external IP.

## 8. One manifest containing Deployment and Service

A YAML document separator (`---`) lets one file define multiple Kubernetes objects. Example `k8s/deployment.yaml`:

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
          image: nginx:1.26
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

Apply/delete both resources together:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl get deploy,rs,pods,svc
kubectl delete -f k8s/deployment.yaml
```

## 9. Static-site demo repository referenced in the slides

The source deck points to `kubernetes-static` and says to follow that repository's README for the image build steps. The slide text does not include the complete source files or Docker build instructions, so they are not reproduced here.

```bash
cd ~
git clone https://github.com/newdelthis/kubernetes-static.git
cd kubernetes-static

eval "$(minikube docker-env)"   # Point Docker CLI at Minikube's image store for this shell
# Follow the repository README to build its cdac-web:v1 image.
kubectl apply -f k8s/deployment.yaml
kubectl get deployments
kubectl get pods -o wide
kubectl get svc cdac-web
minikube service cdac-web --url
```

When you finish, return the Docker CLI to its normal host context by opening a new terminal. The `eval` command only changes environment variables in the current shell.

## 10. Useful YAML and Service checks

```bash
kubectl explain pod.spec.containers
kubectl get pod <POD_NAME> -o yaml
kubectl get pods --show-labels
kubectl get endpoints <SERVICE_NAME>
kubectl get svc
kubectl describe service <SERVICE_NAME>
```

Use spaces, not tabs, for YAML indentation. Leave a space after the colon in `key: value` lines.
