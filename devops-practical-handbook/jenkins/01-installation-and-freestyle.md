# Jenkins: Ubuntu Installation and Freestyle Jobs

Source coverage: Jenkins slides 4-11; official Jenkins installation instructions checked for the current Java/repository commands.

## 1. Install Jenkins on Ubuntu

The current Jenkins Linux installation guide specifies Java 21 or later for current Jenkins releases. Install Java before Jenkins:

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

Add the Jenkins stable APT repository and install the package:

```bash
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
```

Enable and start Jenkins:

```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

Open `http://localhost:8080`.

Unlock Jenkins:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy that value into the browser, install **suggested plugins**, and create your own administrator account. Do not use the slide deck's shared `admin` / `admin` style example credentials. Use a unique strong password.

## 2. Useful Jenkins service commands

```bash
sudo systemctl status jenkins --no-pager
sudo systemctl restart jenkins
sudo systemctl stop jenkins
sudo systemctl start jenkins
sudo journalctl -u jenkins.service -n 100 --no-pager
```

If Jenkins cannot start, check its service log and Java version:

```bash
java -version
sudo journalctl -u jenkins.service -n 100 --no-pager
```

## 3. Create a Freestyle job: Hello Jenkins

In the Jenkins UI:

1. Click **New Item**.
2. Enter `hello_jenkins`.
3. Select **Freestyle project**, then click **OK**.
4. Under **Build Steps**, select **Add build step → Execute shell**.
5. Enter the command below and click **Save**.
6. Click **Build Now**.
7. Open the build number under **Build History**, then open **Console Output**.

Build command:

```bash
echo "Hello Jenkins!"
```

## 4. Freestyle job: variables and date

Open `hello_jenkins` → **Configure**. Replace the Execute shell contents with:

```bash
NAME="C-DAC"
echo "Hello, $NAME. Current date and time is $(date)."
```

Save, build, and read the Console Output. `$NAME` expands to the variable value; `$(date)` runs the `date` command and inserts its output.

## 5. Freestyle job: run a Python one-liner

Use this Execute shell command:

```bash
python3 -c "print('Hello, World from Python!')"
```

The `-c` option tells Python to execute the quoted code directly instead of reading a `.py` file.

## 6. Freestyle job: create and run Java source

Install a JDK on the Jenkins machine if it does not already have the Java compiler:

```bash
sudo apt install -y openjdk-21-jdk
javac -version
java -version
```

Use this Execute shell block:

```bash
cat <<'JAVA_SOURCE' > HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World from Java in Jenkins!");
    }
}
JAVA_SOURCE

javac HelloWorld.java
java HelloWorld
```

The here-document creates the source file in the Jenkins job workspace. `javac` compiles it; `java` runs the compiled class.

## 7. Run a Python script stored outside the Jenkins workspace

Create `/opt/scripts/hello.py` on the Jenkins host:

```bash
sudo mkdir -p /opt/scripts
sudo tee /opt/scripts/hello.py >/dev/null <<'PYTHON_SCRIPT'
print('Hello, World')
print('from Python in Jenkins!')
PYTHON_SCRIPT
sudo chmod 644 /opt/scripts/hello.py
```

In the Jenkins Freestyle job's Execute shell step, run:

```bash
python3 /opt/scripts/hello.py
```

The Jenkins service account must have read access to that path. `python3 /path/to/script.py` does not require the script itself to be executable.

## 8. Write build output to a file

Use an Execute shell build step:

```bash
NAME="C-DAC"
echo "Hello, $NAME. Current date and time is $(date)." > /tmp/info
```

The `>` operator redirects the command output to a file, replacing its previous contents. Inspect the file on the Jenkins host:

```bash
cat /tmp/info
```

The file is in `/tmp` on the Jenkins host, not necessarily in the job workspace.
