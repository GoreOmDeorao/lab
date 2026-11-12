# Jenkins + Docker: Build, Push, and Deploy an NGINX Image

Source coverage: Jenkins slides 23-38. This guide combines the source examples into one consistent Pipeline and fixes mismatched credential IDs and unsafe socket-permission advice.

## 1. Install Docker Pipeline plugin

In Jenkins UI:

1. Open **Manage Jenkins → Plugins**.
2. Install **Docker Pipeline**.
3. Ensure the **Git** plugin is installed.

Docker Engine must be installed on the machine/agent that runs the Pipeline.

## 2. Give Jenkins access to Docker for this lab

The source lab adds the `jenkins` service account to the `docker` group:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Check the account and socket:

```bash
id jenkins
getent group docker
ls -l /var/run/docker.sock
sudo systemctl status docker --no-pager
sudo systemctl status jenkins --no-pager
```

The Jenkins process must restart after group membership changes. If permission is still denied, confirm that Docker is running and that Jenkins is using the expected host/agent.

**Never run `sudo chmod 777 /var/run/docker.sock`.** That makes the Docker daemon socket writable by every local user. Access to the Docker socket can grant root-equivalent control of the host. Docker-group membership is also highly privileged; grant it only on a dedicated trusted lab/CI host. For production, prefer an isolated build agent or controlled remote builder.

## 3. Prepare the GitHub NGINX project

Create a GitHub repository, clone it, and create this structure:

```text
my-nginx-app/
├── Dockerfile
├── Jenkinsfile
└── html/
    └── index.html
```

`html/index.html`:

```html
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>CDAC NGINX app</title></head>
<body><h1>Welcome to C-DAC - Custom NGINX page</h1></body>
</html>
```

`Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY ./html/ /usr/share/nginx/html/
EXPOSE 80
```

`COPY` places the HTML content in NGINX's default document root. `EXPOSE 80` documents the port; it does not publish the port on the host.

Commit and push the source files:

```bash
git add Dockerfile html/index.html
git commit -m "Add custom NGINX website"
git push origin main
```

## 4. Add Docker Hub credentials in Jenkins

Use a Docker Hub access token where possible rather than your account password.

In Jenkins:

1. Open **Manage Jenkins → Credentials**.
2. Under the **Global** domain, click **Add Credentials**.
3. Kind = **Username with password**.
4. Username = your Docker Hub username.
5. Password = your Docker Hub access token.
6. ID = `docker-hub-credentials`.
7. Save.

The credential ID must exactly match the string used in the Jenkinsfile below.

## 5. Jenkinsfile: checkout, build, push, and deploy

Create `Jenkinsfile` in the GitHub repository. Replace both placeholders before committing it:

- `<YOUR_GITHUB_USERNAME>` and `<YOUR_REPOSITORY>` identify your GitHub repository.
- `<YOUR_DOCKERHUB_USERNAME>` is your Docker Hub namespace.

```groovy
pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/<YOUR_GITHUB_USERNAME>/<YOUR_REPOSITORY>.git'
        DOCKER_IMAGE = '<YOUR_DOCKERHUB_USERNAME>/your-nginx-app'
        IMAGE_TAG = '1.0'
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: env.GITHUB_REPO
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${env.DOCKER_IMAGE}:${env.IMAGE_TAG}", '-f Dockerfile .')
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS_ID) {
                        docker.image("${env.DOCKER_IMAGE}:${env.IMAGE_TAG}").push()
                    }
                }
            }
        }

        stage('Deploy Container on Jenkins Host') {
            steps {
                sh 'docker rm -f nginx-app-container || true'
                sh "docker run -d --restart unless-stopped -p 8081:80 --name nginx-app-container ${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
            }
        }
    }

    post {
        success {
            echo 'Pipeline successfully executed.'
        }
        failure {
            echo 'Pipeline failed. Read the stage output in Console Output.'
        }
    }
}
```

**What the stages do:**

1. `Checkout` downloads the source repository.
2. `Build Docker Image` creates a local image from the Dockerfile.
3. `Push Image to Docker Hub` authenticates with the Jenkins credential and uploads the tagged image.
4. `Deploy Container` removes the old lab container (if it exists) and starts a new one.

The deploy stage targets the Jenkins host/agent where Docker runs. This is a teaching lab deployment, not a multi-host production deployment.

## 6. Create the Jenkins Pipeline job

In Jenkins:

1. Click **New Item**.
2. Name it `build-publish-docker-image`.
3. Select **Pipeline** and click **OK**.
4. In Pipeline, set **Definition = Pipeline script from SCM**.
5. SCM = **Git**.
6. Repository URL = your GitHub repository `.git` URL.
7. Branch Specifier = `*/main`.
8. Script Path = `Jenkinsfile`.
9. Save and click **Build Now**.

Open Console Output and read each stage. A successful deployment should be reachable from the Docker host at `http://localhost:8081`.

## 7. Verify the image and deployed container

```bash
docker image ls | grep your-nginx-app
docker ps --filter name=nginx-app-container
docker logs nginx-app-container
curl http://localhost:8081
```

If Jenkins runs on a remote VM, use the VM's reachable address and ensure its firewall allows port 8081. `localhost` from your own laptop is not the remote Jenkins VM.

## 8. Update the NGINX page

Edit `html/index.html`, commit and push the change:

```bash
git add html/index.html
git commit -m "Update NGINX page"
git push origin main
```

Run the Jenkins Pipeline again. The image is rebuilt, pushed, and the container is replaced.

## 9. Build and run only (no registry push)

For the smaller slide example that only builds and runs a Python image, use this Pipeline in a separate repository that contains `hello.py` and a `Dockerfile`:

```groovy
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'hello-world-python:latest'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    if (fileExists('Dockerfile')) {
                        sh "docker build -t ${env.DOCKER_IMAGE} ."
                    } else {
                        error 'Dockerfile not found in the workspace.'
                    }
                }
            }
        }
        stage('Docker Run') {
            steps {
                sh "docker run --rm ${env.DOCKER_IMAGE}"
            }
        }
    }
}
```

## 10. Common problems and corrections from the source slides

- **Credential ID mismatch:** the slide text uses more than one ID (`docker-hub-password-id`, `docker-hub-credentials`). Create one Jenkins credential and use the exact same ID everywhere. This guide uses `docker-hub-credentials`.
- **Docker permission denied:** restart Jenkins after adding it to the Docker group and inspect the service account/group membership. Do not loosen the socket to mode 777.
- **Wrong repository or branch:** verify the `.git` URL and branch `main` in the Jenkins job.
- **Wrong image name:** Docker Hub image names should be `username/repository:tag` for a user repository.
- **Port 80 already used:** this guide publishes NGINX on host port 8081 to avoid common port 80/8080 conflicts.
