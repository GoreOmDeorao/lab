# Docker — Volumes and Persistent Data

Practical examples from Docker slides 39–57 and 76, plus the MySQL example in `Docker notes.txt`.

## 1. MySQL without a volume (data is lost with the container)

Start MySQL:

```bash
docker run -d --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -p 3306:3306 \
  mysql:8
```

Connect to MySQL:

```bash
docker exec -it my-mysql mysql -uroot -proot123
```

Run SQL:

```sql
CREATE DATABASE cdac;
USE cdac;
CREATE TABLE IF NOT EXISTS student (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    score INT
);
INSERT INTO student (name, score) VALUES ('Amit', 85);
SELECT * FROM student;
EXIT;
```

Remove the container:

```bash
docker stop my-mysql
docker rm my-mysql
```

Recreate it without a volume and check that the old database is absent:

```bash
docker run -d --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -p 3306:3306 \
  mysql:8

docker exec -it my-mysql mysql -uroot -proot123
```

```sql
SHOW DATABASES;
EXIT;
```

## 2. MySQL with a named volume (data survives container removal)

Remove the non-persistent container if it still exists, then start MySQL with a named volume:

```bash
docker rm -f my-mysql 2>/dev/null || true
docker volume create mysql-data

docker run -d --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -v mysql-data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8
```

Connect and create data using the SQL from section 1:

```bash
docker exec -it my-mysql mysql -uroot -p
```

Remove only the container:

```bash
docker stop my-mysql
docker rm my-mysql
```

Recreate it using the **same** named volume:

```bash
docker run -d --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -v mysql-data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8

docker exec -it my-mysql mysql -uroot -p
```

```sql
USE cdac;
SELECT * FROM student;
EXIT;
```

## 3. Volume management commands

```bash
docker volume create <VOLUME_NAME>
docker volume ls
docker volume inspect <VOLUME_NAME>
docker volume rm <VOLUME_NAME>

# DESTRUCTIVE: removes unused volumes
docker volume prune
```

Common mount forms:

```bash
# Named volume
-v mysql-data:/var/lib/mysql

# Bind mount (replace with your actual Linux path)
-v /home/<YOUR_USER>/mysql-data:/var/lib/mysql

# Anonymous volume
-v /var/lib/mysql
```

## 4. NGINX serving a host directory

Create a webpage:

```bash
mkdir -p ~/webpages
cd ~/webpages
echo 'Hi from inside our CDAC nginx container!' > index.html
cd ~
```

Run NGINX with the host directory mounted into the NGINX document root:

```bash
docker run -d --name another_nginx \
  -p 8080:80 \
  -v "$HOME/webpages:/usr/share/nginx/html" \
  nginx
```

Open `http://localhost:8080`.

Change the page on the host:

```bash
echo 'Updated from host machine!' > ~/webpages/index.html
echo 'Second update!' > ~/webpages/index.html
```

Refresh the browser. Verify from inside the container:

```bash
docker exec -it another_nginx /bin/bash
cat /usr/share/nginx/html/index.html
exit
```

Cleanup:

```bash
docker rm -f another_nginx
```

## 5. MySQL + Tomcat with bind-mounted data directories

```bash
sudo mkdir -p /data/mysql_data /data/mytomcat_data
docker network create my-network

docker run -d --name mysql-container \
  --network my-network \
  -v /data/mysql_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=pass \
  mysql:latest

docker run -d --name tomcat-container \
  --network my-network \
  -p 7777:8080 \
  -v /data/mytomcat_data:/usr/local/tomcat/webapps \
  tomcat:8
```

Source reference: Docker slides 39–57 and 76; `Docker notes.txt` MySQL lifecycle example.
