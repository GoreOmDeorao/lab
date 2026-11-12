# Jenkins — Practical Labs

Procedures and code from Jenkins slides 4–38.

## 1. Install and unlock Jenkins on Ubuntu

Follow the [official Jenkins Linux installation instructions](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu).

After installation, open `http://localhost:8080`.

Get the initial password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Paste it into the Unlock Jenkins page, install suggested plugins, create your admin user, and keep the default instance URL.

## 2. Create a Freestyle job — Hello Jenkins

In Jenkins:

1. Select **New Item**.
2. Enter `hello_jenkins`.
3. Select **Freestyle project** and click **OK**.
4. Under **Build Steps**, select **Add build step → Execute shell**.
5. Add the command below, save, and click **Build Now**.
6. Open the build number in **Build History → Console Output**.

```bash
echo "Hello Jenkins!"
```

## 3. Freestyle job — shell variables and date

In the same job, replace the Execute shell content with:

```bash
NAME="C-DAC"
echo "Hello, $NAME. Current date and time is $(date)."
```

Save, click **Build Now**, and inspect **Console Output**.

## 4. Freestyle job — Python

```bash
python3 -c "print('Hello, World from Python!')"
```

## 5. Freestyle job — Java source-file mode

This command creates `HelloWorld.java` in the job's current workspace and runs it using Java source-file mode:

```bash
cat <<'JAVA_SOURCE' > HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World from Java in Jenkins!");
    }
}
JAVA_SOURCE

java HelloWorld.java
```

## 6. Run a Python script file from a Jenkins job

Create the script on the Linux machine:

```bash
sudo mkdir -p /opt/scripts
sudo tee /opt/scripts/hello.py >/dev/null <<'PYTHON_SCRIPT'
print('Hello, World')
print('from Python in Jenkins!')
PYTHON_SCRIPT
sudo chmod +x /opt/scripts/hello.py
```

In the Jenkins job's **Execute shell** build step:

```bash
python3 /opt/scripts/hello.py
```

## 7. Write job output to a file

In the job's **Execute shell** build step:

```bash
NAME="C-DAC"
echo "Hello, $NAME. Current date and time is $(date)." > /tmp/info
```

After the build, inspect the file on the machine:

```bash
cat /tmp/info
```

## 8. Create a Pipeline job

In Jenkins:

1. Go to **New Item**.
2. Enter `hello_jenkins_pipeline`.
3. Select **Pipeline** and click **OK**.
4. Under **Pipeline**, set **Definition = Pipeline script**.
5. Use **Try sample Pipeline → Hello World**, or paste this minimal pipeline:

```groovy
pipeline {
    agent any
    stages {
        stage('Hello') {
            steps {
                echo 'Hello from Jenkins Pipeline'
            }
        }
    }
}
```

6. Save, click **Build Now**, and inspect **Console Output**.

## 9. Pipeline — create and append to a workspace file

Paste this into a Pipeline job:

```groovy
pipeline {
    agent any
    stages {
        stage('Assemble') {
            steps {
                echo 'Assemble a new laptop ...'
                sh 'mkdir -p assemble'
                sh 'touch assemble/computer.txt'
                sh 'echo "Motherboard" >> assemble/computer.txt'
                sh 'cat assemble/computer.txt'
                sh 'echo "Display" >> assemble/computer.txt'
                sh 'cat assemble/computer.txt'
                sh 'echo "CPU" >> assemble/computer.txt'
                sh 'cat assemble/computer.txt'
            }
        }
    }
}
```

The file is in the job workspace under `assemble/computer.txt`. Repeated builds append more lines because the commands use `>>`.

## 10. Install GitHub and Docker Pipeline plugins

In Jenkins:

1. Open **Manage Jenkins → Plugins**.
2. Search for and install the GitHub integration plugin.
3. Search for and install Docker Pipeline.
4. Wait for installation to finish.

## 11. Prepare and push a GitHub repository

Create an empty GitHub repository, then on the Linux machine:

```bash
cd ~
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd <YOUR_REPOSITORY_DIRECTORY>
```

Create `say_hello.sh`:

```bash
cat <<'SHELL_SCRIPT' > say_hello.sh
#!/usr/bin/env bash
echo "Hello from Git"
SHELL_SCRIPT

chmod u+x say_hello.sh
```

Configure Git identity if it is not configured already, then commit and push:

```bash
git config --global user.name "<YOUR_NAME>"
git config --global user.email "<YOUR_EMAIL>"
git add say_hello.sh
git commit -m "First commit of say hello"
git log
git push origin main
```

## 12. Jenkins Freestyle job — build from GitHub

In Jenkins:

1. **New Item** → name `my-github-project` → **Freestyle project**.
2. Under **Source Code Management**, select **Git**.
3. Paste your GitHub repository URL ending in `.git`.
4. Set **Branch Specifier** to `*/main`.
5. Under **Build Steps**, add **Execute shell**.
6. Enter `./say_hello.sh`.
7. Save and click **Build Now**.

## 13. Poll SCM schedules

In the job's **Build Triggers**, enable **Poll SCM** and enter one of these schedules:

```text
* * * * *       # Every minute
0 * * * *       # Every hour
0 0 * * *       # Every day at midnight
0 0 1 * *       # First day of every month
0 0 * * 0       # Every Sunday
0 8 * * 1       # Every Monday at 8:00 AM
*/5 * * * *     # Every 5 minutes
0 */2 * * *     # Every 2 hours
30 9 * * 1-5    # Weekdays at 9:30 AM
0 18 * * 1-5    # Weekdays at 6:00 PM
```

## 14. Pipeline with two user input parameters

Create `find_largest.py` in the GitHub repository:

```python
import os
import sys


def main():
    try:
        num1 = float(os.environ.get("NUM1", "0"))
        num2 = float(os.environ.get("NUM2", "0"))
        print(f"The largest number is: {max(num1, num2)}")
    except ValueError:
        print("Error: Please ensure you entered valid numeric values.")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

Create `Jenkinsfile` in the same repository:

```groovy
pipeline {
    agent any

    parameters {
        string(name: 'NUM1', defaultValue: '0', description: 'Enter the first number')
        string(name: 'NUM2', defaultValue: '0', description: 'Enter the second number')
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Run Python Program') {
            steps {
                sh 'python3 find_largest.py'
            }
        }
    }
}
```

Commit both files and push them to `main`:

```bash
chmod +x find_largest.py Jenkinsfile
git add find_largest.py Jenkinsfile
git commit -m "Add parameterized Jenkins pipeline"
git push origin main
```

In Jenkins, create a **Pipeline** job and configure:

- **Definition:** Pipeline script from SCM
- **SCM:** Git
- **Repository URL:** your repository `.git` URL
- **Branch Specifier:** `*/main`
- **Script Path:** `Jenkinsfile`

Save and click **Build Now**. Supply `NUM1` and `NUM2` when prompted.

## 15. Allow Jenkins to run Docker commands

Add the Jenkins service user to the Docker group, then restart Jenkins:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Check whether Docker works for the Jenkins service user:

```bash
sudo -u jenkins docker version
```

If permission is still denied, check group membership and the Docker service before changing socket permissions:

```bash
getent group docker
id jenkins
ls -l /var/run/docker.sock
sudo systemctl status docker
sudo systemctl status jenkins
```

> The slide includes `sudo chmod 777 /var/run/docker.sock`. Do **not** use that command: it makes the Docker control socket writable by every local user. Membership in the Docker group already grants powerful host control and should be limited to trusted service accounts.

## 16. Dockerfile and NGINX app for Jenkins pipeline

Create `index.html` in your GitHub repository:

```html
<h1>Hello from C-DAC</h1>
```

Create `Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

For the alternate slide layout, create `html/index.html` and use this Dockerfile instead:

```dockerfile
FROM nginx:alpine
COPY ./html /usr/share/nginx/html
EXPOSE 80
```

## 17. Configure Docker Hub credentials in Jenkins

In Jenkins:

1. Open **Manage Jenkins → Credentials**.
2. Under **Global**, choose **Add Credentials**.
3. Select **Username with password**.
4. Enter your Docker Hub username and password/token.
5. Set the ID to `docker-hub-credentials`.
6. Save.

Use a Docker Hub access token where available rather than your account password.

## 18. Jenkinsfile — checkout, build, push, and deploy NGINX

Replace the repository and Docker Hub placeholders. Ensure the Jenkins Docker Pipeline plugin is installed and the Jenkins agent can use Docker.

`Jenkinsfile`:

```groovy
pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/<YOUR_GITHUB_USERNAME>/<YOUR_REPOSITORY>.git'
        DOCKER_IMAGE = '<YOUR_DOCKERHUB_USERNAME>/your-nginx-app'
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
                    docker.build("${env.DOCKER_IMAGE}:1.0", '-f Dockerfile .')
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', env.DOCKER_CREDENTIALS_ID) {
                        docker.image("${env.DOCKER_IMAGE}:1.0").push()
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh 'docker rm -f webapp-container || true'
                sh "docker run -d -p 8081:80 --name webapp-container ${env.DOCKER_IMAGE}:1.0"
            }
        }
    }

    post {
        success {
            echo 'Pipeline successfully executed.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
```

In Jenkins, create a Pipeline job using **Pipeline script from SCM**, select Git, enter the repository URL, set branch `*/main`, and set Script Path to `Jenkinsfile`. Then save and build.

Open `http://localhost:8081` on the Jenkins/Docker host. The pipeline maps host port 8081 to container port 80 so it does not conflict with Jenkins, which commonly listens on host port 8080.

## 19. Jenkins Docker example — source mismatch

The source's earlier example stores a credential ID as `docker-hub-password-id`, while the `docker.withRegistry` call uses `docker-hub-credentials`. Those IDs must match the Jenkins credential you created. Use the complete Jenkinsfile above, which uses one ID consistently.

Source reference: Jenkins slides 4–38.
