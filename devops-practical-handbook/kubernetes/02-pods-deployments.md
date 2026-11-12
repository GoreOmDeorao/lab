# Kubernetes: Pods, Deployments, and Self-Healing

Source coverage: Kubernetes slides 20-22; Assignment 6, Q1-Q3. Assumes Minikube is running and `kubectl` works.

## 1. Create and inspect an NGINX Pod

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

`kubectl run` creates a Pod in this command form. `get` lists objects, `describe` shows detailed status/events, `logs` prints container output, and `exec` runs a command inside the container.

Delete the Pod:

```bash
kubectl delete pod nginx
```

A bare Pod is not automatically recreated after it is deleted.

## 2. Create and inspect a Deployment

```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl get pods
kubectl describe deployment nginx
kubectl get deployment nginx -o yaml
kubectl get pods -o wide
```

Find one Pod name and inspect it:

```bash
kubectl get pods
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
```

Scale to three replicas:

```bash
kubectl scale deployment nginx --replicas=3
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

Delete the Deployment and the Pods it manages:

```bash
kubectl delete deployment nginx
```

## 3. Demonstrate self-healing with Apache

Create an Apache (`httpd`) Deployment with five replicas:

```bash
kubectl create deployment apache --image=httpd --replicas=5
kubectl get deployment apache
kubectl get replicasets
kubectl get pods -o wide
```

In Terminal 2, watch Pods:

```bash
kubectl get pods -w
```

In Terminal 1, choose one Apache Pod and delete it:

```bash
kubectl get pods
POD_NAME="replace-with-one-apache-pod-name"
kubectl delete pod "$POD_NAME"
```

Terminal 2 should show the deleted Pod disappearing and a replacement Pod appearing. Verify the desired replica count:

```bash
kubectl get deployment apache
kubectl get replicasets
kubectl get pods -o wide
```

The replacement Pod has a new name and may have a new IP. Kubernetes maintains the desired number of replicas; it does not restart the exact same Pod object.

Cleanup:

```bash
kubectl delete deployment apache
```

## 4. Watch a Pod or Deployment

```bash
kubectl get pods -w
kubectl get deployments -w
kubectl get deploy,rs,pods
```

Press `Ctrl+C` to stop watching. The `deploy,rs,pods` command shows the Deployment, ReplicaSet, and Pod resources together.

## 5. Assignment 6 Apache tasks

The complete solution for all nine questions (including Pod, Deployment, scaling, YAML, Services, Flask, and rolling update) is in [Assignment 6: Kubernetes practical](../assignments/assignment-6-kubernetes.md).
