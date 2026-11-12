# Docker — Dockerfiles, Images, and Registry

Examples from Docker slides 58–74 and `Docker notes.txt`.

## 1. Python Hello World image

```bash
mkdir -p ~/docker_python_hello
cd ~/docker_python_hello
```

Create `hello.py`:

```python
print("Hello World")
```

Create `Dockerfile`:

```dockerfile
FROM python:3.9-slim-buster
WORKDIR /app
COPY hello.py /app
CMD ["python", "hello.py"]
```

Build and run:

```bash
docker build -t my-hello-python .
docker run --rm my-hello-python
```

## 2. Push an image to Docker Hub

```bash
docker login

# Tag a locally built image for your Docker Hub account
docker tag my-hello-python <YOUR_DOCKERHUB_USERNAME>/my-hello-python:1.0

docker push <YOUR_DOCKERHUB_USERNAME>/my-hello-python:1.0
```

Alternatively, build directly with the registry name:

```bash
docker build -t <YOUR_DOCKERHUB_USERNAME>/my-hello-python:1.0 .
docker push <YOUR_DOCKERHUB_USERNAME>/my-hello-python:1.0
```

## 3. Java Hello World image

```bash
mkdir -p ~/docker_java_hello
cd ~/docker_java_hello
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
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY HelloWorld.java /app
RUN javac HelloWorld.java
CMD ["java", "HelloWorld"]
```

Build and run:

```bash
docker build -t <YOUR_DOCKERHUB_USERNAME>/cdac-java:1.0 .
docker run --rm <YOUR_DOCKERHUB_USERNAME>/cdac-java:1.0
```

Push:

```bash
docker login
docker push <YOUR_DOCKERHUB_USERNAME>/cdac-java:1.0
```

## 4. Python program that accepts interactive input

```bash
mkdir -p ~/docker_python_add
cd ~/docker_python_add
```

Create `add.py`:

```python
num1 = float(input("Enter the first number: "))
num2 = float(input("Enter the second number: "))

print(f"The sum is: {num1 + num2}")
```

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /myapplication
COPY add.py /myapplication/add.py
CMD ["python", "add.py"]
```

Build and run interactively:

```bash
docker build -t python-sum-app .
docker run -it --rm python-sum-app
```

`-it` is required because the program waits for keyboard input.

## 5. C program that accepts interactive input

```bash
mkdir -p ~/docker_c_example
cd ~/docker_c_example
```

Create `user_input.c`:

```c
#include <stdio.h>

int main(void) {
    char name[100];
    printf("Enter your name: ");
    scanf("%99s", name);
    printf("Hello, %s!\n", name);
    return 0;
}
```

Create `Dockerfile`:

```dockerfile
FROM gcc:latest
WORKDIR /cdac/ditiss/mycode
COPY user_input.c .
RUN gcc -o user_input user_input.c
CMD ["./user_input"]
```

Build and run:

```bash
docker build -t my-ditiss-c .
docker run -it --rm my-ditiss-c
```

Push and save as a tar archive:

```bash
docker login
docker tag my-ditiss-c <YOUR_DOCKERHUB_USERNAME>/my-ditiss-c:1.0
docker push <YOUR_DOCKERHUB_USERNAME>/my-ditiss-c:1.0
docker save my-ditiss-c -o cprogback.tar
```

## 6. CMD versus ENTRYPOINT example

Create a `Dockerfile`:

```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["Hello from CMD"]
```

Build and run:

```bash
docker build -t entrypoint-demo .
docker run --rm entrypoint-demo
docker run --rm entrypoint-demo "Hi CDAC"
```

## 7. Copy a file into a running Apache container

Start the container:

```bash
docker run -dit --name web5 -p 7777:80 httpd
```

Create `index.html` on the host:

```html
<html>
<body>
    <h1>Hello from CDAC!</h1>
</body>
</html>
```

Copy it into the container and open `http://localhost:7777`:

```bash
docker cp index.html web5:/usr/local/apache2/htdocs/index.html
```

Edit the heading, then copy it again:

```bash
docker cp index.html web5:/usr/local/apache2/htdocs/index.html
```

Cleanup:

```bash
docker rm -f web5
```

## 8. Tomcat container and a test page

```bash
docker run -dit --rm --name tomcat-container -p 7777:8080 tomcat:8.0
docker exec -it tomcat-container /bin/bash
```

Inside the container:

```bash
cd webapps/examples
echo 'Hello from DITISS' > cdac.html
exit
```

Open `http://localhost:7777/examples/cdac.html`.

## 9. Save and load Docker images

```bash
docker pull ubuntu
docker save ubuntu -o ubuntu.tar

# Transfer ubuntu.tar to the other Linux machine, then run:
docker load -i ubuntu.tar
docker images
```

## 10. Trivy image scan

The source slide runs Trivy as a Docker container and mounts the local Docker socket:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image <IMAGE_NAME>:<TAG>
```

Source reference: Docker slides 58–74 and 103; Docker Hub tagging commands from `Docker notes.txt`.
