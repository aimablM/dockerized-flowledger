# FlowLedger: Dockerizing a Fullstack ERP/CRM System

## Overview
FlowLedger is a fullstack open-source ERP/CRM system built with the MERN stack (MongoDB, Express.js, React, Node.js) and designed to help manage quoting, invoicing, and accounting processes. The goal of this project was to take FlowLedger from a localhost development setup to a production-ready Dockerized application.

This documentation outlines the complete Dockerization process, key technical challenges faced, networking issues encountered, and how they were resolved. It serves as both a guide and a portfolio reference for fullstack infrastructure deployment using Docker and Docker Compose.

## Project Architecture
- **Frontend**: React + Vite
- **Backend**: Express.js
- **Database**: MongoDB (Atlas)
- **Containerization**: Docker
- **Orchestration**: Docker Compose

## Dockerization Goals
- Create separate Dockerfiles for frontend and backend
- Set up Docker Compose to orchestrate both services
- Handle environment variables correctly in build and runtime
- Ensure frontend can communicate with backend via browser after containerization
- Prepare the setup for future EC2 and ECS deployments

## Initial Setup
### Dockerfile for Backend (Node.js)
```dockerfile
# backend/Dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
EXPOSE 8888
CMD ["npm", "start"]
```

### Dockerfile for Frontend (React + Vite)
```dockerfile
# frontend/Dockerfile
FROM node:20.9.0 AS builder

WORKDIR /frontend
COPY package*.json ./
RUN npm install
COPY . .
ENV VITE_API_URL=http://backend:8888/api
RUN npm run build

# Stage 2: Serve with a tiny Node-based static server
FROM node:20.9.0
WORKDIR /frontend

RUN npm install -g serve

COPY --from=builder /frontend/dist ./dist

EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
```

### Docker Compose Configuration
```yaml
version: "3.8"
services:
  backend:
    build:
      context: ./backend
    container_name: backend-service
    ports:
      - "8888:8888"
    environment:
      - NODE_ENV=production

  frontend:
    build:
      context: ./frontend
    container_name: frontend-service
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    depends_on:
      - backend
```

## Issues Encountered & Fixes

### 1. Frontend Could Not Reach Backend
**Problem**: The frontend was using `http://backend:8888` to reach the backend. This worked inside Docker but failed when accessed from the browser.

**Reason**: The `backend` hostname is only resolvable inside Docker's internal network. The browser, which runs outside Docker, cannot resolve `backend`.

**Fix**: Updated `.env` in the frontend to:
```env
VITE_BACKEND_SERVER=http://localhost:8888
```
Then rebuilt the frontend image using:
```bash
docker-compose down
docker-compose up --build
```

### 2. Backend Bound to Localhost
**Problem**: The backend server was bound to `localhost`, making it inaccessible to the frontend container.

**Fix**: Updated the backend entry file to:
```js
const server = app.listen(app.get('port'), '0.0.0.0', () => {
  console.log(`Express running → On PORT : ${server.address().port}`);
});
```

### 3. Vite Environment Variables Not Updating
**Problem**: Changing `.env` in the frontend had no effect.

**Reason**: Vite bakes `import.meta.env` at build time. Without rebuilding, old values persist.

**Fix**: Always run with `--build` after env changes:
```bash
docker-compose up --build
```

## Docker Networking Summary
Docker Compose creates a virtual network in which all services (containers) can talk to each other by their service name.

- Inside Docker: `http://backend:8888` works
- In the browser: `http://localhost:8888` must be used to reach backend exposed via Docker `ports:`

Understanding this distinction is crucial to debugging communication between services versus browser-based API calls.

## Screenshots (in `images/` folder)
- Docker Compose terminal build logs
- Running containers in Docker Desktop
- Frontend and backend terminal windows
- DevTools screenshot showing failed vs successful network requests

## What I Learned
- How to write multi-stage Dockerfiles for fullstack services
- How Docker Compose handles container orchestration and networking
- The importance of environment variable scope and timing (build vs runtime)
- Debugging real-world frontend/backend networking issues
- Preparing a containerized app for production use

## What’s Next
- Add NGINX reverse proxy and consolidate frontend/backend routes
- Set up HTTPS using Certbot
- Deploy to EC2 (Docker Compose)
- Migrate to ECS + Fargate for scalable deployment
- Add CI/CD with GitHub Actions

## Repository Purpose
This repo documents the infrastructure-side journey of taking a locally running fullstack application and preparing it for scalable, containerized deployment. It demonstrates DevOps thinking, real-world debugging, and readiness for cloud deployment.

Note: This project is based on the open-source IDURAR ERP/CRM system. For more information and to access the original repository, visit: https://github.com/idurar/idurar-erp-crm 
