# 🚀 DevOps Project: Automated CI/CD Pipeline for a 2-Tier Flask Application on AWS

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat&logo=jenkins&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=flat&logo=amazonaws&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=flat&logo=flask&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.9-3776AB?style=flat&logo=python&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04-E95420?style=flat&logo=ubuntu&logoColor=white)

---

## 📋 Table of Contents

1. [Project Overview](#-project-overview)
2. [Tech Stack](#-tech-stack)
3. [Architecture](#-architecture)
4. [Prerequisites](#-prerequisites)
5. [Step 1 — Launch Two EC2 Instances](#step-1--launch-two-ec2-instances)
6. [Step 2 — Configure Security Groups](#step-2--configure-security-groups)
7. [Step 3 — Setup Jenkins Server](#step-3--setup-jenkins-server-instance-1)
8. [Step 4 — Setup Flask App Server](#step-4--setup-flask-app-server-instance-2)
9. [Step 5 — Enable Jenkins SSH Access to App Server](#step-5--enable-jenkins-ssh-access-to-app-server)
10. [Step 6 — GitHub Repository Configuration](#step-6--github-repository-configuration)
11. [Step 7 — Jenkins Pipeline Creation & Execution](#step-7--jenkins-pipeline-creation--execution)
12. [Verify Deployment](#-verify-deployment)
13. [CI/CD Workflow](#-cicd-workflow)
14. [Project Structure](#-project-structure)
15. [Cost Summary](#-cost-summary)
16. [Troubleshooting](#-troubleshooting)

---

## 📌 Project Overview

This project demonstrates a complete **DevOps implementation** for a two-tier web application — a **Flask** backend paired with a **MySQL** database — deployed on **AWS EC2** using a **2-instance architecture**.

Jenkins runs on a dedicated server and deploys the application to a separate App Server via SSH. The application is fully containerized using **Docker** and **Docker Compose**, and the CI/CD pipeline automates the entire build and deployment process on every `git push` to the `main` branch.

**What gets automated:**
- Code is pushed to GitHub
- Jenkins detects the change and triggers the pipeline
- Jenkins SSHs into the App Server and copies the latest files
- Docker Compose tears down old containers and brings up the updated stack
- The app is live within seconds — zero manual steps

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| Application | Python 3.9, Flask |
| Database | MySQL 8 |
| Containerization | Docker, Docker Compose |
| CI/CD | Jenkins (Pipeline as Code) |
| Cloud | AWS EC2 — 2 × t2.micro (Ubuntu 22.04 LTS) |
| Source Control | GitHub |
| Deployment Method | Jenkins → SSH → Docker Compose |

---

## 🏗 Architecture

```
+-----------------+      +----------------------+
|   Developer     |----->|     GitHub Repo      |
| (pushes code)   |      | (Source Code Mgmt)   |
+-----------------+      +----------+-----------+
                                     |
                              webhook / poll
                                     |
                                     v
                    +--------------------------------+
                    |     Jenkins Server             |
                    |     Instance 1 (t2.micro)      |
                    |     IP: {Jenkins Server}          |
                    |                                |
                    |  1. Clones GitHub repo         |
                    |  2. Copies files via SCP       |
                    |  3. SSHs into App Server       |
                    |  4. Runs docker compose        |
                    +---------------+----------------+
                                    |
                              SSH Deploy
                            (Private IP)
                                    |
                                    v
                    +--------------------------------+
                    |     Flask App Server           |
                    |     Instance 2 (t2.micro)      |
                    |     IP: {Flask Server}             |
                    |                                |
                    |  +----------+  +-----------+  |
                    |  |  Flask   |  |   MySQL   |  |
                    |  | :5000    |->|  :3306    |  |
                    |  +----------+  | (internal)|  |
                    |                +-----------+  |
                    |   Docker Network: two-tier     |
                    +--------------------------------+

Access Jenkins : http://{Jenkins ip}
Access Flask   : http://{Flask ip}
```

> **Why 2 instances?** Separating Jenkins and the App Server keeps both within **AWS free tier (t2.micro)** while avoiding CPU exhaustion that occurs when running Jenkins + Docker builds on the same machine.

---

## ✅ Prerequisites

Before you begin, make sure you have:

- An **AWS account** with EC2 access
- A **GitHub account** with your Flask app repository
- Basic knowledge of Linux, Docker, and Git
- **Windows PowerShell** (Admin) for SSH key permission fix
- Two `.pem` key files — one for each EC2 instance

---

## Step 1 — Launch Two EC2 Instances

### Instance 1 — Jenkins Server

| Setting | Value |
|---|---|
| **Name** | `P1-jenkins-server` |
| **AMI** | Ubuntu 22.04 LTS |
| **Instance Type** | `t2.micro` ✅ Free tier |
| **Key Pair** | Create `jenkins-key.pem` |
| **Storage** | 15 GB |

### Instance 2 — Flask App Server

| Setting | Value |
|---|---|
| **Name** | `P1-flask-server` |
| **AMI** | Ubuntu 22.04 LTS |
| **Instance Type** | `t2.micro` ✅ Free tier |
| **Key Pair** | Create `app-key.pem` |
| **Storage** | 15 GB |

### Fix .pem Key Permissions on Windows (PowerShell as Admin)

```powershell
# Run for each .pem file
$keyPath = "C:\path\to\your-key.pem"
icacls $keyPath /reset
icacls $keyPath /inheritance:r
icacls $keyPath /grant:r "$($env:USERNAME):(R)"
```

> ⚠️ SSH will refuse to use the key if other Windows users (like `NT AUTHORITY\Authenticated Users`) have access to the file.

### Connect via SSH

```bash
# Jenkins Server
ssh -i "jenkins-key.pem" ubuntu@<jenkins-public-ip>

# App Server
ssh -i "app-key.pem" ubuntu@<app-public-ip>
```

---

## Step 2 — Configure Security Groups

> ⚠️ Create **separate** security groups for each instance. Do NOT share one security group between both — AWS reuses the last-used SG by default during launch.

### Jenkins Server Security Group — `jenkins-server-sg`

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | Your SSH access |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | Jenkins dashboard |

### Flask App Server Security Group — `flask-app-sg`

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | Your SSH access |
| SSH | TCP | 22 | `jenkins-server-sg` | Jenkins deploys via SSH |
| Custom TCP | TCP | 5000 | 0.0.0.0/0 | Flask app public access |

> ✅ Port **3306 (MySQL) does NOT need an inbound rule** — MySQL stays inside Docker's internal `two-tier` network and is never exposed to the internet.

---

## Step 3 — Setup Jenkins Server (Instance 1)

Good call — your current README uses the **APT repo method**, which already caused issues on Ubuntu 24 (noble).

👉 I’ll give you a **clean, production-safe version** that works on **both Ubuntu 22.04 + 24.04** without GPG problems.

---

# ✅ 🔥 Updated Jenkins Setup (README Ready)

Use this instead of your current section 👇

---

## 🚀 Step 3 — Setup Jenkins Server (Stable Method)

> ⚠️ This method avoids GPG/repository issues and works reliably on Ubuntu 22.04 and 24.04.

---

### 🔹 1. Update system

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 🔹 2. Install required dependencies

```bash
sudo apt install openjdk-17-jdk git docker.io wget -y
```

---

### 🔹 3. Start and enable Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

---

### 🔹 4. Download Jenkins package (Direct Install)

```bash
wget https://pkg.jenkins.io/debian-stable/binary/jenkins_2.452.3_all.deb
```

---

### 🔹 5. Install Jenkins

```bash
sudo dpkg -i jenkins_2.452.3_all.deb
```

---

### 🔹 6. Fix dependencies (if any)

```bash
sudo apt -f install -y
```

---

### 🔹 7. Start and enable Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

---

### 🔹 8. Grant Docker access to Jenkins

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

---

## 🔑 Access Jenkins

### Get initial admin password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

### Open Jenkins in browser

```
http://<jenkins-public-ip>:8080
```

---

## ⚠️ Important Notes

* Always use **Public IP of Jenkins EC2**, not Flask server
* Ensure **port 8080 is open** in Amazon Web Services Security Group
* Jenkins may take **1–2 minutes to fully start**

---

## 🔍 Verify Jenkins is Running

```bash
sudo systemctl status jenkins
```

Expected:

```
active (running)
```

---

# 🧠 Why this update is better

| Old                | New ा                |
| ------------------ | --------------------- |
| GPG key issues     | No repo dependency    |
| Fails on Ubuntu 24 | Works on all versions |
| More debugging     | Faster setup          |

---

# 🚀 Optional (Advanced Note for README)

You can add this line (looks very professional):

> Jenkins was installed using a direct `.deb` package to avoid repository key issues commonly encountered on newer Ubuntu distributions.

---

If you want next:

* I can update your **entire README to production-level (resume-ready)**
* Or fix **pipeline section + diagrams wording**

Just tell me 👍


### Install SSH Agent Plugin

Jenkins → **Manage Jenkins** → **Plugins** → **Available plugins** → search `SSH Agent` → **Install** → Restart Jenkins.

---

## Step 4 — Setup Flask App Server (Instance 2)

SSH into the App server and run:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker and Docker Compose only (no Jenkins needed here)
sudo apt install docker.io docker-compose-v2 git -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker ubuntu

# Pre-pull images to avoid CPU spike during first deployment
docker pull python:3.9-slim
docker pull mysql:latest

# Create app directory for Jenkins to deploy into
mkdir -p /home/ubuntu/app
```

---

## Step 5 — Enable Jenkins SSH Access to App Server

### On Jenkins Server — Generate SSH Key Pair

```bash
# Switch to jenkins user
sudo su - jenkins

# Generate key pair (no passphrase)
ssh-keygen -t rsa -b 2048 -f ~/.ssh/app-server-key -N ""

# Display the PUBLIC key — copy this entire output
cat ~/.ssh/app-server-key.pub
```

### On App Server — Authorize the Jenkins Public Key

```bash
# Paste the public key copied from above
echo "paste-jenkins-public-key-here" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Test the Connection (from Jenkins Server as jenkins user)

```bash
ssh -i ~/.ssh/app-server-key -o StrictHostKeyChecking=no ubuntu@<app-server-private-ip>
```

You should connect without a password prompt ✅

### Add Private Key to Jenkins Credentials

1. Jenkins → **Manage Jenkins** → **Credentials** → **Global** → **Add Credentials**
2. Fill in:
   - **Kind:** `SSH Username with private key`
   - **ID:** `app-server-ssh`
   - **Username:** `ubuntu`
   - **Private Key:** Enter directly → paste output of `cat ~/.ssh/app-server-key`
3. Click **Save**

---

## Step 6 — GitHub Repository Configuration

Your repository must contain these files alongside your Flask app code:

### `Dockerfile`

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    gcc \
    default-libmysqlclient-dev \
    pkg-config && \
    rm -rf /var/lib/apt/lists/*

COPY requirement.txt .
RUN pip install --no-cache-dir -r requirement.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

### `docker-compose.yml`

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

```groovy
pipeline {
    agent any

    environment {
        APP_SERVER = 'ubuntu@172.31.34.158'   // App Server private IP
        APP_DIR    = '/home/ubuntu/app'
    }

    stages {
        stage('Clone Repo') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/zaheerbeg21/P1-Deploying-Two-Tier-Flask-App-.git'
            }
        }

        stage('Copy Files to App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no \
                            docker-compose.yml \
                            Dockerfile \
                            app.py \
                            requirement.txt \
                            $APP_SERVER:$APP_DIR/
                    '''
                }
            }
        }

        stage('Deploy on App Server') {
            steps {
                sshagent(['app-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no $APP_SERVER "
                            cd $APP_DIR &&
                            docker compose down || true &&
                            docker compose up -d --build
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful! Flask app is live at http://65.0.31.38:5000'
        }
        failure {
            echo '❌ Pipeline failed. Check Console Output for details.'
        }
    }
}
```

> 📝 Use the App Server's **private IP** (`172.31.34.158`) for inter-instance communication — traffic between instances in the same VPC via private IP is free and faster.

---

## Step 7 — Jenkins Pipeline Creation & Execution

### 7.1 Create New Pipeline Job

1. Jenkins dashboard → **New Item**
2. Name: `two-tier-flask-pipeline`
3. Select **Pipeline** → **OK**

### 7.2 Configure Pipeline from SCM

1. Scroll to **Pipeline** section
2. **Definition** → `Pipeline script from SCM`
3. **SCM** → `Git`
4. **Repository URL** → `https://github.com/zaheerbeg21/P1-Deploying-Two-Tier-Flask-App-.git`
5. **Branch** → `*/main`
6. **Script Path** → `Jenkinsfile`
7. Click **Save**

### 7.3 Run the Pipeline

- Click **Build Now**
- Monitor via **Stage View** or **Console Output**

Pipeline stages: **Clone Repo → Copy Files to App Server → Deploy on App Server**

---

## ✅ Verify Deployment

### On App Server — check running containers

```bash
docker ps
```

Expected output:
```
CONTAINER ID   IMAGE        STATUS         PORTS                    NAMES
xxxxxxxxxxxx   flask-app    Up 2 minutes   0.0.0.0:5000->5000/tcp   two-tier-app
xxxxxxxxxxxx   mysql        Up 2 minutes   0.0.0.0:3306->3306/tcp   mysql
```

### Access the application in browser

```
http://65.0.31.38:5000
```

---

## 🔄 CI/CD Workflow

```
git push origin main
        │
        ▼
   GitHub Repo
        │  (Jenkins polls / webhook trigger)
        ▼
  Jenkins Server — Instance 1 (52.66.201.106)
  ┌────────────────────────────────────────┐
  │ Stage 1: Clone Repo                    │ → pulls latest code from GitHub
  │ Stage 2: Copy Files via SCP            │ → sends files to App Server
  │ Stage 3: Deploy via SSH                │ → docker compose up --build
  └────────────────────────────────────────┘
        │
        │ SSH via private IP (172.31.34.158)
        ▼
  Flask App Server — Instance 2 (65.0.31.38)
        │
        ▼
  Flask + MySQL containers running 🎉
  Live at http://65.0.31.38:5000
```

---

## 📁 Project Structure

```
your-repo/
├── app.py                  # Flask application entry point
├── requirement.txt         # Python dependencies
├── Dockerfile              # Flask container image definition
├── docker-compose.yml      # Multi-container orchestration
├── Jenkinsfile             # CI/CD pipeline definition
├── message.sql             # Database initialization SQL
├── templates/              # Flask HTML templates
├── diagrams/               # Architecture and workflow diagrams
│   ├── Infrastructure.png
│   └── project_workflow.png
└── README.md               # This file
```

---

## 💰 Cost Summary

| Resource | Type | Cost |
|---|---|---|
| Jenkins EC2 (Instance 1) | t2.micro | ✅ Free tier |
| Flask App EC2 (Instance 2) | t2.micro | ✅ Free tier |
| EBS Storage (2 × 15 GB) | gp2 | ✅ 30 GB free tier |
| VPC Data Transfer (private IP) | Same VPC | ✅ Free |
| **Total** | | **$0 / month** |

> Free tier provides 750 hrs/month per instance — two t2.micro instances running simultaneously are fully covered within the free tier limit.

---

## 🔧 Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Jenkins can't run `docker` | Jenkins user not in docker group | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| Jenkins GPG / NO_PUBKEY error | Wrong key format on Ubuntu 22.04 | Use `gpg --dearmor` method — see Step 3 |
| SSH permission denied (Jenkins → App) | Public key not added to App Server | Add Jenkins public key to `~/.ssh/authorized_keys` on App Server |
| `.pem` bad permissions on Windows | `NT AUTHORITY\Authenticated Users` has access | Run `icacls` fix in PowerShell as Admin |
| Flask container keeps restarting | MySQL not ready yet | `healthcheck` + `depends_on` handles it — wait ~60s |
| Port 5000 not reachable | Security group missing rule | Add TCP :5000 inbound from 0.0.0.0/0 on flask-app-sg |
| Port 8080 not reachable | Jenkins SG missing rule | Add TCP :8080 inbound on jenkins-server-sg |
| Terminal stuck after `systemctl status` | Output opened in `less` pager | Press `q` to exit — use `--no-pager` flag next time |
| Both instances sharing same security group | AWS reused SG during launch | Create separate SGs, assign via Actions → Security → Change Security Groups |
| CPU maxed out / build hanging | t2.micro overloaded with Jenkins + Docker | Use 2 separate t2.micro instances — this is why the 2-instance setup exists |
| `docker compose down` fails | No containers running | `\|\| true` in Jenkinsfile handles this gracefully |
| Repository URL error in Jenkins | Missing `https://` prefix | Use full URL: `https://github.com/username/repo.git` |

---

## 📜 License

This project is open-source and available under the [MIT License](LICENSE).

---

> Built with ❤️ as part of a DevOps learning journey — including all the real-world debugging and problem solving!
> Feel free to fork, star ⭐, and contribute!
