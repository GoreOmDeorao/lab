# Assignment 3: Docker Volumes and Dockerfiles

This assignment is covered by the practical guides below. The commands here provide the compact task sequence; use the linked guide for the complete file contents and explanation where indicated.

## Q1. Docker volume persistence

```bash
docker volume create data_vol
docker run -dit --name volume-lab -v data_vol:/data alpine sleep infinity
docker exec volume-lab sh -c 'echo "Persistent data" > /data/file.txt'
docker exec volume-lab cat /data/file.txt
docker rm -f volume-lab
docker run --rm -v data_vol:/data alpine cat /data/file.txt
```

Expected output after the first container is removed: `Persistent data`. The volume exists separately from the container.

To clean up the volume after the task (this deletes its data):

```bash
docker volume rm data_vol
```

Full explanation: [Volumes and persistence](../docker/03-volumes-and-persistence.md#3-create-and-reuse-a-named-volume-data_vol).

## Q2. Dockerfile - Python application

Use the complete [Python Hello World image lab](../docker/04-dockerfiles-and-apps.md#1-python-hello-world). The essential flow is:

```bash
mkdir -p ~/assignment3-python
cd ~/assignment3-python
```

`app.py`:

```python
print("Hello from Python inside Docker")
```

`Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

```bash
docker build -t assignment3-python:1.0 .
docker run --rm assignment3-python:1.0
```

## Q3. Dockerfile - Java application

Use [Java Hello World](../docker/04-dockerfiles-and-apps.md#2-java-hello-world).

`HelloWorld.java`:

```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello from Java inside Docker");
    }
}
```

`Dockerfile`:

```dockerfile
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY HelloWorld.java .
RUN javac HelloWorld.java
CMD ["java", "HelloWorld"]
```

```bash
docker build -t assignment3-java:1.0 .
docker run --rm assignment3-java:1.0
```

## Q4. Docker volume with MySQL

Follow [the MySQL named-volume persistence lab](../docker/03-volumes-and-persistence.md#2-repeat-mysql-lab-with-a-named-volume). It creates a database/table/row, removes the container, starts a new container with the same volume, and verifies that the row remains.

Core commands:

```bash
docker volume create mysql-data
docker run -d --name mysql-lab -e MYSQL_ROOT_PASSWORD=lab-root-password -v mysql-data:/var/lib/mysql mysql:8
sleep 25
docker exec -it mysql-lab mysql -uroot -plab-root-password
```

Inside MySQL, create the database and table, insert a row, and query it. Then:

```bash
docker rm -f mysql-lab
docker run -d --name mysql-lab -e MYSQL_ROOT_PASSWORD=lab-root-password -v mysql-data:/var/lib/mysql mysql:8
sleep 25
docker exec -it mysql-lab mysql -uroot -plab-root-password -e 'USE cdac; SELECT * FROM student;'
```

## Q5. Dockerfile - NGINX web server

Use [custom NGINX image with your HTML page](../docker/04-dockerfiles-and-apps.md#6-custom-nginx-image-with-your-own-html-page).

Key `Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

```bash
docker build -t assignment3-nginx:1.0 .
docker run -d --name assignment3-nginx -p 8080:80 assignment3-nginx:1.0
```

Open `http://localhost:8080`.

## Q6. Bind mount versus named volume

Full hands-on comparison: [Bind mount and named volume lab](../docker/03-volumes-and-persistence.md#4-volume-versus-bind-mount).

- Bind mount: host path that you choose, e.g. `$HOME/site:/usr/share/nginx/html`.
- Named volume: Docker-managed storage, e.g. `site-data:/usr/share/nginx/html`.

The assignment's practical bind-mount command:

```bash
mkdir -p ~/host-data
echo 'Hello from host' > ~/host-data/message.txt
docker run --rm -it --mount type=bind,source="$HOME/host-data",target=/data alpine cat /data/message.txt
```

## Q7. Dockerfile - shell script execution

Use [Shell script inside a container](../docker/04-dockerfiles-and-apps.md#5-shell-script-inside-a-container).

`hello.sh`:

```sh
#!/usr/bin/env sh
printf 'Hello from shell script\n'
date
```

`Dockerfile`:

```dockerfile
FROM alpine:3.20
WORKDIR /app
COPY hello.sh .
RUN chmod +x hello.sh
CMD ["./hello.sh"]
```

```bash
docker build -t assignment3-shell:1.0 .
docker run --rm assignment3-shell:1.0
```

## Q8. Shared volume between containers

Full example: [Share one volume between two containers](../docker/03-volumes-and-persistence.md#6-share-one-volume-between-two-different-containers).

```bash
docker volume create shared-vol
docker run --rm -v shared-vol:/shared alpine sh -c 'echo "shared by both" > /shared/file.txt'
docker run --rm -v shared-vol:/shared alpine cat /shared/file.txt
```

Expected output: `shared by both`.
