# Docker: Networking and Container-to-Container Communication

Source coverage: Docker slides 26-38 and 75-77; `Docker notes.txt`; Assignment 2, Q7.

## 1. Inspect container networking and publish a web port

```bash
docker run -d --name nginx-net -p 8080:80 nginx:latest
# Host port 8080 forwards to port 80 inside the container.
docker ps
docker inspect nginx-net
docker inspect -f '{{.NetworkSettings.IPAddress}}' nginx-net
```

Open `http://localhost:8080`. A container's private IP is internal to Docker networking; publishing a port is what makes the service reachable from the host or other permitted networks.

```bash
docker network ls                 # List available Docker networks
docker network inspect bridge    # Inspect Docker's default bridge network
```

## 2. Default bridge versus a user-defined bridge

The default `bridge` network is created automatically. User-defined bridge networks provide Docker DNS name resolution between attached containers and better isolation between separate networks.

Start two BusyBox containers on the default bridge:

```bash
docker run -dit --name busy-default-1 busybox sh
docker run -dit --name busy-default-2 busybox sh
```

Inspect both IP addresses:

```bash
docker exec busy-default-1 hostname
docker exec busy-default-1 hostname -i
docker inspect -f '{{.NetworkSettings.IPAddress}}' busy-default-2
```

Get the second container's IP into a shell variable and test IP-based communication:

```bash
TARGET_IP=$(docker inspect -f '{{.NetworkSettings.IPAddress}}' busy-default-2)
docker exec busy-default-1 ping -c 3 "$TARGET_IP"
```

The default bridge does not provide the same automatic container-name DNS behavior as a user-defined bridge. Container IPs can also change when containers are recreated. After the test, create a user-defined network:

```bash
docker network create custom
docker network ls
docker network inspect custom
```

Run containers attached to that network:

```bash
docker run -dit --name busybox1 --network custom busybox
docker run -dit --name busybox2 --network custom busybox
```

Test name-based communication:

```bash
docker exec busybox1 ping -c 3 busybox2
docker exec busybox2 ping -c 3 busybox1
```

Because both containers are attached to `custom`, Docker's embedded DNS resolves the container names. Test by IP if required:

```bash
docker inspect -f '{{.NetworkSettings.Networks.custom.IPAddress}}' busybox1
docker inspect -f '{{.NetworkSettings.Networks.custom.IPAddress}}' busybox2
```

Cleanup:

```bash
docker rm -f busy-default-1 busy-default-2 busybox1 busybox2
docker network rm custom
```

## 3. Python application connecting to MySQL in another container

**Important correction from the slide:** `host="localhost"` inside the Python container points to the Python container itself, not the MySQL container. Put both containers on the same user-defined network and connect to the database by its container name (`mysql-db`).

Create a project:

```bash
mkdir -p ~/docker-networking-demo
cd ~/docker-networking-demo
```

Create `app.py`:

```python
import mysql.connector

conn = mysql.connector.connect(
    host="mysql-db",       # Docker DNS name of the MySQL container
    user="root",
    password="root",
    database="school",
)

cursor = conn.cursor()
cursor.execute(
    "CREATE TABLE IF NOT EXISTS students (name VARCHAR(50), age INT)"
)
cursor.execute("INSERT INTO students (name, age) VALUES ('Alice', 20)")
conn.commit()
print("Inserted one student row.")

cursor.execute("SELECT name, age FROM students")
for row in cursor.fetchall():
    print(row)

cursor.close()
conn.close()
```

Create `Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir mysql-connector-python
CMD ["python", "app.py"]
```

Create network, build image, start MySQL, wait for it to initialize, and run the client app:

```bash
docker network create mynet
docker build -t student-app:1.0 .

docker run -d \
  --name mysql-db \
  --network mynet \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=school \
  mysql:8

# Give the database time to initialize on the first run.
sleep 25

docker run --rm --network mynet --name student-app-run student-app:1.0
```

Verify the row directly in MySQL:

```bash
docker exec -it mysql-db mysql -uroot -proot -D school \
  -e 'SELECT name, age FROM students;'
```

Cleanup:

```bash
docker rm -f mysql-db
docker network rm mynet
```

This slide example does not use a persistent volume. Removing `mysql-db` removes its writable container layer and the database data. See [Docker volumes](03-volumes-and-persistence.md) for persistence.

## 4. Tomcat and MySQL on one Docker network

Create one user-defined network so Tomcat can resolve MySQL by its container name:

```bash
docker network create my-network
mkdir -p /data/mysql_data /data/mytomcat_data

docker run -d \
  --name mysql-container \
  --network my-network \
  -v /data/mysql_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=pass \
  -e MYSQL_DATABASE=ditiss \
  mysql:8

docker run -d \
  --name tomcat-container \
  --network my-network \
  -p 7777:8080 \
  -v /data/mytomcat_data:/usr/local/tomcat/webapps \
  tomcat:8
```

The source uses Tomcat 8. It is an old image line; use it only where it is still available and compatible with the lab. For a fresh production setup, select a maintained Tomcat version deliberately.

Connect from the Tomcat container (the package command depends on the Tomcat base-image distribution):

```bash
docker exec -it tomcat-container bash
apt-get update
apt-get install -y default-mysql-client
mysql -h mysql-container -uroot -ppass
```

Inside the MySQL client:

```sql
CREATE DATABASE IF NOT EXISTS ditiss;
USE ditiss;
CREATE TABLE IF NOT EXISTS student_tbl (
    prn INT,
    name VARCHAR(30)
);
INSERT INTO student_tbl (prn, name) VALUES (1, 'My name');
SELECT * FROM student_tbl;
EXIT;
```

Cleanup:

```bash
docker rm -f tomcat-container mysql-container
docker network rm my-network
```

The host folders under `/data` keep their files after the containers are removed. Delete them only if you intentionally want to remove the lab data.
