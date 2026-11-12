# Docker: Volumes, Bind Mounts, and Persistent Data

Source coverage: Docker slides 39-57 and 76; `Docker notes.txt`; Assignment 3, Q1, Q4, Q6, Q8; Assignment 4, Q3-Q4.

## 1. Demonstrate why container writable-layer data is temporary

Start MySQL without a volume:

```bash
docker run -d --name mysql-no-volume \
  -e MYSQL_ROOT_PASSWORD=lab-root-password \
  -p 3306:3306 mysql:8

# Wait for the database's first initialization.
sleep 25

docker exec -it mysql-no-volume mysql -uroot -plab-root-password
```

Inside MySQL:

```sql
CREATE DATABASE cdac;
USE cdac;
CREATE TABLE student (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    score INT
);
INSERT INTO student (name, score) VALUES ('Amit', 85);
SELECT * FROM student;
EXIT;
```

Remove the container and create a new one without a volume:

```bash
docker stop mysql-no-volume
docker rm mysql-no-volume

docker run -d --name mysql-no-volume \
  -e MYSQL_ROOT_PASSWORD=lab-root-password \
  -p 3306:3306 mysql:8
sleep 25
docker exec -it mysql-no-volume mysql -uroot -plab-root-password -e 'SHOW DATABASES;'
```

The `cdac` database is gone because it was stored in the removed container's writable layer.

## 2. Repeat MySQL lab with a named volume

```bash
docker volume create mysql-data

docker run -d --name mysql-with-volume \
  -e MYSQL_ROOT_PASSWORD=lab-root-password \
  -v mysql-data:/var/lib/mysql \
  -p 3306:3306 mysql:8
sleep 25
docker exec -it mysql-with-volume mysql -uroot -plab-root-password
```

Inside MySQL:

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

Remove only the container, then create a new container using the same volume:

```bash
docker stop mysql-with-volume
docker rm mysql-with-volume

docker run -d --name mysql-with-volume \
  -e MYSQL_ROOT_PASSWORD=lab-root-password \
  -v mysql-data:/var/lib/mysql \
  -p 3306:3306 mysql:8
sleep 25
docker exec -it mysql-with-volume mysql -uroot -plab-root-password \
  -e 'USE cdac; SELECT * FROM student;'
```

The database and row should still exist because the data is in `mysql-data`, not the container layer.

Inspect volume metadata:

```bash
docker volume ls
docker volume inspect mysql-data
```

Cleanup. The last command permanently deletes the database data:

```bash
docker rm -f mysql-with-volume
docker volume rm mysql-data  # DESTRUCTIVE: deletes the stored database files
```

## 3. Create and reuse a named volume (`data_vol`)

```bash
docker volume create data_vol
docker run -d --name volume-writer -v data_vol:/data alpine sleep infinity
docker exec volume-writer sh -c 'echo "This file survives container removal" > /data/keep.txt'
docker exec volume-writer cat /data/keep.txt

docker rm -f volume-writer

# A new, temporary container mounts the same volume and reads the old file.
docker run --rm -v data_vol:/data alpine cat /data/keep.txt
```

Expected output: `This file survives container removal`.

## 4. Volume versus bind mount

- **Named volume:** Docker manages the storage location. Typical for database or service data.
- **Bind mount:** You choose a host filesystem path. Useful while developing files on the host and immediately seeing them in a container.

Create a host website directory and bind-mount it into NGINX:

```bash
mkdir -p ~/docker-bind-demo
cat > ~/docker-bind-demo/index.html <<'HTML'
<!doctype html>
<html><body><h1>Hello from a bind mount</h1></body></html>
HTML

docker run -d --name bind-nginx -p 8080:80 \
  --mount type=bind,source="$HOME/docker-bind-demo",target=/usr/share/nginx/html,readonly \
  nginx:alpine
```

Open `http://localhost:8080`. Edit the host file:

```bash
cat > ~/docker-bind-demo/index.html <<'HTML'
<!doctype html>
<html><body><h1>Updated directly on the host</h1></body></html>
HTML
```

Refresh the browser. The page updates without rebuilding the image because the container reads the mounted host directory.

Inspect the file from inside the container:

```bash
docker exec bind-nginx cat /usr/share/nginx/html/index.html
```

Cleanup:

```bash
docker rm -f bind-nginx
```

The `readonly` flag protects the host directory from writes made by NGINX.

## 5. NGINX live update using the slide's `webpages` folder

```bash
mkdir -p ~/webpages
cd ~/webpages
printf '%s\n' '<h1>Hi from the NGINX container</h1>' > index.html
cd ~

docker run -d --name another-nginx -p 8080:80 \
  -v "$HOME/webpages:/usr/share/nginx/html:ro" nginx
```

Open `http://localhost:8080`. Change the page on the host:

```bash
printf '%s\n' '<h1>Updated from the host machine</h1>' > ~/webpages/index.html
```

Refresh the browser, then inspect the file in the container:

```bash
docker exec another-nginx cat /usr/share/nginx/html/index.html
docker rm -f another-nginx
```

## 6. Share one volume between two different containers

```bash
docker volume create shared-vol

docker run --rm --name volume-writer -v shared-vol:/shared alpine \
  sh -c 'echo "Written by container one" > /shared/message.txt'

docker run --rm -v shared-vol:/shared alpine cat /shared/message.txt
```

Both containers mount the same Docker-managed storage. The second container reads the file created by the first.

## 7. Important mount syntax

```bash
# Named volume: volume-name:container-path
docker run --rm -v mysql-data:/var/lib/mysql mysql:8

# Bind mount: host-path:container-path
docker run --rm -v "$HOME/site:/usr/share/nginx/html:ro" nginx:alpine

# Anonymous volume: only the container path; Docker creates an unnamed volume
docker run --rm -v /usr/share/nginx/html nginx:alpine
```

The left side is the storage source; the right side is the directory inside the container. Removing a container does not normally remove a named volume. Removing the volume does delete its data.

## 8. Volume administration commands

```bash
docker volume create <VOLUME_NAME>   # Create a named volume
docker volume ls                    # List Docker volumes
docker volume inspect <VOLUME_NAME> # Show its mountpoint and metadata
docker volume rm <VOLUME_NAME>      # Delete a volume that is not in use
docker volume prune                 # DESTRUCTIVE: remove all unused local volumes
```

A named volume remains after the container is stopped or removed. Removing the volume permanently deletes its stored data.
