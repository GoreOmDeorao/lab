# Jenkins: Pipeline Jobs, GitHub, and User Parameters

Source coverage: Jenkins slides 5, 12-22.

## 1. Create a basic Pipeline job

In Jenkins:

1. Click **New Item**.
2. Enter `hello_jenkins_pipeline`.
3. Select **Pipeline** and click **OK**.
4. In the Pipeline section set **Definition = Pipeline script**.
5. Paste the script below, save, and click **Build Now**.
6. Open **Console Output** to inspect the run.

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

A Pipeline job stores its stages and steps as code. A `Jenkinsfile` can be committed to Git along with the application.

## 2. Pipeline that assembles a file in the workspace

Paste into a Pipeline job:

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

The file is stored in the job workspace at `assemble/computer.txt`. `>>` appends. Therefore, repeated builds accumulate additional lines unless the file is removed or truncated first. `>` overwrites instead.

## 3. Install the plugins used by the labs

Open **Manage Jenkins → Plugins**. Install/check for:

- Git plugin (for Git SCM checkout).
- GitHub integration plugin (when using GitHub-specific integrations/webhooks).
- Docker Pipeline plugin (for `docker.build` and `docker.withRegistry` in Pipeline code).

Plugin names and UI labels may vary slightly by Jenkins release. Restart Jenkins only when the UI requests it or a plugin requires it.

## 4. Create a shell script and push it to GitHub

Create an empty GitHub repository first. Then on the Linux machine:

```bash
cd ~
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd <YOUR_REPOSITORY_DIRECTORY>
```

Replace both placeholders with the HTTPS or SSH URL and the resulting directory name.

Create `say_hello.sh`:

```bash
cat <<'SHELL_SCRIPT' > say_hello.sh
#!/usr/bin/env bash
echo "Hello from Git"
SHELL_SCRIPT
chmod u+x say_hello.sh
```

Configure your Git identity once (use your own identity, not the lecturer's values):

```bash
git config --global user.name "YOUR_NAME"
git config --global user.email "YOUR_EMAIL"
```

Commit and push:

```bash
git add say_hello.sh
git commit -m "Add say hello script"
git log -1
git push origin main
```

If your repository's default branch is `master` or another name, use that branch instead of `main`.

## 5. Create a Freestyle Jenkins job that clones GitHub

In Jenkins:

1. **New Item** → name `my-github-project` → **Freestyle project** → **OK**.
2. Under **Source Code Management**, select **Git**.
3. Enter your repository URL ending in `.git`.
4. Set Branch Specifier to `*/main` (adjust if your default branch differs).
5. Under **Build Steps**, select **Execute shell**.
6. Enter `./say_hello.sh`.
7. Save and click **Build Now**.

If Jenkins reports permission denied, verify the executable bit in the repository (`git ls-files --stage say_hello.sh`) and the agent's workspace permissions.

## 6. Poll SCM schedules

In the job configuration, enable **Build Triggers → Poll SCM**. Enter one schedule in the input field, without adding explanatory comments:

| Schedule | Meaning |
|---|---|
| `* * * * *` | Every minute |
| `0 * * * *` | At minute 0 every hour |
| `0 0 * * *` | Daily at midnight |
| `0 0 1 * *` | First day of each month |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 8 * * 1` | Every Monday at 08:00 |
| `*/5 * * * *` | Every five minutes |
| `0 */2 * * *` | Every two hours |
| `30 9 * * 1-5` | Weekdays at 09:30 |
| `0 18 * * 1-5` | Weekdays at 18:00 |

Poll SCM checks for source changes. It does not mean Jenkins must build when no source changes were detected.

## 7. Parameterized Pipeline: find the larger of two numbers

Create `find_largest.py` in your GitHub repository:

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

The lecture asks to set execute permission on both files. The Python file may be marked executable, but Jenkins reads `Jenkinsfile` as pipeline text; it does not need an executable bit. The pipeline runs Python using `python3`, so the Python executable bit is not technically required either.

```bash
chmod u+x find_largest.py
```

Commit and push:

```bash
git add find_largest.py Jenkinsfile
git commit -m "Add parameterized Jenkins pipeline"
git push origin main
```

Create a Jenkins Pipeline job:

1. **New Item** → choose **Pipeline**.
2. **Definition** = **Pipeline script from SCM**.
3. **SCM** = **Git**.
4. Enter your repository `.git` URL.
5. Branch Specifier = `*/main`.
6. Script Path = `Jenkinsfile`.
7. Save and click **Build with Parameters** (or Build Now, depending on the Jenkins UI).
8. Enter `NUM1` and `NUM2`, then run the build.

The Jenkins parameters become environment variables for the shell step. The Python script reads them with `os.environ.get`.

## 8. Common Pipeline checks

```bash
python3 --version
git --version
```

In Jenkins, check **Console Output** first. For Git checkout errors, confirm the repository URL, branch name, and credentials. For script errors, verify the filename and interpreter on the Jenkins agent.
