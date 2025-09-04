# 🚀 Todo App Deployment on AWS

> **Full-Stack Application Deployment Guide**  
> Spring Boot + React + Docker + AWS Infrastructure

[![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20S3%20%7C%20CloudFront-orange)](https://aws.amazon.com/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-blue)](https://www.docker.com/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-Backend-green)](https://spring.io/projects/spring-boot)
[![React](https://img.shields.io/badge/React-Frontend-61DAFB)](https://reactjs.org/)

## 📋 Table of Contents

- [🏗️ Architecture Overview](#️-architecture-overview)
- [📦 Source Code Repositories](#-source-code-repositories)
- [🔧 Step-by-Step Deployment](#-step-by-step-deployment)
- [⚙️ Configuration Details](#️-configuration-details)
- [📝 Commands Summary](#-commands-summary)
- [📸 Screenshots](#-screenshots)
- [🌐 Final Access URLs](#-final-access-urls)

## 🏗️ Architecture Overview

This project demonstrates deploying a full-stack Todo Application on AWS using modern DevOps practices:

### 🛠️ Technology Stack

| Component | Technology |
|-----------|------------|
| **Backend** | Spring Boot |
| **Frontend** | React |
| **Database** | PostgreSQL |
| **Cache** | Redis |
| **Queue** | ElasticMQ (SQS Emulator) |
| **Containerization** | Docker & Docker Compose |
| **Reverse Proxy** | NGINX |
| **Cloud Infrastructure** | AWS EC2, S3, CloudFront |

### 🏛️ Deployment Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   CloudFront    │    │      EC2        │    │       S3        │
│   (Frontend)    │◄──►│   (Backend)     │    │   (Assets)      │
│                 │    │                 │    │                 │
│  React App      │    │ Spring Boot API │    │ Static Files    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**🔄 Data Flow:**
- **Backend:** Runs on AWS EC2 inside Docker containers
- **Frontend:** React app served from S3 bucket via CloudFront  
- **API Access:** Through EC2 public IP via NGINX reverse proxy (`/api`)

## 📦 Source Code Repositories

| Repository | Description | Link |
|------------|-------------|------|
| 🏗️ **Infrastructure** | Docker + NGINX + Compose | [`todo-infrastructure`](https://github.com/salahuddin09/todo-infrastructure) |
| 🔧 **Backend** | Spring Boot API | [`todo-backend`](https://github.com/salahuddin09/todo-backend) |
| ⚡ **Worker Service** | Background Processing | [`todo-worker`](https://github.com/salahuddin09/todo-worker) |
| 🎨 **Frontend** | React Application | [`todo-frontend`](https://github.com/salahuddin09/todo-frontend) |

## 🔧 Step-by-Step Deployment

### 🖥️ Backend Deployment on EC2

#### Step 1: Launch EC2 Instance

**Instance Configuration:**
- **AMI:** Amazon Linux 2023
- **Instance Type:** t2.micro (Free Tier)

**🔐 Security Group Configuration:**

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 22 | TCP | `103.137.46.8/32` | SSH Access (Your IP) |
| 80 | TCP | `0.0.0.0/0` | HTTP Access (Public) |
| 8080 | TCP | `103.137.46.8/32` | Direct API Testing |

#### Step 2: Install Docker & Docker Compose

```bash
# Update system packages
sudo yum update -y

# Install Docker
sudo yum install docker -y

# Start Docker service
sudo service docker start

# Add ec2-user to docker group
sudo usermod -aG docker ec2-user

# Verify installation
docker --version
docker-compose --version
```

#### Step 3: Clone Repository & Deploy Backend

```bash
# Clone infrastructure repository
git clone https://github.com/salahuddin09/todo-infrastructure.git

# Navigate to deployment directory
cd todo-infrastructure/todo-deploy

# Start all services
docker-compose up -d
```

#### Step 4: Verify Container Deployment

```bash
# Check running containers
docker ps
```

**✅ Expected Services:**

| Container | Service | Port |
|-----------|---------|------|
| `backend` | Spring Boot API | 8080 |
| `worker` | Background Worker | - |
| `postgres` | PostgreSQL Database | 5432 |
| `redis` | Cache Store | 6379 |
| `elasticmq` | SQS Emulator | 9324 |
| `nginx` | Reverse Proxy | 80 |

#### Step 5: Test Backend API

```bash
# Test API endpoint
curl http://3.80.153.2/api/todos
```

### 🎨 Frontend Deployment (S3 + CloudFront)

#### Step 1: Build React Application

```bash
# Navigate to frontend directory
cd ../todo-frontend

# Install dependencies
npm install

# Create production build
npm run build
```

#### Step 2: Upload Build to S3

```bash
# Sync build files to S3 bucket
aws s3 sync build/ s3://ic-devops-b4-salahuddin-s3-bucket-todo/ --delete
```

#### Step 3: Configure S3 Bucket

- ✅ **Disable Block Public Access** (for HTTP testing)
- ✅ **Enable Static Website Hosting**
- ✅ **Set Index Document:** `index.html`

#### Step 4: Create CloudFront Distribution

**📡 Distribution Settings:**
- **Origin:** S3 bucket endpoint
- **Viewer Protocol Policy:** HTTP Only (for testing)
- **Default Root Object:** `index.html`
- **Error Pages:** Custom 404 → `index.html` (for SPA routing)

## ⚙️ Configuration Details

### 🌐 NGINX Reverse Proxy Configuration

```nginx
events { 
    worker_connections 1024; 
}

http {
    upstream backend {
        server backend:8080;
    }

    server {
        listen 80;

        # Health check endpoint
        location /health {
            return 200 "ok";
            add_header Content-Type text/plain;
        }

        # API proxy configuration
        location /api/ {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # CORS headers for CloudFront
            add_header Access-Control-Allow-Origin "*" always;
            add_header Access-Control-Allow-Methods "GET,POST,PUT,DELETE,OPTIONS" always;
            add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;

            # Handle preflight requests
            if ($request_method = OPTIONS) {
                return 204;
            }
        }
    }
}
```

### 🔑 Environment Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://postgres:5432/tododb` | Database connection |
| `SPRING_DATASOURCE_USERNAME` | `user` | Database username |
| `SPRING_DATASOURCE_PASSWORD` | `password` | Database password |
| `REDIS_HOST` | `redis` | Redis cache host |
| `AWS_REGION` | `us-east-1` | AWS region |
| `SQS_QUEUE_NAME` | `todo-queue` | SQS queue name |

## 📝 Commands Summary

### 🔧 Backend Operations

```bash
# Start all services
docker-compose up -d

# Check service status
docker ps

# View logs
docker-compose logs -f

# Stop all services
docker-compose down
```

### 🎨 Frontend Operations

```bash
# Install dependencies
npm install

# Development server
npm start

# Production build
npm run build

# Deploy to S3
aws s3 sync build/ s3://ic-devops-b4-salahuddin-s3-bucket-todo/ --delete
```

## 📸 Screenshots

### 🖥️ EC2 Instance Running Backend Services

![EC2 Instance](https://github.com/user-attachments/assets/1fac06c4-5005-46a7-af06-15043d6b7706)

### ☁️ CloudFront Distribution Settings

![CloudFront Distribution](https://github.com/user-attachments/assets/e8ff8e62-7884-47b9-9cf6-a0a4bc3b8e44)

### 🌐 Working Application in Browser

![Working Application](https://github.com/user-attachments/assets/77eab3f7-aaa7-4b48-be79-13fbc6ad22a9)

## 🌐 Final Access URLs

### 🔗 Live Application Links

| Service | URL | Description |
|---------|-----|-------------|
| 🎨 **Frontend** | [`http://d11gkewlic9ku8.cloudfront.net/`](http://d11gkewlic9ku8.cloudfront.net/) | React App via CloudFront |
| 🔧 **Backend API** | [`http://3.80.153.2/api/todos`](http://3.80.153.2/api/todos) | REST API Endpoints |

---

---

<div align="center">
  <strong>🎉 Happy Deploying!</strong><br>
  <em>Built with ❤️ by Salahuddin</em>
</div>




















