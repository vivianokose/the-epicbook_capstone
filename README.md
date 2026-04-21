# The EpicBook — Production-Ready Dockerized Deployment

![Azure](https://img.shields.io/badge/Azure-VM-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-27.5.1-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-v2.24.0-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-18_Alpine-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-Alpine-009639?style=for-the-badge&logo=nginx&logoColor=white)

> Deploying The EpicBook, a full-stack online bookstore, from source to production on an Azure VM using Docker Compose with three containers, split networks, named volumes, healthchecks, and a reverse proxy.

---

## What This Project Does

This is the capstone project for the Docker for DevOps Engineers track. It takes a real Node.js + MySQL bookstore application and deploys it in a production-style three-container stack:

- **MySQL 8.0** stores 54 books and 53 authors
- **Node.js + Express** serves the API and Handlebars frontend on port 8080
- **Nginx** acts as a reverse proxy, receiving all public traffic on port 80 and forwarding to the app

The database and app are never directly exposed to the internet. Only Nginx faces the public.

---

## Architecture

```
Internet (port 80)
        |
        v
epicbook_nginx (nginx:alpine)
        |  frontend_network
        v
epicbook_app (Node.js + Express, port 8080)
        |  backend_network
        v
epicbook_db (MySQL 8.0, port 3306)
```

| Container | Image | Port | Network |
|---|---|---|---|
| epicbook_nginx | nginx:alpine | 80 (public) | frontend_network |
| epicbook_app | theepicbook-app | 8080 (internal) | frontend + backend |
| epicbook_db | mysql:8.0 | 3306 (internal only) | backend_network |

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Microsoft Azure | Cloud provider |
| Ubuntu 24.04 LTS | VM operating system |
| Docker 27.5.1 | Container runtime |
| Docker Compose v2.24.0 | Multi-container orchestration |
| MySQL 8.0 | Database |
| Node.js 18 Alpine | Application runtime |
| Express.js | Web framework |
| Sequelize | ORM for MySQL |
| Handlebars | Frontend templating |
| Nginx Alpine | Reverse proxy |

---

## Project Structure

```
theepicbook/
├── Dockerfile
├── docker-compose.yml
├── .env                    (not committed to Git)
├── .dockerignore
├── server.js
├── config/
│   └── config.json
├── db/
│   ├── BuyTheBook_Schema.sql
│   ├── books_seed.sql
│   └── author_seed.sql
├── models/
├── routes/
├── views/
├── public/
└── nginx/
    └── nginx.conf
```

---

## Dockerfile (Multi-Stage)

```dockerfile
# Stage 1 - install production dependencies
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2 - production runtime
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```

---

## Nginx Config

```nginx
upstream epicbook_app {
    server app:8080;
}

server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    location / {
        proxy_pass http://epicbook_app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## .env File Structure

```env
# MySQL Configuration
MYSQL_ROOT_PASSWORD=<your-root-password>
MYSQL_DATABASE=bookstore
MYSQL_USER=<your-db-user>
MYSQL_PASSWORD=<your-db-password>

# App Configuration
NODE_ENV=production
PORT=8080
DB_HOST=db
DB_USER=<your-db-user>
DB_PASSWORD=<your-db-password>
DB_NAME=bookstore
```

Never commit your .env file to Git.

---

## Step-by-Step Deployment Guide

### Step 1: Provision the Azure VM

Create a VM with these settings and paste the cloud-init script into Custom Data:

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg
  - lsb-release

runcmd:
  - apt-get update -y
  - apt-get install -y docker.io
  - systemctl enable docker
  - systemctl start docker
  - usermod -aG docker azureuser
  - curl -L "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  - chmod +x /usr/local/bin/docker-compose
```

Open NSG inbound rules: port 22 (SSH) and port 80 (HTTP) only.

### Step 2: Clone and Configure

```bash
git clone https://github.com/pravinmishraaws/theepicbook.git
cd theepicbook
```

Create your .env file and update config/config.json production block to point to the db service host.

### Step 3: Build and Run the Stack

```bash
docker-compose up -d --build
```

### Step 4: Seed the Database

```bash
docker cp db/BuyTheBook_Schema.sql epicbook_db:/tmp/01_schema.sql
docker cp db/author_seed.sql epicbook_db:/tmp/02_author_seed.sql
docker cp db/books_seed.sql epicbook_db:/tmp/03_books_seed.sql

docker exec epicbook_db mysql -u <user> -p<password> bookstore -e "source /tmp/01_schema.sql"
docker exec epicbook_db mysql -u <user> -p<password> bookstore -e "source /tmp/02_author_seed.sql"
docker exec epicbook_db mysql -u <user> -p<password> bookstore -e "source /tmp/03_books_seed.sql"
```

### Step 5: Verify

```bash
docker-compose ps
```

Open http://<YOUR_PUBLIC_IP> in a browser.

---

## Persistence Test

| Step | Command | Result |
|---|---|---|
| Check before | SELECT COUNT(*) FROM Book | 54 |
| Restart stack | docker-compose down and up | All containers recreated |
| Check after | SELECT COUNT(*) FROM Book | 54 |

Data survived a full stack restart because it lives in the db_data named volume.

---

## Healthchecks and Startup Order

```
db (MySQL) starts first
    healthcheck: mysqladmin ping
        |
        | condition: service_healthy
        v
app (Node.js) starts second
    healthcheck: wget http://localhost:8080
        |
        | condition: service_healthy
        v
nginx starts last
```

---

## Logging

| Service | Log Location | Method |
|---|---|---|
| Nginx | /var/log/nginx/ | Named volume nginx_logs |
| Node.js app | stdout | docker-compose logs app |
| MySQL | Container internal | docker-compose logs db |

---

## Runbook

```bash
# Start stack
docker-compose up -d

# Stop stack
docker-compose down

# View logs
docker-compose logs app
docker-compose logs nginx

# Check status
docker-compose ps

# Backup database
docker exec epicbook_db mysqldump -u <user> -p<password> bookstore > backup.sql

# Restart single service
docker-compose restart nginx
```

---

## Success Criteria

- [x] Production-lean multi-stage image
- [x] Three-container Compose stack with healthchecks and depends_on
- [x] Split networks with database never exposed publicly
- [x] Named volumes with persistence proven across restarts
- [x] Nginx reverse proxy routing all traffic
- [x] Secrets managed through .env file
- [x] Logging strategy documented
- [x] Cloud VM deployment publicly accessible

---

## Author

**Vivian Chiamaka Okose**
DevOps Engineer

- GitHub: [vivianokose](https://github.com/vivianokose)
- LinkedIn: [linkedin.com/in/okosechiamaka](https://linkedin.com/in/okosechiamaka)
