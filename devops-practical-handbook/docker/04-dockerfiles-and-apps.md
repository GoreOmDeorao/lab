# Docker: Dockerfiles and Application Images

Source coverage: Docker slides 58-74; Assignment 3, Q2-Q3, Q5, Q7; parts of Assignment 4 and Assignment 5.

## 1. Python Hello World

Create a project directory:

```bash
mkdir -p ~/docker-python-hello
cd ~/docker-python-hello
```

Create `hello.py`:

```python
print("Hello World")
```

Create a file named `Dockerfile` (no extension):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY hello.py .
CMD ["python", "hello.py"]
```

What the Dockerfile instructions do:

- `FROM` selects the base image and runtime.
- `WORKDIR` sets the working directory for following build/run instructions.
- `COPY` copies a host project file into the image.
- `CMD` defines the default process when a container starts.

Build and run:

```bash
docker build -t my-hello-python:1.0 . # Build image from Dockerfile in current directory
docker run --rm my-hello-python:1.0  # Run and automatically remove the stopped container
```

## 2. Java Hello World

```bash
mkdir -p ~/docker-java-hello
cd ~/docker-java-hello
```

Create `HelloWorld.java`:

```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

Create `Dockerfile`:

```dockerfile
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY HelloWorld.java .
RUN javac HelloWorld.java
CMD ["java", "HelloWorld"]
```

Build and run:

```bash
docker build -t cdac-java:1.0 .
docker run --rm cdac-java:1.0
```

The image uses a JDK because `javac` is needed during the build. To publish it to Docker Hub:

```bash
DOCKERHUB_USER="your-dockerhub-username"
docker login -u "$DOCKERHUB_USER"
docker tag cdac-java:1.0 "$DOCKERHUB_USER/cdac-java:1.0"
docker push "$DOCKERHUB_USER/cdac-java:1.0"
```

## 3. Python program that reads keyboard input

Create project:

```bash
mkdir -p ~/docker-python-add
cd ~/docker-python-add
```

Create `add.py`:

```python
num1 = float(input("Enter the first number: "))
num2 = float(input("Enter the second number: "))
print(f"The sum is: {num1 + num2}")
```

Create `Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY add.py .
CMD ["python", "add.py"]
```

Build and run interactively:

```bash
docker build -t python-sum-app:1.0 .
docker run -it --rm python-sum-app:1.0
```

`-it` attaches an interactive terminal so `input()` can receive your keystrokes. Without it, the program may fail because it has no interactive standard input.

## 4. C program that reads keyboard input

```bash
mkdir -p ~/docker-c-example
cd ~/docker-c-example
```

Create `user_input.c`:

```c
#include <stdio.h>

int main(void) {
    char name[100];
    printf("Enter your name: ");
    if (scanf("%99s", name) != 1) {
        return 1;
    }
    printf("Hello, %s!\n", name);
    return 0;
}
```

Create `Dockerfile`:

```dockerfile
FROM gcc:latest
WORKDIR /app
COPY user_input.c .
RUN gcc -Wall -Wextra -o user_input user_input.c
CMD ["./user_input"]
```

Build and test:

```bash
docker build -t my-ditiss-c:1.0 .
docker run -it --rm my-ditiss-c:1.0
```

Publish and export the image:

```bash
DOCKERHUB_USER="your-dockerhub-username"
docker login -u "$DOCKERHUB_USER"
docker tag my-ditiss-c:1.0 "$DOCKERHUB_USER/my-ditiss-c:1.0"
docker push "$DOCKERHUB_USER/my-ditiss-c:1.0"
docker save my-ditiss-c:1.0 -o cprogback.tar
```

## 5. Shell script inside a container

```bash
mkdir -p ~/docker-shell-demo
cd ~/docker-shell-demo
```

Create `hello.sh`:

```bash
#!/usr/bin/env sh
printf 'Hello from a shell script in Docker\n'
printf 'Current date: '
date
```

Create `Dockerfile`:

```dockerfile
FROM alpine:3.20
WORKDIR /app
COPY hello.sh .
RUN chmod +x hello.sh
CMD ["./hello.sh"]
```

Build and run:

```bash
docker build -t shell-demo:1.0 .
docker run --rm shell-demo:1.0
```

## 6. Custom NGINX image with your own HTML page

```bash
mkdir -p ~/docker-nginx-page
cd ~/docker-nginx-page
```

Create `index.html`:

```html
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>My NGINX page</title></head>
<body><h1>Hello from my custom NGINX container</h1></body>
</html>
```

Create `Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Build and run:

```bash
docker build -t my-nginx-page:1.0 .
docker run -d --name my-nginx-page -p 8080:80 my-nginx-page:1.0
```

Open `http://localhost:8080`.

Update by rebuilding with a new tag (recommended): change the HTML, then:

```bash
docker build -t my-nginx-page:2.0 .
docker rm -f my-nginx-page
docker run -d --name my-nginx-page -p 8080:80 my-nginx-page:2.0
```

## 7. Copy a changed HTML file into a running Apache container

This is the specific `httpd` exercise from the slide deck.

```bash
mkdir -p ~/httpd-copy-demo
cd ~/httpd-copy-demo
cat > index.html <<'HTML'
<!doctype html>
<html><body><h1>Hello from CDAC!</h1></body></html>
HTML

docker run -dit --name web5 -p 7777:80 httpd
# docker cp copies a host file into a path inside a running container.
docker cp index.html web5:/usr/local/apache2/htdocs/index.html
```

Open `http://localhost:7777`. Edit `index.html`, then copy it again:

```bash
cat > index.html <<'HTML'
<!doctype html>
<html><body><h1>Hello from DITISS!</h1></body></html>
HTML

docker cp index.html web5:/usr/local/apache2/htdocs/index.html
```

Refresh the browser. The updated file is in the container's writable layer, so it is not a durable deployment method. A bind mount or rebuilt image is better for repeatable deployments.

```bash
docker rm -f web5
```

## 8. Tomcat test page (as in the lecture)

```bash
docker run -dit --rm --name tomcat-container -p 7777:8080 tomcat:8.0
docker exec -it tomcat-container /bin/bash
```

Inside the container, create a page under the examples web application:

```bash
cd webapps/examples
echo '<h1>Hello from DITISS</h1>' > cdac.html
exit
```

Open `http://localhost:7777/examples/cdac.html`. The source uses the old `tomcat:8.0` tag. If the registry no longer provides it, use a maintained Tomcat image and adjust the webapp path for that image version.

## 9. Dockerfile instructions used in the slides

| Instruction | What it does |
|---|---|
| `FROM` | Selects the base image. |
| `WORKDIR` | Sets the working directory inside the image. |
| `COPY` | Copies files from the build context into the image. |
| `RUN` | Executes a command while building the image. |
| `CMD` | Default command/arguments when a container starts; can be overridden. |
| `ENTRYPOINT` | Main executable for the container; `CMD` can provide default arguments. |
| `EXPOSE` | Documents a container port; it does not publish the port by itself. |
| `ENV` | Sets an environment variable in the image/container. |
| `VOLUME` | Declares a mount point for persistent/external data. |

## 10. `CMD` versus `ENTRYPOINT`

Create `Dockerfile`:

```dockerfile
FROM ubuntu:24.04
ENTRYPOINT ["echo"]
CMD ["Hello from CMD"]
```

Build and run:

```bash
docker build -t entrypoint-demo:1.0 .
docker run --rm entrypoint-demo:1.0
docker run --rm entrypoint-demo:1.0 "Hi CDAC"
```

With no extra argument, the default command is `echo Hello from CMD`. When you pass `Hi CDAC`, it replaces the default `CMD` arguments while keeping the `echo` entrypoint.
