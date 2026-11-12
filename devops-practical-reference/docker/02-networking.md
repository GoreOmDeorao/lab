# Docker — Networking

Commands and application example from Docker slides 26–38 and `Docker notes.txt`.

## 1. Inspect the default bridge network

```bash
docker run -dit --name nginx-net-test nginx
docker inspect nginx-net-test
docker inspect -f '{{.NetworkSettings.IPAddress}}' nginx-net-test
docker network ls
docker network inspect bridge
```

## 2. Publish a container port

```bash
docker run -d --name nginx-web -p 8080:80 nginx
```

Open `http://localhost:8080`.

Stop and remove:

```bash
docker rm -f nginx-web
```

## 3. Create a custom bridge network

```bash
docker network create custom
docker network ls
docker network inspect custom
```

Run two BusyBox containers in the same network:

```bash
docker run -dit --name busybox1 --network custom busybox
docker run -dit --name busybox2 --network custom busybox
```

Test container name/DNS communication:

```bash
docker exec -it busybox1 sh
hostname
hostname -i
ping -c 3 busybox2
exit
```

Check the second container as well:

```bash
docker exec -it busybox2 sh
ping -c 3 busybox1
exit
```

Cleanup:

```bash
docker rm -f busybox1 busybox2
docker network rm custom
```

## 4. Python app connecting to MySQL by container name

Create a project directory and `app.py`:

```bash
mkdir -p ~/docker-networking
cd ~/docker-networking
nano app.py
```

`app.py`:

```python
import mysql.connector

conn = mysql.connector.connect(
    host="mysql-db",
    user="root",
    password="root",
    database="school"
)

cursor = conn.cursor()
cursor.execute(
    "CREATE TABLE IF NOT EXISTS students (name VARCHAR(50), age INT)"
)
cursor.execute("INSERT INTO students VALUES ('Alice', 20)")
conn.commit()
print("Inserted!")
cursor.close()
conn.close()
```

`Dockerfile`:

```dockerfile
FROM python:3.9
WORKDIR /app
COPY app.py .
RUN pip install mysql-connector-python
CMD ["python", "app.py"]
```

Build the image and create a network:

```bash
docker build -t student-app .
docker network create mynet
```

Start MySQL and then run the application container on the same network:

```bash
docker run -d --network mynet --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=school \
  mysql:8

# Wait for MySQL to finish initialization before running the app.
docker run --rm --network mynet --name app student-app
```

Verify the inserted row:

```bash
docker exec -it mysql-db mysql -uroot -proot
```

At the MySQL prompt:

```sql
USE school;
SELECT * FROM students;
EXIT;
```

Cleanup:

```bash
docker rm -f mysql-db
docker network rm mynet
```

> The source slide initially uses `host="localhost"`, which is wrong from inside the application container. On a user-defined bridge network, connect to the MySQL container by its name (`mysql-db`).

## 5. Tomcat and MySQL on one custom network

```bash
docker network create my-network

sudo mkdir -p /data/mysql_data /data/mytomcat_data

 docker container run --name mysql-container \
  -v /data/mysql_data:/var/lib/mysql \
  --network my-network \
  -e MYSQL_ROOT_PASSWORD=pass \
  -d mysql:latest

docker container run --name tomcat-container \
  --network my-network \
  -p 7777:8080 \
  -v /data/mytomcat_data:/usr/local/tomcat/webapps \
  -d tomcat:8
```

Open a shell in Tomcat and install a MySQL client as in the source lab:

```bash
docker exec -it tomcat-container /bin/bash
apt-get update
apt-get install -y default-mysql-client
mysql -h mysql-container -uroot -ppass
```

At the MySQL prompt:

```sql
CREATE DATABASE ditiss;
USE ditiss;
CREATE TABLE student_tbl (prn INT, name CHAR(30));
INSERT INTO student_tbl VALUES (1, 'My name');
SELECT * FROM student_tbl;
EXIT;
```

Source reference: Docker slides 26–38 and `Docker notes.txt` custom bridge network section.
