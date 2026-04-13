Here are the **Dockerfiles** for both **backend** and **frontend** of the **Server Magic Input Hub** application.

---

## 🐳 Backend Dockerfile

Place this file at `server-magic-hub/backend/Dockerfile`

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --omit=dev

COPY . .

EXPOSE 5000

CMD ["npm", "start"]
```

> **Note**: --omit=dev is the modern replacement for --only=production. It installs only production dependencies and does not require a lockfile.
---

## 🐳 Frontend Dockerfile (multi‑stage – production ready)

Place this file at `server-magic-hub/frontend/Dockerfile`

```dockerfile
# Stage 1: Build the React application
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm install

# Copy source code and build
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine

# Copy built assets from builder stage
COPY --from=builder /app/build /usr/share/nginx/html

# Create Nginx configuration with reverse proxy to backend
# The backend service name is expected to be "backend" (when using docker-compose)
RUN echo 'server { \
    listen 80; \
    server_name localhost; \
    location / { \
        root /usr/share/nginx/html; \
        index index.html; \
        try_files $uri $uri/ /index.html; \
    } \
    location /api { \
        proxy_pass http://backend:5000; \
        proxy_http_version 1.1; \
        proxy_set_header Upgrade $http_upgrade; \
        proxy_set_header Connection "upgrade"; \
        proxy_set_header Host $host; \
        proxy_cache_bypass $http_upgrade; \
    } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

> **Note**: The proxy passes to `http://backend:5000`. This works when both containers are linked via Docker Compose or a custom network with service name `backend`. If you run containers separately, you can override the configuration or use an environment variable.

---

## 🚀 How to Build and Run with Docker Compose (recommended)

Create a `docker-compose.yml` at the root of `server-magic-hub/`:

```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    container_name: magic-hub-backend
    ports:
      - "5000:5000"
    restart: unless-stopped

  frontend:
    build: ./frontend
    container_name: magic-hub-frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped
```

Then run:

```bash
docker-compose up -d
```

Access the app at `http://localhost`

---

## 🔧 Manual Docker commands (without Compose)

### Build images
```bash
# Backend
cd server-magic-hub/backend
docker build -t magic-hub-backend .

# Frontend
cd ../frontend
docker build -t magic-hub-frontend .
```

### Run containers (with a shared network)
```bash
# Create network
docker network create magic-hub-net

# Run backend
docker run -d --name backend --network magic-hub-net -p 5000:5000 magic-hub-backend

# Run frontend
docker run -d --name frontend --network magic-hub-net -p 80:80 magic-hub-frontend
```

The frontend will proxy `/api` requests to `http://backend:5000` because of the network alias `backend`.

---

## 📁 Final project structure

```
server-magic-hub/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── public/
│   │   └── index.html
│   └── src/
│       ├── index.js
│       └── App.js
└── docker-compose.yml
```

---

## ✅ Final fix – use relative API URLs (no hardcoded backend)

### 1. Edit `App.js` in your frontend source

```bash
cd /workspaces/codespaces-blank/nodejs-2tier/frontend/src
nano App.js
```

Find the line:
```javascript
const API_URL = `http://backend:5000`;
```

Change it to:
```javascript
const API_URL = '';   // empty = relative URLs
```

Save the file.

### 2. Rebuild the frontend Docker image

```bash
cd /workspaces/codespaces-blank/nodejs-2tier/frontend
docker build -t magic-hub-frontend .
```

### 3. Remove the old frontend container and run a new one

```bash
docker rm -f frontend
docker run -d --name frontend --network magic-hub-net -p 80:80 magic-hub-frontend
```

### 4. Test the application

Open your browser at the Codespace’s public URL (or `http://localhost:80`).  
You should now see the 5 default spells loaded correctly.

