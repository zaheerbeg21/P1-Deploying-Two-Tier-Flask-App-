# 🚀 DevOps Project: Automated CI/CD Pipeline for a 2-Tier Flask Application on AWS

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat&logo=jenkins&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=flat&logo=amazonaws&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=flat&logo=flask&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.9-3776AB?style=flat&logo=python&logoColor=white)

---

## 📋 Table of Contents

1. [Project Overview](#-project-overview)
2. [Tech Stack](#-tech-stack)
3. [Architecture](#-architecture)
4. [Prerequisites](#-prerequisites)
5. [Step 1 — AWS EC2 Setup](#step-1--aws-ec2-setup)
6. [Step 2 — Install Dependencies](#step-2--install-dependencies-on-ec2)
7. [Step 3 — Jenkins Installation & Setup](#step-3--jenkins-installation--setup)
8. [Step 4 — GitHub Repository Configuration](#step-4--github-repository-configuration)
9. [Step 5 — Jenkins Pipeline Creation & Execution](#step-5--jenkins-pipeline-creation--execution)
10. [Verify Deployment](#-verify-deployment)
11. [CI/CD Workflow](#-cicd-workflow)
12. [Project Structure](#-project-structure)
13. [Troubleshooting](#-troubleshooting)

---

## 📌 Project Overview

This project demonstrates a complete **DevOps implementation** for a two-tier web application — a **Flask** backend paired with a **MySQL** database — deployed on **AWS EC2**.

The application is fully containerized using **Docker** and **Docker Compose**, and a **Jenkins CI/CD pipeline** automates the build and deployment process on every `git push` to the `main` branch. No manual deployment steps are needed after the initial setup.

**What gets automated:**
- Code is pushed to GitHub
- Jenkins detects the change and triggers the pipeline
- Docker builds a fresh image from the latest code
- Docker Compose tears down old containers and brings up the updated stack
- The app is live within seconds

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Application | Python 3.9, Flask |
| Database | MySQL 8 |
| Containerization | Docker, Docker Compose |
| CI/CD | Jenkins (Pipeline as Code) |
| Cloud | AWS EC2 (Ubuntu 22.04 LTS) |
| Source Control | GitHub |

---

## 🏗 Architecture

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on AWS EC2 :8080)         |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys (same EC2)
                                                                      v
                                                       +-----------------------------+
                                                       |      AWS EC2 Instance       |
                                                       |      (Ubuntu 22.04)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Flask Container  :5000  | |
                                                       | +----------+--------------+ |
                                                       |            | (two-tier net) |
                                                       |            v               |
                                                       | +-------------------------+ |
                                                       | | MySQL Container  :3306  | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```

> **Note:** In this project, Jenkins and the application both run on the same EC2 instance. For production environments, separate instances are recommended.

---

## ✅ Prerequisites

Before you begin, make sure you have:

- An **AWS account** with EC2 access
- A **GitHub account** with My Flask app repository
- Basic knowledge of Linux, Docker, and Git
- My EC2 **key pair** (`.pem` file) for SSH access

---

## Step 1 — AWS EC2 Setup

### 1.1 Launch EC2 Instance

- Navigate to the AWS EC2 console and click **Launch Instance**
- **AMI:** Ubuntu 22.04 LTS
- **Instance type:** `t2.micro` (free-tier eligible)
- **Key pair:** Create a new key pair and download the `.pem` file

### 1.2 Configure Security Group

Add the following **inbound rules** to My security group:

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | Secure shell access |
| HTTP | TCP | 80 | 0.0.0.0/0 | Web traffic |
| Custom TCP | TCP | 5000 | 0.0.0.0/0 | Flask application |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | Jenkins dashboard |

### 1.3 Connect to EC2

```bash
ssh -i /path/to/My-key.pem ubuntu@<ec2-public-ip>
```

---

## Step 2 — Install Dependencies on EC2

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Git, Docker, and Docker Compose
sudo apt install git docker.io docker-compose-v2 -y

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Allow current user to run Docker without sudo
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker is running:

```bash
docker --version
docker compose version
```

---

## Step 3 — Jenkins Installation & Setup

### 3.1 Install Java (Required by Jenkins)

```bash
sudo apt install openjdk-17-jdk -y
java -version
```

### 3.2 Install Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
```

### 3.3 Start Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

### 3.4 Initial Setup

1. Get the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
2. Open `http://<ec2-public-ip>:8080` in browser
3. Paste the password, install **suggested plugins**, and create admin user

### 3.5 Grant Jenkins Docker Permissions

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

> ⚠️ This step is critical — without it, Jenkins pipeline stages that run `docker` commands will fail with a permission denied error.

---

## Step 4 — GitHub Repository Configuration

Repository must contain the following three files alongside My Flask application code.

### `Dockerfile`

Defines the Flask application container image:

```dockerfile
# Use official Python 3.9 slim base image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Install system dependencies for mysqlclient
RUN apt-get update && apt-get install -y \
    gcc \
    default-libmysqlclient-dev \
    pkg-config && \
    rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies first (leverages Docker layer cache)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application source code
COPY . .

# Expose Flask port
EXPOSE 5000

# Start the application
CMD ["python", "app.py"]
```

### `docker-compose.yml`

Orchestrates the Flask + MySQL two-tier stack:

```yaml
version: "3.8"

services:
  mysql:
    container_name: mysql
    image: mysql
    environment:
      MYSQL_DATABASE: "devops"
      MYSQL_ROOT_PASSWORD: "root"
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - two-tier
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask:
    build:
      context: .
    container_name: two-tier-app
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=devops
    networks:
      - two-tier
    depends_on:
      - mysql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

volumes:
  mysql-data:

networks:
  two-tier:
```

### `Jenkinsfile`

Pipeline-as-code definition — Jenkins reads this automatically:

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/My-username/My-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                // Tear down existing containers gracefully
                sh 'docker compose down || true'
                // Rebuild and start updated containers
                sh 'docker compose up -d --build'
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful! App is live at http://<ec2-ip>:5000'
        }
        failure {
            echo '❌ Pipeline failed. Check Console Output for details.'
        }
    }
}
```

> 📝 Replace `username/repo.git` with actual GitHub repository URL.

---

## Step 5 — Jenkins Pipeline Creation & Execution

### 5.1 Create a New Pipeline Job

1. From the Jenkins dashboard, click **New Item**
2. Enter a project name (e.g., `two-tier-flask-pipeline`)
3. Select **Pipeline** and click **OK**

### 5.2 Configure Pipeline from SCM

1. Scroll to the **Pipeline** section
2. Set **Definition** → `Pipeline script from SCM`
3. Set **SCM** → `Git`
4. Enter My **GitHub repository URL**
5. Set **Branch** → `*/main`
6. Set **Script Path** → `Jenkinsfile`
7. Click **Save**

### 5.3 Run the Pipeline

- Click **Build Now** to trigger the first run manually
- Monitor the progress in **Stage View** or **Console Output**

The pipeline runs three stages: **Clone Code → Build Docker Image → Deploy with Docker Compose**

---

## ✅ Verify Deployment

After a successful pipeline run, check that containers are up:

```bash
docker ps
```

You should see both `two-tier-app` (Flask) and `mysql` containers with status `Up`.

Access My application at:

```
http://<ec2-public-ip>:5000
```

---

## 🔄 CI/CD Workflow

Once the pipeline is configured, every push to `main` triggers the full automated flow:

```
git push origin main
       │
       ▼
  GitHub Repo
       │  (Jenkins polls or webhook)
       ▼
  Jenkins Pipeline
  ┌────────────────────┐
  │ Stage 1: Clone     │  → pulls latest code from GitHub
  │ Stage 2: Build     │  → docker build -t flask-app:latest .
  │ Stage 3: Deploy    │  → docker compose down && docker compose up -d --build
  └────────────────────┘
       │
       ▼
  Flask app updated and live at :5000 🎉
```

---

## 📁 Project Structure

```
Repo/
├── app.py                  # Flask application entry point
├── requirements.txt        # Python dependencies
├── Dockerfile              # Flask container image definition
├── docker-compose.yml      # Multi-container orchestration
├── Jenkinsfile             # CI/CD pipeline definition
├── diagrams/               # Architecture and workflow diagrams
│   ├── Infrastructure.png
│   └── project_workflow.png
└── README.md               # This file
```

---

## 🔧 Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Jenkins can't run `docker` | Jenkins user not in docker group | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| Flask container keeps restarting | MySQL not ready yet | `depends_on` + `healthcheck` handles this — wait ~60s |
| Port 5000 not reachable | Security group missing rule | Add inbound TCP :5000 from 0.0.0.0/0 in AWS console |
| `docker compose down` fails | No containers running | The `|| true` in Jenkinsfile handles this gracefully |
| Jenkins initial password not found | Jenkins not started | `sudo systemctl start jenkins` then retry |

---

## 📜 License

This project is open-source and available under the [MIT License](LICENSE).

---

> Built with ❤️ as part of a DevOps learning journey. Feel free to fork, star ⭐, and contribute!
