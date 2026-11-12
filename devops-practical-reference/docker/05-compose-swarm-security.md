# Docker — Compose, Swarm, and Trivy

Practical examples from Docker slides 78–103.

## 1. Compose: Flask + MySQL application

Create project structure:

```bash
mkdir -p ~/student-app
cd ~/student-app
```

Create `app.py`:

```python
from flask import Flask
import mysql.connector

app = Flask(__name__)


def get_db():
    return mysql.connector.connect(
        host="db",
        user="root",
        password="root123",
        database="college"
    )


@app.route("/")
def students():
    conn = get_db()
    cur = conn.cursor()

    cur.execute("""
        CREATE TABLE IF NOT EXISTS student (
            id INT PRIMARY KEY,
            name VARCHAR(50),
            marks INT
        )
    """)

    cur.executemany(
        "INSERT IGNORE INTO student VALUES (%s, %s, %s)",
        [
            (1, "Amit", 85),
            (2, "Neha", 90),
            (3, "Raj", 78)
        ]
    )
    conn.commit()

    cur.execute("SELECT * FROM student")
    rows = cur.fetchall()
    cur.close()
    conn.close()

    html = "<h2>Student Records</h2><table border='1'>"
    html += "<tr><th>ID</th><th>Name</th><th>Marks</th></tr>"
    for row in rows:
        html += f"<tr><td>{row[0]}</td><td>{row[1]}</td><td>{row[2]}</td></tr>"
    return html + "</table>"


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Create `requirements.txt`:

```text
flask
mysql-connector-python
```

Create `Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

Create `docker-compose.yml`:

```yaml
services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: college
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-proot123"]
      interval: 5s
      timeout: 3s
      retries: 20

  app:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  mysql-data:
```

Build and run:

```bash
docker compose up --build -d

docker compose ps
docker network ls
docker compose logs app
```

Open `http://localhost:5000`.

Stop and remove the stack:

```bash
docker compose down
```

Remove the named database volume only when you intentionally want to delete the stored database:

```bash
# DESTRUCTIVE: permanently removes the Compose database volume
docker volume rm student-app_mysql-data
```

## 2. Compose: Java Hello World

Create project:

```bash
mkdir -p ~/java_docker_compose
cd ~/java_docker_compose
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
WORKDIR /my-compose-app
COPY HelloWorld.java .
RUN javac HelloWorld.java
CMD ["java", "HelloWorld"]
```

Create `docker-compose.yml` (the compose service configuration was embedded as an image on slide 95):

```yaml
services:
  java-helloworld:
    build: .
    container_name: java-helloworld-container
    command: ["java", "HelloWorld"]
```

Run and remove:

```bash
docker compose up --build
docker compose down
```

## 3. Compose command reference

```bash
docker compose build
docker compose up
docker compose up -d
docker compose ps
docker compose logs <SERVICE_NAME>
docker compose down
```

The slide deck also links to a ready-made NGINX + phpMyAdmin Compose example at https://github.com/newdelthis/demo/blob/main/docker-compose.yml. The YAML body is not included in the slides. The stated commands are `docker compose up`, then browse `localhost:8080` for NGINX and `localhost:8081` for phpMyAdmin (the example credentials shown are root / root123), and finally `docker compose down`.

## 4. Docker Swarm: initialize and inspect

```bash
docker info | grep -i swarm
docker swarm init
```

If Docker reports multiple network interfaces and asks for an advertise address, use an IP assigned to the machine:

```bash
docker swarm init --advertise-addr <THIS_MACHINE_IP>
```

List nodes:

```bash
docker node ls
```

## 5. Docker Swarm: create and scale an NGINX service

```bash
docker service create --name nginx-service --replicas 1 -p 9999:80 \
  <YOUR_DOCKERHUB_USERNAME>/custom-nginx:1.0

docker service ls
docker service ps nginx-service

docker service scale nginx-service=10
docker service ps nginx-service
docker service scale nginx-service=2
docker service ps nginx-service
```

## 6. Build and publish a custom NGINX image for Swarm

Create project and page:

```bash
mkdir -p ~/nginx_project
cd ~/nginx_project
echo '<h1>Welcome to my custom NGINX server</h1>' > index.html
```

Create `Dockerfile`:

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
```

Build and test locally:

```bash
docker build -t <YOUR_DOCKERHUB_USERNAME>/custom-nginx:1.0 .
docker run -d --name nginx-test -p 9999:80 <YOUR_DOCKERHUB_USERNAME>/custom-nginx:1.0
```

Open `http://localhost:9999`, then publish:

```bash
docker login
docker push <YOUR_DOCKERHUB_USERNAME>/custom-nginx:1.0
docker rm -f nginx-test
```

Update the website and publish version 2:

```bash
echo '<h1>Welcome to the updated NGINX server</h1>' > index.html
docker build -t <YOUR_DOCKERHUB_USERNAME>/custom-nginx:2.0 .
docker push <YOUR_DOCKERHUB_USERNAME>/custom-nginx:2.0
```

Update the Swarm service:

```bash
docker service update --image <YOUR_DOCKERHUB_USERNAME>/custom-nginx:2.0 nginx-service
docker service ps nginx-service
```

Remove the service:

```bash
docker service rm nginx-service
```

If you created a single-node lab Swarm and no longer need it:

```bash
docker swarm leave --force
```

## 7. Run a one-shot Alpine command

```bash
docker run --rm alpine sh -c "echo 'hello world'"
```

Source reference: Docker slides 78–103. The Java Compose file on slide 95 was read from the embedded image rather than parsed slide text.
