# 🎬 Disney+ Hotstar Clone — End-to-End Secure & Scalable Deployment on Kubernetes EKS

> A production-grade **DevSecOps CI/CD project** deploying a Disney+ Hotstar clone application on **AWS EKS** using Jenkins, Docker, Kubernetes, Terraform, and a full security + monitoring stack.

---

## 📌 Project Overview

This project demonstrates an enterprise-style deployment pipeline for a containerized streaming application. It covers infrastructure provisioning, security scanning, continuous integration/delivery, Kubernetes orchestration, SSL termination via Cloudflare, and real-time monitoring with Prometheus and Grafana.

**Live GitHub Repo:** [ananyaamishraa/hotstar-clone-deployment-project](https://github.com/ananyaamishraa/hotstar-clone-deployment-project)

---

## 🏗️ Architecture

```
Developer Push → Jenkins CI/CD Pipeline
                     │
          ┌──────────┼──────────────────┐
          │          │                  │
     SonarQube   OWASP DC          Trivy Scan
     (Code       (Dependency        (Image &
      Quality)    Vulnerabilities)   FS Scan)
          │          │                  │
          └──────────┴──────────────────┘
                     │
              Docker Build & Push
               (Docker Hub Registry)
                     │
           Kubernetes EKS Cluster (AWS)
           ┌─────────────────────────┐
           │  Deployment (2 replicas)│
           │  Service (LoadBalancer) │
           │  Auto-Healing Enabled   │
           └─────────────────────────┘
                     │
              Cloudflare (CNAME + SSL)
                     │
              Public HTTPS Access
                     │
         Prometheus + Grafana (Monitoring)
```

---

## 🛠️ Tech Stack

| Category | Tools |
|---|---|
| **Containerization** | Docker |
| **Orchestration** | Kubernetes (AWS EKS) |
| **CI/CD** | Jenkins (Parameterized Pipeline) |
| **Infrastructure as Code** | Terraform |
| **Code Quality** | SonarQube |
| **Vulnerability Scanning** | Trivy, OWASP Dependency-Check |
| **Monitoring** | Prometheus, Grafana, Blackbox Exporter, Node Exporter |
| **Notifications** | Gmail SMTP (Email Alerts) |
| **Cloud Provider** | AWS (EC2, EKS, IAM, VPC) |
| **Version Control** | GitHub |

---

## 📋 Prerequisites

- AWS Account with IAM permissions (EC2, EKS, VPC, CloudFormation, IAM)
- GitHub account with a Personal Access Token
- Docker Hub account
- Gmail account with 2FA enabled (for SMTP app password)
- NVD API key (for OWASP Dependency-Check)
- Cloudflare account (for DNS + SSL)

---

## 🚀 Setup & Deployment Guide

### I. Infrastructure Setup (AWS EC2)

1. **Launch EC2 Instance**
   - AMI: Ubuntu Server 22.04 LTS
   - Instance Type: `t3.xlarge` (4 vCPUs, 16 GB RAM)
   - Storage: 30 GB GP3 SSD

2. **Configure Security Group** — open the following ports:

   | Port | Protocol | Purpose |
   |------|----------|---------|
   | 22 | TCP | SSH |
   | 80 | TCP | HTTP |
   | 443 | TCP | HTTPS |
   | 8080 | TCP | Jenkins |
   | 9000 | TCP | SonarQube |
   | 3000 | TCP | Grafana / App |
   | 587 | TCP | SMTP (TLS) |
   | 465 | TCP | SMTP (SSL) |

3. **Install all required tools** via the provided shell scripts:
   ```bash
   git clone https://github.com/ananyaamishraa/hotstar-clone-deployment-project.git
   cd hotstar-clone-deployment-project/scripts/
   chmod +x *.sh
   # Run individual scripts for: jenkins, docker, awscli, terraform, kubectl, eksctl, trivy, grafana
   ```

---

### II. Jenkins Configuration

1. Access Jenkins at `http://<PUBLIC_IP>:8080`
2. Unlock with the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Install the following plugins:
   - Eclipse Temurin Installer (JDK)
   - SonarQube Scanner
   - NodeJS Plugin (v16.20.2)
   - OWASP Dependency-Check
   - Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step
   - Stage View / Blue Ocean

4. **Configure Global Tools** (`Manage Jenkins → Tools`):
   - JDK: `jdk-17.0.9+9` via adoptium.net
   - NodeJS: `17.9.0` via nodejs.org
   - SonarQube Scanner: `7.0.2.4839`
   - Dependency-Check: `12.1.0`

---

### III. SonarQube Setup

1. Start the SonarQube container:
   ```bash
   docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
   ```
2. Access at `http://<PUBLIC_IP>:9000` (default: `admin / admin`)
3. Generate an authentication token: `Administration → Security → Users → Tokens`
4. Add a webhook pointing back to Jenkins:
   ```
   http://<JENKINS_IP>:8080/sonarqube-webhook/
   ```
5. Add the SonarQube token as a **Secret Text** credential in Jenkins with ID `Sonar-token`

---

### IV. Credentials to Configure in Jenkins

| Credential ID | Kind | Purpose |
|---|---|---|
| `github-token` | Username + Password | GitHub repo access |
| `docker` | Username + Password | Docker Hub push |
| `Sonar-token` | Secret Text | SonarQube integration |
| `smtp-gmail` | Username + Password | Email notifications |
| `AWS_ACCESS_KEY_ID` | Secret Text | AWS infrastructure |
| `AWS_SECRET_ACCESS_KEY` | Secret Text | AWS infrastructure |

---

### V. Gmail SMTP Setup

1. Enable **2-Step Verification** on your Google account
2. Generate an **App Password** (Google Account → Security → App Passwords → "Jenkins SMTP")
3. Configure in Jenkins (`Manage Jenkins → Configure System → E-mail Notification`):
   - SMTP Server: `smtp.gmail.com`
   - Port: `587` (TLS) or `465` (SSL)
   - Use SMTP Authentication: ✅
   - Credentials: your Gmail + app password

---

### VI. Main CI/CD Pipeline

The pipeline (`Jenkinsfile`) runs these stages:

```
clean workspace → Checkout from Git → SonarQube Analysis → Quality Gate
→ Install Dependencies → OWASP FS Scan → Trivy FS Scan
→ Docker Build & Push → Trivy Image Scan → Deploy to Container
→ Post: Email Notification (with trivy reports attached)
```

Key pipeline snippet:
```groovy
pipeline {
    agent any
    tools { jdk 'jdk'; nodejs 'node' }
    environment { SCANNER_HOME = tool 'sonar-scanner' }
    stages {
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t hotstar ."
                        sh "docker tag hotstar <your-dockerhub-username>/hotstar:latest"
                        sh "docker push <your-dockerhub-username>/hotstar:latest"
                    }
                }
            }
        }
        // ... additional stages
    }
}
```

---

### VII. EKS Cluster Provisioning

1. **Configure AWS CLI:**
   ```bash
   aws configure
   # Enter: ACCESS_KEY, SECRET_KEY, region, output format
   ```

2. **Create the EKS cluster** (takes 5–10 minutes):
   ```bash
   # Mumbai region
   eksctl create cluster --name hotstar-cluster \
     --region ap-south-1 \
     --node-type t2.medium \
     --zones ap-south-1a,ap-south-1b

   # OR US East (N. Virginia)
   eksctl create cluster --name hotstar-cluster \
     --region us-east-1 \
     --node-type t2.medium \
     --zones us-east-1a,us-east-1b
   ```

3. **Deploy to Kubernetes:**
   ```bash
   cd hotstar-clone-deployment-project/K8S/
   kubectl apply -f manifest.yml
   kubectl get all
   ```

   The `manifest.yml` deploys 2 replicas with a `LoadBalancer` service (port 80 → 3000).

4. **Test Kubernetes Auto-Healing:**
   ```bash
   kubectl delete pod <pod-name>
   # Kubernetes automatically replaces the deleted pod
   kubectl get pods  # New pod appears with a fresh name
   ```

---

### VIII. Cloudflare DNS & SSL

1. Go to **Cloudflare DNS Settings** for your domain
2. Add a **CNAME record**:
   - Name: `hotstar` (or your subdomain)
   - Target: EKS LoadBalancer external hostname
   - Proxy Status: **Proxied** (Orange Cloud) — enables Cloudflare SSL
3. Application becomes accessible at `https://hotstar.yourdomain.com`

---

### IX. Monitoring Setup (Prometheus + Grafana)

#### Grafana
Installed via `grafana.sh` script on a dedicated monitoring EC2 instance (provisioned by the Jenkins + Terraform pipeline).

Access at: `http://<MONITORING_SERVER_IP>:3000` (default: `admin / admin`)

#### Prometheus + Blackbox Exporter
Edit `prometheus.yml` to add scrape jobs:

```yaml
- job_name: 'blackbox'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - http://prometheus.io
        - http://<APP_IP>:3000
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: <MONITORING_IP>:9115

- job_name: node_exporter
  static_configs:
    - targets: ['<APP_IP>:9100']
```

Restart Prometheus after editing:
```bash
pgrep prometheus | xargs kill
./prometheus &
```

#### Connect Prometheus to Grafana
1. `Connections → Data Sources → Add Data Source → Prometheus`
2. URL: `http://localhost:9090`
3. Click **Save & Test**

#### Import Dashboards
- Dashboards → Import → ID **13659** (Prometheus Blackbox Exporter HTTP prober)

---

### X. Monitoring Server via Jenkins + Terraform

A separate Jenkins job (`monitoringserver`) provisions and tears down the monitoring EC2 instance using Terraform, with a **Choice Parameter** (`apply` / `destroy`) for on-demand infrastructure.

---

### XI. Cleanup

To avoid AWS charges, tear down all resources:

```bash
# Destroy monitoring server via Jenkins pipeline (select "destroy")

# Delete EKS cluster
eksctl delete cluster --name hotstar-cluster --region ap-south-1

# Terminate EC2 instances from the AWS Console
```

---

## 📁 Repository Structure

```
hotstar-clone-deployment-project/
├── scripts/               # Shell scripts for tool installation
│   ├── jenkins.sh
│   ├── docker.sh
│   ├── awscli.sh
│   ├── terraform.sh
│   ├── kubectl.sh
│   ├── eksctl.sh
│   ├── trivy.sh
│   └── grafana.sh
├── K8S/
│   └── manifest.yml       # Kubernetes Deployment + LoadBalancer Service
├── Terraform/             # Terraform configs for monitoring server
├── Dockerfile
├── Jenkinsfile            # Main CI/CD pipeline
└── README.md
```

---

## 🔐 Security Highlights

- **SonarQube** — static code analysis with quality gate enforcement
- **Trivy** — filesystem and Docker image vulnerability scanning (reports emailed post-build)
- **OWASP Dependency-Check** — CVE scanning of npm dependencies using NVD API
- **Cloudflare** — DDoS protection + TLS 1.3 SSL termination
- **Jenkins credentials store** — all secrets stored as encrypted credentials, never hardcoded
- **IAM least-privilege** — dedicated AWS user with scoped permissions for EKS provisioning

---

## 📧 Email Notifications

Jenkins sends automated build status emails after every pipeline run, including:
- Build status (SUCCESS / FAILURE)
- Project name and build number
- Triggered by (user or GitHub)
- Trivy scan reports (`trivyfs.txt`, `trivyimage.txt`) as attachments

---

## 📚 References & Credits

- **Tutorial by:** [Mohammed Aseem Akram](https://www.linkedin.com/in/mohammed-aseem-akram) — [CloudAseem YouTube](https://www.youtube.com/@clouddevopswithaseem)
- **Reference GitHub:** [Aseemakram19/hotstar-kubernetes](https://github.com/Aseemakram19/hotstar-kubernetes)

---

## 🏷️ Tags

`#DevSecOps` `#Kubernetes` `#Jenkins` `#Docker` `#AWS` `#EKS` `#Terraform` `#SonarQube` `#Trivy` `#OWASP` `#Prometheus` `#Grafana` `#Cloudflare` `#CI/CD` `#Cloud`
