# Docker — Pull to Run 🚀

A practical Docker reference for running common development services:

* PostgreSQL
* MySQL
* Redis
* Nginx
* RabbitMQ

---

# 1. Docker Basics

## Check Docker

```bash
docker --version
```

```bash
docker info
```

Check Docker service:

```bash
sudo systemctl status docker
```

Start Docker:

```bash
sudo systemctl start docker
```

Enable Docker on boot:

```bash
sudo systemctl enable docker
```

---

# 2. Docker Basic Concepts

```text
Docker Hub
    │
    │ docker pull
    ▼
  Image
    │
    │ docker run
    ▼
Container
    │
    ├── start
    ├── stop
    ├── restart
    ├── logs
    └── exec
```

### Image

Image is a template used to create containers.

### Container

Container is a running instance of an image.

```text
Image → docker run → Container
```

---

# 3. Image Commands

List images:

```bash
docker images
```

Pull image:

```bash
docker pull <image>:<tag>
```

Example:

```bash
docker pull postgres:18
```

Remove image:

```bash
docker rmi <image>:<tag>
```

Remove unused images:

```bash
docker image prune
```

---

# 4. Container Commands

List running containers:

```bash
docker ps
```

List all containers:

```bash
docker ps -a
```

Run a container:

```bash
docker run -d --name <container-name> <image>:<tag>
```

Stop:

```bash
docker stop <container-name>
```

Start:

```bash
docker start <container-name>
```

Restart:

```bash
docker restart <container-name>
```

Remove:

```bash
docker rm <container-name>
```

Force remove:

```bash
docker rm -f <container-name>
```

---

# 5. Container Logs

Show logs:

```bash
docker logs <container-name>
```

Follow logs:

```bash
docker logs -f <container-name>
```

Example:

```bash
docker logs -f postgres-db
```

---

# 6. Enter a Container

Using bash:

```bash
docker exec -it <container-name> bash
```

If bash is unavailable:

```bash
docker exec -it <container-name> sh
```

Exit:

```bash
exit
```

---

# 7. PostgreSQL 🐘

## Pull Image

```bash
docker pull postgres:18
```

Check:

```bash
docker images
```

## Run PostgreSQL

```bash
docker run -d \
  --name postgres-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=ecommerce \
  -p 5432:5432 \
  postgres:18
```

### Port

```text
Host:5432 → Container:5432
```

## Check

```bash
docker ps
```

## Logs

```bash
docker logs postgres-db
```

## Connect to PostgreSQL

```bash
docker exec -it postgres-db psql -U postgres -d ecommerce
```

Inside PostgreSQL:

List databases:

```sql
\l
```

List tables:

```sql
\dt
```

Exit:

```sql
\q
```

## NestJS / Prisma Connection

If NestJS runs on the host machine:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/ecommerce"
```

---

# 8. MySQL 🐬

## Pull Image

```bash
docker pull mysql:8
```

## Run MySQL

```bash
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=ecommerce \
  -p 3306:3306 \
  mysql:8
```

### Port

```text
Host:3306 → Container:3306
```

## Check

```bash
docker ps
```

## Logs

```bash
docker logs mysql-db
```

## Connect to MySQL

```bash
docker exec -it mysql-db mysql -u root -p
```

Password:

```text
root
```

Inside MySQL:

Show databases:

```sql
SHOW DATABASES;
```

Use database:

```sql
USE ecommerce;
```

Show tables:

```sql
SHOW TABLES;
```

Exit:

```sql
exit;
```

---

# 9. Redis 🔴

## Pull Image

```bash
docker pull redis:7
```

## Run Redis

```bash
docker run -d \
  --name redis-server \
  -p 6379:6379 \
  redis:7
```

### Port

```text
Host:6379 → Container:6379
```

## Check

```bash
docker ps
```

## Logs

```bash
docker logs redis-server
```

## Connect to Redis

```bash
docker exec -it redis-server redis-cli
```

Test:

```redis
PING
```

Expected:

```text
PONG
```

Set value:

```redis
SET name "Asik"
```

Get value:

```redis
GET name
```

Exit:

```redis
exit
```

---

# 10. Nginx 🌐

## Pull Image

```bash
docker pull nginx:latest
```

## Simple Run

```bash
docker run -d \
  --name nginx-server \
  -p 8080:80 \
  nginx:latest
```

### Port

```text
Host:8080 → Container:80
```

Open:

```text
http://localhost:8080
```

You should see the Nginx welcome page.

## Check

```bash
docker ps
```

## Logs

```bash
docker logs nginx-server
```

## Enter Container

```bash
docker exec -it nginx-server bash
```

Nginx configuration:

```bash
cd /etc/nginx
```

Default config:

```bash
cat /etc/nginx/conf.d/default.conf
```

Exit:

```bash
exit
```

---

# 11. Nginx Reverse Proxy → NestJS

If NestJS runs on the host machine at:

```text
localhost:3000
```

Create:

```text
nginx/
└── default.conf
```

### `nginx/default.conf`

```nginx
server {
    listen 80;

    server_name localhost;

    location / {
        proxy_pass http://host.docker.internal:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Remove old container:

```bash
docker rm -f nginx-server
```

Run Nginx with custom config:

```bash
docker run -d \
  --name nginx-server \
  --add-host=host.docker.internal:host-gateway \
  -p 8080:80 \
  -v "$(pwd)/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro" \
  nginx:latest
```

Test configuration:

```bash
docker exec nginx-server nginx -t
```

Reload configuration:

```bash
docker exec nginx-server nginx -s reload
```

Now:

```text
Client
   │
   ▼
localhost:8080
   │
   ▼
Nginx Container
   │
   ▼
host.docker.internal:3000
   │
   ▼
NestJS
```

---

# 12. RabbitMQ 🐇

Use the management image so that the RabbitMQ web dashboard is available.

## Pull Image

```bash
docker pull rabbitmq:4-management
```

## Run RabbitMQ

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:4-management
```

### Ports

```text
5672  → RabbitMQ application/AMQP
15672 → RabbitMQ Management UI
```

## Check

```bash
docker ps
```

## Logs

```bash
docker logs rabbitmq
```

## Management Dashboard

Open:

```text
http://localhost:15672
```

Default development credentials:

```text
Username: guest
Password: guest
```

## RabbitMQ Container

```bash
docker exec -it rabbitmq bash
```

Exit:

```bash
exit
```

---

# 13. All Services Together

After running all the above containers:

```bash
docker ps
```

Expected setup:

```text
CONTAINER        IMAGE
------------------------------------------------
nginx-server     nginx:latest
postgres-db      postgres:18
mysql-db         mysql:8
redis-server     redis:7
rabbitmq         rabbitmq:4-management
```

Ports:

```text
Nginx       → 8080
PostgreSQL  → 5432
MySQL       → 3306
Redis       → 6379
RabbitMQ    → 5672
RabbitMQ UI → 15672
```

---

# 14. Quick Status Check

```bash
docker ps
```

Detailed:

```bash
docker ps -a
```

---

# 15. Start Everything

```bash
docker start postgres-db
docker start mysql-db
docker start redis-server
docker start nginx-server
docker start rabbitmq
```

---

# 16. Stop Everything

```bash
docker stop postgres-db
docker stop mysql-db
docker stop redis-server
docker stop nginx-server
docker stop rabbitmq
```

---

# 17. Restart Everything

```bash
docker restart postgres-db
docker restart mysql-db
docker restart redis-server
docker restart nginx-server
docker restart rabbitmq
```

---

# 18. Remove Everything

⚠️ Removing containers does not necessarily remove Docker volumes.

```bash
docker rm -f postgres-db
docker rm -f mysql-db
docker rm -f redis-server
docker rm -f nginx-server
docker rm -f rabbitmq
```

---

# 19. Docker Volumes 💾

Volumes are used for persistent data.

List volumes:

```bash
docker volume ls
```

Create volume:

```bash
docker volume create postgres-data
```

Inspect:

```bash
docker volume inspect postgres-data
```

Remove:

```bash
docker volume rm postgres-data
```

⚠️ Removing a database volume can permanently delete its data.

---

# 20. PostgreSQL With Persistent Volume

Recommended for PostgreSQL:

```bash
docker volume create postgres-data
```

Then:

```bash
docker run -d \
  --name postgres-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=ecommerce \
  -p 5432:5432 \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:18
```

Architecture:

```text
PostgreSQL Container
        │
        ▼
/var/lib/postgresql/data
        │
        ▼
postgres-data
Docker Volume
```

Even if the container is removed, the volume remains.

---

# 21. Useful Docker Inspection Commands

Inspect container:

```bash
docker inspect <container-name>
```

Inspect image:

```bash
docker image inspect <image>
```

Inspect volume:

```bash
docker volume inspect <volume-name>
```

See container resource usage:

```bash
docker stats
```

See Docker networks:

```bash
docker network ls
```

---

# 22. Docker Network

Create a network:

```bash
docker network create backend-network
```

List networks:

```bash
docker network ls
```

Connect container:

```bash
docker network connect backend-network <container-name>
```

Now containers on the same network can communicate using container/service names.

Example:

```text
NestJS
  │
  ├── postgres-db:5432
  ├── redis-server:6379
  └── rabbitmq:5672
```

Instead of:

```text
localhost
```

---

# 23. Important `localhost` Rule

### Application runs directly on Ubuntu

Use:

```text
localhost:5432
localhost:6379
localhost:5672
localhost:3000
```

### Application runs inside Docker

Do NOT use:

```text
localhost:5432
localhost:6379
localhost:5672
```

Use Docker service/container names:

```text
postgres-db:5432
redis-server:6379
rabbitmq:5672
```

Because:

```text
localhost
```

inside a container means **that container itself**.

---

# 24. Production Architecture

Typical architecture:

```text
                    Internet
                       │
                       ▼
                ┌─────────────┐
                │    Nginx    │
                │   :80/:443  │
                └──────┬──────┘
                       │
                       ▼
                ┌─────────────┐
                │   NestJS    │
                │    :3000    │
                └──────┬──────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     PostgreSQL       Redis       RabbitMQ
       :5432          :6379         :5672
```

Docker can run each service in its own container:

```text
nginx container
nestjs container
postgres container
redis container
rabbitmq container
```

For production, databases may also be hosted as managed services instead of Docker containers.

---

# 25. Most Important Commands — Cheat Sheet

```bash
# Images
docker pull <image>
docker images
docker rmi <image>

# Containers
docker run -d --name <name> <image>
docker ps
docker ps -a
docker start <name>
docker stop <name>
docker restart <name>
docker rm <name>
docker rm -f <name>

# Debug
docker logs <name>
docker logs -f <name>
docker exec -it <name> bash
docker inspect <name>

# Resources
docker stats
docker volume ls
docker network ls
```

---

# 26. Current Development Stack

```text
┌──────────────────────────────────────────────┐
│                  Ubuntu                      │
│                                              │
│  ┌─────────────── Docker ────────────────┐  │
│  │                                       │  │
│  │  Nginx        → 8080                  │  │
│  │  PostgreSQL   → 5432                  │  │
│  │  MySQL        → 3306                  │  │
│  │  Redis        → 6379                  │  │
│  │  RabbitMQ     → 5672                  │  │
│  │  RabbitMQ UI  → 15672                 │  │
│  │                                       │  │
│  └───────────────────────────────────────┘  │
│                                              │
│              NestJS → 3000                   │
│                                              │
└──────────────────────────────────────────────┘
```

---

# 27. Recommended Next Step

After learning these commands, learn:

```text
Docker
  ↓
Docker Network
  ↓
Docker Volume
  ↓
Dockerfile
  ↓
Docker Compose
  ↓
NestJS + PostgreSQL
  ↓
NestJS + Redis
  ↓
NestJS + RabbitMQ
  ↓
Nginx Reverse Proxy
  ↓
Production Deployment
```

The next major step is **Docker Compose**, where all services can be started with:

```bash
docker compose up -d
```

and stopped with:

```bash
docker compose down
```

Instead of manually running every container.
