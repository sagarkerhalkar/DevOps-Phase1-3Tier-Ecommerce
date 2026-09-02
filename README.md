# DevOps Phase 1 - 3-Tier E-Commerce Application

This repository contains my DevOps Phase 1 practical implementation using an existing 3-tier e-commerce application.

The original application was not Dockerized. I containerized the frontend and backend, added MySQL persistence, configured a custom Docker network, added Nginx reverse proxying, created a single Docker Compose stack, and prepared the application for AWS EC2 deployment.

> The original application README is preserved in `README_OLD.md`.

---

## 1. Original Application

**Project:** Mazon Online - Full Stack E-Commerce Application  
**Original source repository:** https://github.com/mohd14shoeb/Ecommerce-Project  
**Fork / DevOps repository:** https://github.com/sagarkerhalkar/DevOps-Phase1-3Tier-Ecommerce

### Original application stack

- Frontend: React.js + Redux
- Backend: Node.js + Express.js
- ORM: Sequelize
- Database: MySQL
- Other integrations: Cloudinary, Nodemailer, Axios, jsPDF

### Existing application features

- User signup and login
- Product search
- Shopping cart
- Checkout
- PDF receipt download
- Password reset
- Admin product management
- Admin category management

---

## 2. DevOps Changes Added

The following DevOps changes were added to the original application:

- Added root `.gitignore`
- Added separate frontend and backend Dockerfiles
- Added `.dockerignore` files
- Added multi-stage Docker builds
- Added Nginx as the frontend production web server
- Replaced hardcoded `http://localhost:3001` frontend API calls with relative `/api/...` paths
- Added Nginx reverse proxying from `/api/` to the backend service
- Added Docker Compose for the complete 3-tier stack
- Added a custom bridge network
- Used Docker service names instead of container IP addresses
- Added MySQL named-volume persistence
- Added MySQL health check
- Kept backend and database ports internal
- Configured backend container to run as the non-root `node` user
- Added `.env.example` while keeping real `.env` secrets out of Git

---

## 3. System Architecture

### Final logical architecture

```text
                        Internet / Browser
                                |
                                | HTTP
                                v
                    +------------------------+
                    | Frontend Container     |
                    | React + Nginx          |
                    | Container Port: 80     |
                    | Public Entry Point     |
                    +-----------+------------+
                                |
                                | /api/*
                                | Docker DNS: backend
                                v
                    +------------------------+
                    | Backend Container      |
                    | Node.js + Express      |
                    | Internal Port: 3001    |
                    | Non-root user: node    |
                    +-----------+------------+
                                |
                                | DB_HOST=db
                                | Docker DNS
                                v
                    +------------------------+
                    | Database Container     |
                    | MySQL 8.0              |
                    | Internal Port: 3306    |
                    +-----------+------------+
                                |
                                v
                    +------------------------+
                    | Named Docker Volume    |
                    | db_data                |
                    | Persistent Storage     |
                    +------------------------+

        All containers communicate through the custom bridge network:
                              app-network
```

### Traffic flow

```text
Browser
  -> Frontend Nginx
  -> /api/*
  -> backend:3001
  -> db:3306
  -> Docker named volume
```

No container IP addresses are hardcoded.

---

## 4. Public vs Internal Ports

### Local Docker test environment

| Service | Container Port | Host Port | Exposure |
|---|---:|---:|---|
| Frontend | 80 | 8080 | Public to host |
| Backend | 3001 | Not published | Internal only |
| MySQL | 3306 | Not published | Internal only |

Current local access:

```text
http://localhost:8080
```

### AWS EC2 target

For the final EC2 deployment, the frontend should be published as:

```yaml
ports:
  - "80:80"
```

Recommended EC2 Security Group rules:

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 80 | TCP | 0.0.0.0/0 | Public HTTP access |
| 22 | TCP | My IP only | SSH administration |

Backend port `3001` and MySQL port `3306` should not be opened publicly.

---

## 5. Project Structure

```text
DevOps-Phase1-3Tier-Ecommerce/
|
|-- back-end/
|   |-- Dockerfile
|   |-- .dockerignore
|   |-- server.js
|   |-- database.js
|   |-- package.json
|   `-- package-lock.json
|
|-- front-end/
|   |-- Dockerfile
|   |-- .dockerignore
|   |-- nginx.conf
|   |-- package.json
|   |-- package-lock.json
|   `-- src/
|
|-- .env.example
|-- .gitignore
|-- docker-compose.yml
|-- README.md
`-- README_OLD.md
```

---

## 6. Backend Dockerfile

The backend uses a multi-stage Docker build.

### Stage 1 - Dependencies

- Base image: `node:22-alpine`
- Copies only `package.json` and `package-lock.json` first
- Installs production-only dependencies with:

```bash
npm ci --omit=dev
```

### Stage 2 - Runtime

- Base image: `node:22-alpine`
- Copies production dependencies from the first stage
- Copies application source code
- Uses `NODE_ENV=production`
- Runs as the non-root `node` user
- Exposes internal port `3001`

Security verification:

```bash
docker compose exec backend whoami
```

Expected output:

```text
node
```

This proves the backend container is not running as root.

---

## 7. Frontend Dockerfile

The frontend uses a true multi-stage build.

### Stage 1 - React build

- Base image: `node:24-alpine`
- Installs frontend dependencies
- Uses `NODE_OPTIONS=--openssl-legacy-provider` because the original React/Webpack stack is older
- Creates the optimized production build with:

```bash
npm run build
```

### Stage 2 - Nginx runtime

- Base image: `nginx:alpine`
- Copies only the generated React `build/` output
- Copies the custom `nginx.conf`
- Serves the application on port `80`

Verification:

```bash
docker compose exec frontend nginx -v
```

Observed:

```text
nginx version: nginx/1.31.4
```

Node.js is intentionally absent from the final frontend runtime:

```bash
docker compose exec frontend node --version
```

Expected result:

```text
node: executable file not found
```

This proves the frontend runtime does not contain unnecessary Node.js build tooling.

---

## 8. Nginx Reverse Proxy

The original frontend had API URLs hardcoded to:

```text
http://localhost:3001
```

That would fail on EC2 because `localhost` in a user's browser refers to the user's own machine.

The frontend now uses relative API paths:

```text
/api/categories
/api/session
/api/cart
/api/order
...
```

Nginx forwards `/api/` requests internally to the backend service:

```nginx
location /api/ {
    proxy_pass http://backend:3001/;
}
```

Request flow:

```text
Browser
  -> /api/categories
  -> Frontend Nginx
  -> backend:3001/categories
  -> MySQL
```

Docker DNS resolves the Compose service name `backend`.

---

## 9. Docker Compose

The complete application is managed using one `docker-compose.yml`.

The stack contains three services:

- `frontend`
- `backend`
- `db`

It also defines:

- Custom network: `app-network`
- Named volume: `db_data`
- MySQL health check
- Environment-variable based database configuration

### Start the full stack

```bash
docker compose up -d --build
```

### Check service status

```bash
docker compose ps
```

Expected design:

```text
frontend   -> host 8080 -> container 80
backend    -> internal 3001
db         -> internal 3306
```

### Stop and remove containers

```bash
docker compose down
```

A normal `docker compose down` removes containers and the Compose network, but preserves the named database volume.

> Do not use `docker compose down -v` when database persistence must be preserved.

---

## 10. Docker Networking

All services are attached to:

```text
app-network
```

This is a custom Docker bridge network.

### Frontend to backend

Nginx uses:

```text
backend:3001
```

### Backend to database

The backend receives:

```text
DB_HOST=db
```

The names `backend` and `db` are Docker Compose service names.

Docker's internal DNS resolves those names to the correct containers, so no fixed container IP addresses are required.

---

## 11. Database Configuration and Persistence

The database service uses the official Docker image:

```text
mysql:8.0
```

MySQL stores its data inside the container at:

```text
/var/lib/mysql
```

Docker Compose mounts that path to:

```text
db_data:/var/lib/mysql
```

The Compose-managed volume created locally is:

```text
devops-phase1-3tier-ecommerce_db_data
```

### Database schema

The application creates multiple real tables, including:

- `users`
- `categories`
- `products`
- `shoppingCarts`
- `productCarts`
- `orders`

---

## 12. Persistence Proof

Persistence was tested through the real application API.

A category record was created:

```text
FINAL-Persistence-Proof-20260902-192554
```

The test sequence was:

```text
1. Create record through POST /api/category
2. Confirm record through GET /api/categories
3. Run docker compose down
4. Confirm containers are removed
5. Confirm Docker named volume still exists
6. Run docker compose up -d
7. Wait for MySQL to become healthy
8. Query GET /api/categories again
9. Confirm the same record still exists
```

After all containers were removed and recreated, the record was still present:

```text
1  FINAL-Persistence-Proof-20260902-192554
```

This proves that database data survives container recreation because MySQL data is stored in a named Docker volume.

---

## 13. MySQL Health Check

The database service includes a health check using `mysqladmin ping`.

The backend is configured to wait until the database becomes healthy:

```yaml
depends_on:
  db:
    condition: service_healthy
```

This prevents the backend from attempting to connect before MySQL is ready.

Observed service state:

```text
db   Up (...) (healthy)
```

---

## 14. Docker Image Sizes

Final locally measured image sizes:

| Image | Disk Usage | Content Size |
|---|---:|---:|
| Frontend | 105 MB | 28.6 MB |
| Backend | 268 MB | 61.5 MB |
| MySQL 8.0 | 1.1 GB | 249 MB |

### Optimization choices

Frontend:

- Multi-stage build
- Alpine-based Node build image
- Nginx-only runtime
- Node.js not present in final image
- `.dockerignore`

Backend:

- Alpine-based Node image
- Multi-stage build
- Production-only dependencies
- Non-root runtime user
- `.dockerignore`

---

## 15. Environment Variables and Secret Handling

Real credentials are stored locally in:

```text
.env
```

The real `.env` file is ignored by Git and must never be committed.

The repository contains:

```text
.env.example
```

Example:

```env
MYSQL_ROOT_PASSWORD=change_me_root_password
MYSQL_DATABASE=shoppingOnline
MYSQL_USER=phase1user
MYSQL_PASSWORD=change_me_app_password
```

Create the local environment file:

### Linux / EC2

```bash
cp .env.example .env
nano .env
```

### Windows PowerShell

```powershell
Copy-Item .env.example .env
notepad .env
```

Then replace the placeholder values with real local credentials.

> `docker compose config` resolves environment variables and can print their values to the terminal. Avoid including resolved secrets in screenshots, documentation, or recordings.

---

## 16. Local Deployment

### Prerequisites

- Git
- Docker Desktop or Docker Engine
- Docker Compose v2

### Clone

```bash
git clone https://github.com/sagarkerhalkar/DevOps-Phase1-3Tier-Ecommerce.git
cd DevOps-Phase1-3Tier-Ecommerce
```

### Create environment file

```bash
cp .env.example .env
```

Edit `.env` and set secure passwords.

### Build and start

```bash
docker compose up -d --build
```

### Verify

```bash
docker compose ps
```

API test:

```bash
curl http://localhost:8080/api/categories
```

Windows PowerShell:

```powershell
Invoke-RestMethod http://localhost:8080/api/categories
```

Open the application:

```text
http://localhost:8080
```

---

## 17. Useful Verification Commands

```bash
docker compose ps
docker compose logs backend --tail 30
docker compose logs frontend --tail 30
docker volume ls
docker network ls
docker image ls
```

Verify backend non-root execution:

```bash
docker compose exec backend whoami
```

Verify frontend runtime:

```bash
docker compose exec frontend nginx -v
```

Verify Node is absent from the frontend runtime:

```bash
docker compose exec frontend node --version
```

---

## 18. AWS EC2 Deployment Plan

Local Docker and Compose validation is complete.

The AWS EC2 deployment evidence should be added after deployment is performed.

### Recommended EC2 flow

1. Launch an Ubuntu EC2 instance.
2. Configure Security Group:
   - SSH 22 from your IP only
   - HTTP 80 from `0.0.0.0/0`
3. Connect through SSH.
4. Install Git, Docker Engine, and Docker Compose plugin.
5. Clone this repository.
6. Create `.env` from `.env.example`.
7. Change the frontend Compose port mapping from:

```yaml
- "8080:80"
```

to:

```yaml
- "80:80"
```

8. Run:

```bash
docker compose up -d --build
```

9. Verify:

```bash
docker compose ps
```

10. Open:

```text
http://<EC2-PUBLIC-IP>
```

### EC2 evidence to capture

- EC2 instance launch
- Security Group inbound rules
- SSH session
- `docker compose up -d --build`
- `docker compose ps`
- Browser showing the application through the EC2 public IP

---

## 19. Git Hygiene

Important DevOps commits currently include:

```text
49f7566  chore: add gitignore and remove OS metadata
9db2f2e  docker: add optimized backend container
bde3e51  config: replace hardcoded backend URLs with API paths
95eabc8  docker: add optimized frontend image and nginx reverse proxy
cc71a2f  docker: add compose stack with network and persistent database
```

The repository uses `.gitignore` for:

- `node_modules`
- `.env`
- build output
- logs
- operating-system files
- IDE metadata
- temporary files

---

## 20. Evidence Collected

Completed locally:

- Original 3-tier application verified
- MySQL database verified with multiple tables
- Backend API verified
- Frontend React application verified
- Backend Docker image built successfully
- Frontend Docker image built successfully
- Multi-stage frontend runtime verified
- Backend non-root user verified
- Custom Docker network verified
- Frontend -> Nginx -> backend -> MySQL communication verified
- Backend and database ports kept internal
- MySQL health check verified
- Named-volume persistence verified
- Final image sizes recorded
- Docker Compose stack verified

Pending final submission evidence:

- AWS EC2 deployment screenshots
- Final system-design image/diagram export
- Final ordered screenshot PDF/document
- Loom walkthrough
- LinkedIn write-up

---

## 21. Key Learning

This project demonstrates a complete containerized 3-tier application architecture:

```text
Browser
  -> Nginx / React
  -> Node.js / Express
  -> MySQL
  -> Persistent Docker Volume
```

The main DevOps concepts demonstrated are:

- Git hygiene
- Docker images
- Dockerfiles
- Multi-stage builds
- Image optimization
- Non-root container execution
- Docker Compose
- Custom Docker networking
- Docker DNS / service discovery
- Reverse proxying
- Environment variables
- Secret handling
- Database health checks
- Persistent storage
- Port exposure and service isolation
- AWS EC2 deployment preparation

---

## 22. Original README

The original application's README has been preserved without replacing its project history:

```text
README_OLD.md
```

This allows the original application documentation and the DevOps implementation documentation to remain available separately.
