# Kubernetes: Rolling Updates, Rollback, and Troubleshooting

Source coverage: Kubernetes slides 44-52.

## 1. Prepare a Deployment that supports a controlled rolling update

Example `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
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
          image: nginx:1.26
          ports:
            - containerPort: 80
```

`maxSurge: 1` allows one Pod above the desired five temporarily. `maxUnavailable: 1` allows at most one Pod below the desired available count during the rollout. `minReadySeconds` gives each new Pod time in the Ready state before Kubernetes treats it as available.

Apply initial version and inspect:

```bash
kubectl apply -f deployment.yaml
kubectl get deploy,rs,pods
```

## 2. Roll forward from NGINX 1.26 to 1.27

Edit `deployment.yaml` and change the image tag to `nginx:1.27`. In Terminal 2:

```bash
kubectl get pods -w
```

In Terminal 1:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/web
kubectl get rs
kubectl exec deploy/web -- nginx -v
kubectl rollout history deployment/web
```

Use a versioned image tag. If you use `nginx:latest` both before and after, the tag name alone does not identify a new version to roll out.

## 3. Trigger and recover from a bad image update

Edit `deployment.yaml` and set an image tag that does not exist, for example `nginx:9.99`:

```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl describe pod <NEW_POD_NAME>
```

The new Pods should fail to pull the image. The rollout may never complete. Stop waiting with `Ctrl+C` if you are watching it, then roll back:

```bash
kubectl rollout status deployment/web
# If it stays pending, press Ctrl+C.
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
kubectl get pods
```

**Important:** also restore the manifest file to the last working image tag. Otherwise, the next `kubectl apply -f deployment.yaml` will reintroduce the bad tag.

## 4. Rollout commands

```bash
kubectl rollout status deployment/<DEPLOYMENT_NAME>
kubectl rollout history deployment/<DEPLOYMENT_NAME>
kubectl rollout undo deployment/<DEPLOYMENT_NAME>
kubectl set image deployment/<DEPLOYMENT_NAME> <CONTAINER_NAME>=<IMAGE:TAG>
kubectl get rs
kubectl get pods -w
```

## 5. Separate rolling-update demo repository from the slides

The source deck names this repository and its files:

```bash
cd ~
git clone https://github.com/newdelthis/rolling_initial.git
cd rolling_initial
kubectl apply -f deployment.yml
kubectl apply -f service.yml
minikube service cdac-service --url
```

In a second terminal:

```bash
kubectl get pods -w
```

Edit `deployment.yml` and change its image to `newdelthis/rolling:2.0`, then:

```bash
kubectl apply -f deployment.yml
kubectl rollout status deployment/cdac-demo
kubectl get rs
kubectl rollout history deployment/cdac-demo
kubectl rollout undo deployment/cdac-demo
```

The Service does not need re-applying when only the Deployment's Pod template image changes. The remote repository's actual manifest contents are maintained by its owner and can change independently of this local handbook.

## 6. Common errors and first checks

| Symptom | First check |
|---|---|
| Pod is `Pending` | `kubectl describe pod <POD_NAME>`; inspect Events and available CPU/memory. |
| `ErrImagePull` / `ImagePullBackOff` | `kubectl describe pod <POD_NAME>`; check image name/tag and whether local image is in Minikube's store. |
| `CrashLoopBackOff` | `kubectl logs <POD_NAME> --previous` and `kubectl describe pod <POD_NAME>`. |
| Service gives no response | `kubectl get endpoints <SERVICE_NAME>`; verify selector matches Pod labels. |
| Apply rejects Deployment selector | Compare `spec.selector.matchLabels` with `spec.template.metadata.labels`. |
| YAML parse error | Check spaces/indentation; do not use tabs. |
| Rollout never completes | Check Pod readiness/events, then `kubectl rollout undo deployment/<NAME>` if needed. |

`kubectl describe` includes an Events section near the bottom; read it before guessing.

## 7. kubectl quick reference

```bash
# Look
kubectl get pods -o wide
kubectl get deploy,rs,svc
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
kubectl get pod <POD_NAME> -o yaml

# Change
kubectl apply -f <FILE.yaml>
kubectl delete -f <FILE.yaml>
kubectl scale deployment <DEPLOYMENT> --replicas=3
kubectl exec <POD_NAME> -- <COMMAND>
kubectl explain <KIND.FIELD>

# Rollouts
kubectl rollout status deployment/<DEPLOYMENT>
kubectl rollout history deployment/<DEPLOYMENT>
kubectl rollout undo deployment/<DEPLOYMENT>
kubectl set image deployment/<DEPLOYMENT> <CONTAINER>=<IMAGE:TAG>

# Minikube
minikube start --driver=docker
minikube status
minikube service <SERVICE> --url
minikube tunnel
minikube stop
```
