# Assignment 6: Kubernetes and Flask Deployment

This is a complete practical solution to Assignment 6, Q1-Q9. It assumes Minikube is running. Setup instructions: [Install and start Minikube](../kubernetes/01-minikube-setup.md).

Check cluster status before starting:

```bash
minikube status
kubectl get nodes
```

## Q1. Run an Apache Pod named `apache-pod`

```bash
kubectl run apache-pod --image=httpd --restart=Never
kubectl get pods
kubectl describe pod apache-pod
```

What the commands do:

- `kubectl run` creates a Pod.
- `--image=httpd` uses the official Apache HTTP Server image.
- `--restart=Never` asks for a standalone Pod rather than a Deployment.
- `get` checks status; `describe` shows details and scheduling/image-pull events.

Optional: view the Apache container logs:

```bash
kubectl logs apache-pod
```

## Q2. Create an Apache Deployment named `apache-deployment`

```bash
kubectl create deployment apache-deployment --image=httpd
kubectl get deployments
kubectl get pods
kubectl describe deployment apache-deployment
```

A Deployment manages a ReplicaSet, which creates and replaces Pods to maintain the requested replica count.

## Q3. Increase replicas to 4, then decrease to 2

Scale up:

```bash
kubectl scale deployment apache-deployment --replicas=4
kubectl get deployment apache-deployment
kubectl get rs
kubectl get pods
```

Scale down:

```bash
kubectl scale deployment apache-deployment --replicas=2
kubectl get deployment apache-deployment
kubectl get rs
kubectl get pods
```

The `--replicas` value is the desired number of Pods. Kubernetes creates or removes Pods until the actual count converges to that value.

## Q4. Create the Apache Deployment using YAML with 3 replicas

Create a folder and file:

```bash
mkdir -p ~/assignment-6-kubernetes
cd ~/assignment-6-kubernetes
```

Create `apache-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
        - name: apache
          image: httpd
          ports:
            - containerPort: 80
```

Apply and verify all related resources:

```bash
kubectl apply -f apache-deployment.yaml
kubectl get deployments
kubectl get rs
kubectl get pods --show-labels
kubectl get all
```

`kubectl apply -f` creates or updates the resources described in the file. This YAML sets the desired replica count to three. If you earlier scaled the same Deployment to two with `kubectl scale`, applying this file sets it back to three.

## Q5. Expose Apache using a ClusterIP Service named `apache-clusterip`

Create `apache-clusterip.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-clusterip
spec:
  type: ClusterIP
  selector:
    app: apache
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply and check endpoints:

```bash
kubectl apply -f apache-clusterip.yaml
kubectl get svc apache-clusterip
kubectl get endpoints apache-clusterip
```

Test from inside the Kubernetes cluster using a temporary BusyBox Pod:

```bash
kubectl run test-client \
  --rm -it \
  --restart=Never \
  --image=busybox:1.36 \
  -- wget -qO- http://apache-clusterip
```

The NGINX/Apache welcome HTML should print in the terminal. A ClusterIP address is internal to the cluster; it is not meant to be opened directly from your laptop browser.

## Q6. Expose Apache using a NodePort Service named `apache-nodeport`

Create `apache-nodeport.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-nodeport
spec:
  type: NodePort
  selector:
    app: apache
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
```

Apply and verify:

```bash
kubectl apply -f apache-nodeport.yaml
kubectl get svc apache-nodeport
kubectl get endpoints apache-nodeport
```

Get the Minikube URL:

```bash
minikube service apache-nodeport --url
```

Open the returned URL in a browser. On a Linux Minikube Docker-driver setup, you can also try:

```bash
echo "http://$(minikube ip):30080"
```

If that address is not reachable on your platform, use the URL returned by `minikube service`.

## Q7. Expose Apache using a LoadBalancer Service named `apache-loadbalancer`

Create `apache-loadbalancer.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: apache
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply and inspect:

```bash
kubectl apply -f apache-loadbalancer.yaml
kubectl get svc apache-loadbalancer --watch
```

In a separate terminal, keep the Minikube tunnel running:

```bash
minikube tunnel
```

Return to the first terminal and check the external IP:

```bash
kubectl get svc apache-loadbalancer
```

Use the displayed external IP in the browser, for example `http://<EXTERNAL_IP>`. The IP is assigned by the local Minikube tunnel; without the tunnel, `EXTERNAL-IP` can remain `pending`.

## Q8. Build and deploy a Flask app showing PRN and student name

### 8.1 Create the application

```bash
mkdir -p ~/assignment-6-kubernetes/flask-app
cd ~/assignment-6-kubernetes/flask-app
```

Create `app.py` (initial version: PRN and name only):

```python
import os
from flask import Flask

app = Flask(__name__)

PRN = os.getenv("STUDENT_PRN", "REPLACE_WITH_YOUR_PRN")
STUDENT_NAME = os.getenv("STUDENT_NAME", "REPLACE_WITH_YOUR_NAME")


@app.route("/")
def home():
    return f"""<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>Student Details</title></head>
<body>
  <h1>Student Details</h1>
  <p><strong>PRN:</strong> {PRN}</p>
  <p><strong>Student Name:</strong> {STUDENT_NAME}</p>
</body>
</html>"""


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Create `requirements.txt`:

```text
Flask==3.1.2
```

Create `Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

`0.0.0.0` is important: it makes Flask listen on the container interface instead of only the container's loopback interface.

### 8.2 Build the image inside Minikube

Replace the tag if you prefer another image name. Build inside Minikube so the Kubernetes node can find the image without pulling it from Docker Hub:

```bash
cd ~/assignment-6-kubernetes/flask-app
minikube image build -t flask-student-app:v1 .
minikube image ls | grep flask-student-app
```

### 8.3 Create the Flask Deployment

Create `flask-deployment.yaml` in `~/assignment-6-kubernetes/`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-student
  template:
    metadata:
      labels:
        app: flask-student
    spec:
      containers:
        - name: flask-app
          image: flask-student-app:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
          env:
            - name: STUDENT_PRN
              value: "YOUR_PRN"
            - name: STUDENT_NAME
              value: "YOUR_NAME"
```

Edit `YOUR_PRN` and `YOUR_NAME` before applying. Do not leave those placeholders unchanged.

### 8.4 Create a Service for the Flask app

Create `flask-service.yaml` in `~/assignment-6-kubernetes/`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: NodePort
  selector:
    app: flask-student
  ports:
    - name: http
      port: 80
      targetPort: 5000
      nodePort: 30081
```

The Service receives traffic on port 80 and forwards it to Flask's port 5000 in the Pod. NodePort 30081 exposes it outside the cluster.

Apply and verify:

```bash
cd ~/assignment-6-kubernetes
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
kubectl get deployments
kubectl get pods -o wide
kubectl get svc flask-service
kubectl get endpoints flask-service
```

Get the browser URL and open it:

```bash
minikube service flask-service --url
```

The page should display your PRN and student name.

## Q9. Rolling update: add course name and build image version 2

### 9.1 Modify `app.py`

Replace the contents of `flask-app/app.py` with this updated version. It now displays the course name:

```python
import os
from flask import Flask

app = Flask(__name__)

PRN = os.getenv("STUDENT_PRN", "REPLACE_WITH_YOUR_PRN")
STUDENT_NAME = os.getenv("STUDENT_NAME", "REPLACE_WITH_YOUR_NAME")
COURSE_NAME = os.getenv("COURSE_NAME", "PG-CPITISS")


@app.route("/")
def home():
    return f"""<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>Student Details</title></head>
<body>
  <h1>Student Details - Updated Version 2</h1>
  <p><strong>PRN:</strong> {PRN}</p>
  <p><strong>Student Name:</strong> {STUDENT_NAME}</p>
  <p><strong>Course Name:</strong> {COURSE_NAME}</p>
</body>
</html>"""


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 9.2 Build a new image tag

```bash
cd ~/assignment-6-kubernetes/flask-app
minikube image build -t flask-student-app:v2 .
minikube image ls | grep flask-student-app
```

Use a new tag (`v2`) rather than reusing `v1`. This makes the Deployment's Pod-template change explicit and avoids ambiguity from cached image tags.

### 9.3 Update the Deployment manifest

Edit `~/assignment-6-kubernetes/flask-deployment.yaml`:

- Change `image: flask-student-app:v1` to `image: flask-student-app:v2`.
- Add the `COURSE_NAME` environment variable.

The final container section should be:

```yaml
      containers:
        - name: flask-app
          image: flask-student-app:v2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
          env:
            - name: STUDENT_PRN
              value: "YOUR_PRN"
            - name: STUDENT_NAME
              value: "YOUR_NAME"
            - name: COURSE_NAME
              value: "PG-CPITISS"
```

Keep your actual PRN and name values from Q8 when editing the file.

### 9.4 Apply and verify the rolling update

In Terminal 2, watch Pods:

```bash
kubectl get pods -w
```

In Terminal 1:

```bash
cd ~/assignment-6-kubernetes
kubectl apply -f flask-deployment.yaml
kubectl rollout status deployment/flask-deployment
kubectl rollout history deployment/flask-deployment
kubectl get replicasets
kubectl get pods
```

Open the existing Service URL again:

```bash
minikube service flask-service --url
```

The page should now display PRN, student name, and course name. The Service does not need to be re-created because its selector and port have not changed.

### 9.5 Roll back if version 2 fails

```bash
kubectl rollout undo deployment/flask-deployment
kubectl rollout status deployment/flask-deployment
kubectl get pods
```

If you roll back, also change the manifest image tag back to `flask-student-app:v1`; otherwise the next `kubectl apply` will roll forward to v2 again.

## Cleanup after the assignment

Delete the resources created for the Apache and Flask exercises:

```bash
kubectl delete service apache-clusterip apache-nodeport apache-loadbalancer flask-service --ignore-not-found
kubectl delete deployment apache-deployment flask-deployment --ignore-not-found
kubectl delete pod apache-pod --ignore-not-found
```

If you want to delete the entire local cluster and all its data:

```bash
minikube delete  # DESTRUCTIVE: deletes the Minikube cluster
```

## If a Pod does not start

```bash
kubectl get pods
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
kubectl get events --sort-by=.lastTimestamp
```

For `ImagePullBackOff`, confirm you built the image inside Minikube with the exact tag in the Deployment YAML. For a Service with no response, check that its selector matches the Pod labels and that endpoints exist:

```bash
kubectl get pods --show-labels
kubectl get endpoints flask-service
kubectl describe service flask-service
```
