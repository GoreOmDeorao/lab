# Docker Compose: Multi-Container Applications

Source coverage: Docker slides 78-95; Assignment 4, Q1-Q4.

## 1. Check Docker Compose

```bash
docker compose version # The current plugin-style Compose command
```

All examples below use `docker compose` (space), not the older standalone `docker-compose` command.

## 2. NGINX static website with a bind mount (Assignment 4, Q1)

Create project and page:

```bash
mkdir -p ~/compose-nginx/website
cd ~/compose-nginx
cat > website/index.html <<'HTML'
<!doctype html>
<html><body><h1>Hello from Docker Compose</h1></body></html>
HTML
```

Create `compose.yaml`:

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./website:/usr/share/nginx/html:ro
```

Run and verify:

```bash
docker compose up -d
docker compose ps
docker compose logs nginx
```

Open `http://localhost:8080`. The host `website/` folder is mounted read-only into NGINX's document root. Edit `website/index.html` and refresh to see the change.

Stop and remove the Compose container/network:

```bash
docker compose down
```

## 3. Run a one-shot Python script with Compose (Assignment 4, Q2)

```bash
mkdir -p ~/compose-python
cd ~/compose-python
```

Create `app.py`:

```python
print("Hello from Python Container")
```

Create `Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

Create `compose.yaml`:

```yaml
services:
  python-job:
    build: .
```

Run it in the foreground:

```bash
docker compose up --build
```

The script prints once and exits. Compose may then report that the service stopped; this is expected for a one-shot command. Clean up:

```bash
docker compose down
```

## 4. MySQL with persistent named volume (Assignment 4, Q3)

Create project:

```bash
mkdir -p ~/compose-mysql
cd ~/compose-mysql
```

Create `.env` with a local lab password. Do not commit this file:

```bash
cat > .env <<'ENVVARS'
MYSQL_ROOT_PASSWORD=replace-with-a-local-lab-password
MYSQL_DATABASE=college
ENVVARS
```

Create `.gitignore`:

```text
.env
```

Create `compose.yaml`:

```yaml
services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?Set MYSQL_ROOT_PASSWORD in .env}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-college}
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD --silent"]
      interval: 5s
      timeout: 3s
      retries: 20

volumes:
  mysql-data:
```

Start and verify:

```bash
docker compose up -d
docker compose ps
```

Connect to MySQL (type the password from `.env` when prompted):

```bash
docker compose exec db mysql -uroot -p
```

Run SQL:

```sql
USE college;
CREATE TABLE IF NOT EXISTS test (
    id INT PRIMARY KEY AUTO_INCREMENT,
    message VARCHAR(100)
);
INSERT INTO test (message) VALUES ('data survives container recreation');
SELECT * FROM test;
EXIT;
```

Recreate the container while retaining the named volume:

```bash
docker compose down
docker compose up -d
docker compose exec db mysql -uroot -p -D college -e 'SELECT * FROM test;'
```

The data remains because normal `docker compose down` does not remove the `mysql-data` named volume.

Delete the database data only when intentionally cleaning up:

```bash
docker compose down -v  # DESTRUCTIVE: -v also removes the named mysql-data volume
```

## 5. Three NGINX containers, one shared volume, three different pages (Assignment 4, Q4)

### Important filesystem detail

Mounting one volume at `/usr/share/nginx/html` in all three containers means that the containers see the same files at that path. If all use the default NGINX document root, they will display the same page. To meet both parts of the assignment, this example stores three pages in separate subdirectories inside one shared volume and gives each NGINX container its own document-root configuration.

Create project and config files:

```bash
mkdir -p ~/compose-three-nginx/conf
cd ~/compose-three-nginx
```

Create `conf/nginx1.conf`:

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html/nginx1;
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
```

Create `conf/nginx2.conf` (same settings, but `root /usr/share/nginx/html/nginx2;`):

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html/nginx2;
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
```

Create `conf/nginx3.conf` (same settings, but `root /usr/share/nginx/html/nginx3;`):

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html/nginx3;
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
```

Create `compose.yaml`:

```yaml
services:
  init-shared-data:
    image: busybox:1.36
    command:
      - sh
      - -c
      - |
        mkdir -p /shared/nginx1 /shared/nginx2 /shared/nginx3 &&
        printf '%s\n' '<h1>Hello from Container 1</h1>' > /shared/nginx1/index.html &&
        printf '%s\n' '<h1>Hello from Container 2</h1>' > /shared/nginx2/index.html &&
        printf '%s\n' '<h1>Hello from Container 3</h1>' > /shared/nginx3/index.html
    volumes:
      - shared-data:/shared

  nginx1:
    image: nginx:alpine
    depends_on:
      init-shared-data:
        condition: service_completed_successfully
    ports:
      - "8081:80"
    volumes:
      - shared-data:/usr/share/nginx/html:ro
      - ./conf/nginx1.conf:/etc/nginx/conf.d/default.conf:ro

  nginx2:
    image: nginx:alpine
    depends_on:
      init-shared-data:
        condition: service_completed_successfully
    ports:
      - "8082:80"
    volumes:
      - shared-data:/usr/share/nginx/html:ro
      - ./conf/nginx2.conf:/etc/nginx/conf.d/default.conf:ro

  nginx3:
    image: nginx:alpine
    depends_on:
      init-shared-data:
        condition: service_completed_successfully
    ports:
      - "8083:80"
    volumes:
      - shared-data:/usr/share/nginx/html:ro
      - ./conf/nginx3.conf:/etc/nginx/conf.d/default.conf:ro

volumes:
  shared-data:
```

The one-shot `init-shared-data` service creates the folders and their pages in the volume. Each NGINX container mounts that same named volume at the assignment's target path, but uses a different subdirectory as its `root`.

Start and verify all three:

```bash
docker compose up -d
docker compose ps
curl http://localhost:8081
curl http://localhost:8082
curl http://localhost:8083
```

Open each URL in a browser. Expected headings:

- `http://localhost:8081` - Hello from Container 1
- `http://localhost:8082` - Hello from Container 2
- `http://localhost:8083` - Hello from Container 3

Cleanup containers and network but retain the named volume:

```bash
docker compose down
```

Remove stored pages only when you intentionally want to reset the lab:

```bash
docker compose down -v  # DESTRUCTIVE: deletes shared-data
```

## 6. Python + MySQL Compose app from the lecture slides

Project structure:

```text
student-app/
├── app.py
├── requirements.txt
├── Dockerfile
└── compose.yaml
```

Create `app.py`:

```python
from flask import Flask
import mysql.connector

app = Flask(__name__)


def get_db():
    return mysql.connector.connect(
        host="db",                 # Compose service name, not localhost
        user="root",
        password="root123",       # Local classroom example only
        database="college",
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
        [(1, "Amit", 85), (2, "Neha", 90), (3, "Raj", 78)],
    )
    conn.commit()
    cur.execute("SELECT * FROM student ORDER BY id")
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

`requirements.txt`:

```text
flask
mysql-connector-python
```

`Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

`compose.yaml` (classroom password for local-only use; do not expose this database to an untrusted network):

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
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -proot123 --silent"]
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

Run and verify:

```bash
docker compose up --build -d
docker compose ps
docker compose logs app
```

Open `http://localhost:5000`.

```bash
docker compose down
# Keep mysql-data for next run. Use `docker compose down -v` only to delete it.
```

## 7. Java Hello World with Compose (Docker slide 94-95)

Create project:

```bash
mkdir -p ~/java-docker-compose
cd ~/java-docker-compose
```

`HelloWorld.java`:

```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

`Dockerfile`:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /my-compose-app
COPY HelloWorld.java .
RUN javac HelloWorld.java
CMD ["java", "HelloWorld"]
```

`compose.yaml`:

```yaml
services:
  java-helloworld:
    build: .
    container_name: java-helloworld-container
```

Run and remove:

```bash
docker compose up --build
docker compose down
```

This is a one-shot container: it prints the Java output and exits.

## 8. Compose YAML keys used in the slides

```yaml
services:
  web:
    image: nginx:alpine
    container_name: my-nginx-container  # Optional fixed container name
    ports:
      - "8080:80"                       # Host port 8080 -> container port 80
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: local-lab-only
      MYSQL_DATABASE: cdac
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

`container_name` is optional. Avoid setting it for services where you want Compose to scale multiple replicas of the same service; fixed names prevent Compose from creating multiple containers with the same name.

## 9. Compose commands quick reference

```bash
docker compose config             # Validate and render merged Compose configuration
docker compose build              # Build services that have a build: section
docker compose up                 # Start services in foreground
docker compose up -d               # Start services in background
docker compose ps                  # Show Compose containers and state
docker compose logs -f <service>   # Follow logs from one service
docker compose down                # Stop and remove containers/network
docker compose down -v              # Also delete named volumes: destructive
```

The slides also point to `https://github.com/newdelthis/demo/blob/main/docker-compose.yml` for an NGINX + phpMyAdmin example. The YAML is not included in the presentation, so it is not recreated here. The slide's browse ports are 8080 (NGINX) and 8081 (phpMyAdmin), and its sample credentials are `root` / `root123`; treat those as lab-only, not secure credentials.
