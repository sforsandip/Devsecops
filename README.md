# 🔐 DevSecOps Pipeline — Secure CI/CD with Automated Security Testing

A production-grade **DevSecOps** project demonstrating how security is integrated at every stage of the CI/CD pipeline — from static code analysis to live vulnerability scanning — using a deliberately vulnerable Java web application (EasyBuggy) as the target.

> **"Shift Left Security"** — Security checks run automatically on every commit, catching vulnerabilities before they reach production.

---

## 🏗️ Architecture Overview

```
Developer Push (GitHub)
        ↓
  Jenkins CI/CD Pipeline (AWS EC2 - Amazon Linux 2023)
        ↓
  ┌─────────────────────────────────────────────────────────┐
  │  SAST: SonarCloud Analysis (code quality + security)    │
  │  SCA:  Snyk Test (dependency vulnerability scanning)    │
  └─────────────────────────────────────────────────────────┘
        ↓
  Docker Build → Push to AWS ECR
        ↓
  Kubernetes Deployment → AWS EKS (namespace: devsecops)
        ↓
  ┌─────────────────────────────────────────────────────────┐
  │  DAST: OWASP ZAP (live application vulnerability scan)  │
  │  ZAP Report archived as Jenkins build artifact          │
  └─────────────────────────────────────────────────────────┘
```

---

## 🛡️ Security Testing Layers

This pipeline implements three distinct layers of automated security testing:

| Layer | Tool | Type | What It Catches |
|---|---|---|---|
| **SAST** | SonarCloud | Static Analysis | Code vulnerabilities, SQL injection patterns, insecure code |
| **SCA** | Snyk | Dependency Scan | Vulnerable libraries, outdated packages, CVEs |
| **DAST** | OWASP ZAP | Dynamic Analysis | XSS, CSRF, injection flaws in the running app |

---

## 🛠️ Full Tech Stack

| Category | Tool |
|---|---|
| Application | Java (Maven), EasyBuggy vulnerable web app |
| CI/CD | Jenkins (Declarative Pipeline) |
| SAST | SonarCloud |
| SCA | Snyk |
| DAST | OWASP ZAP |
| Containerization | Docker |
| Container Registry | AWS ECR |
| Infrastructure as Code | Terraform |
| Orchestration | Kubernetes (AWS EKS) |
| Cloud | AWS (EC2, EKS, ECR, IAM) |
| OS | Amazon Linux 2023 |

---

## 🚀 CI/CD Pipeline Stages

The Jenkins pipeline (`Jenkinsfile`) runs the following stages automatically on every push:

### Stage 1 — SAST: SonarCloud Analysis
Runs `mvn clean verify sonar:sonar` to perform static application security testing. Catches code smells, bugs, security hotspots, and quality gate violations before any build artifact is created.

### Stage 2 — SCA: Snyk Dependency Scan
Runs `mvn snyk:test` using the Snyk Maven plugin to scan all third-party dependencies for known CVEs. Uses Jenkins credentials store for secure token management (`Snyk_token`).

### Stage 3 — Docker Build
Builds the application Docker image using the Dockerfile. Authenticates via Jenkins Docker credentials (`dockerlogin`).

### Stage 4 — Push to AWS ECR
Tags and pushes the Docker image to AWS Elastic Container Registry (`us-east-1`) using AWS IAM credentials stored in Jenkins.

### Stage 5 — Kubernetes Deployment (EKS)
Updates kubeconfig for `kubernetes-cluster-2` in `us-east-1`, clears the `devsecops` namespace, and applies `deployment.yaml` to roll out the latest image with a LoadBalancer service.

### Stage 6 — Wait for Deployment
Waits 180 seconds to allow the application to fully start and become accessible via the EKS LoadBalancer endpoint.

### Stage 7 — DAST: OWASP ZAP Scan
Performs a live security scan against the running application using OWASP ZAP. Dynamically resolves the LoadBalancer hostname via `kubectl`, runs the scan, and archives `zap_report.html` as a Jenkins build artifact.

---

## ☁️ Infrastructure (Terraform)

`main.tf` provisions the Jenkins server on AWS:

- **EC2 Instance** — `c7i-flex.large`, Amazon Linux 2023, 30GB EBS volume
- **Security Group** — Opens port `8080` (Jenkins UI) and port `22` (SSH)
- **IAM Role + Instance Profile** — Grants EC2 instance AWS permissions for ECR, EKS operations
- **User Data** — Runs `install_jenkins.sh` on launch to auto-install Jenkins

### Provision Infrastructure

```bash
cd terraform_files

# Initialize Terraform
terraform init

# Preview resources
terraform plan -var-file="vars/dev-west-2.tfvars"

# Create resources
terraform apply -var-file="vars/dev-west-2.tfvars"

# Get Jenkins initial admin password after EC2 is running
chmod 400 <your-keypair.pem>
ssh -i <your-keypair.pem> ec2-user@<public-dns>
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Destroy when done
terraform destroy -var-file="vars/dev-west-2.tfvars"
```

---

## ☸️ Kubernetes Deployment

`deployment.yaml` defines:

- **Deployment** — 1 replica of the EasyBuggy app pulled from AWS ECR (`imagePullPolicy: Always`)
- **Service** — `LoadBalancer` type, exposing port `80` → container port `8080`
- **Namespace** — `devsecops`

```bash
# Create the namespace
kubectl create namespace devsecops

# Apply manifests
kubectl apply -f deployment.yaml --namespace=devsecops

# Check deployment status
kubectl get deployments --namespace=devsecops

# Get LoadBalancer URL
kubectl get svc --namespace=devsecops

# Clean up
kubectl delete all --all -n devsecops
```

---

## 🔧 EKS Cluster Setup

```bash
# Create cluster
eksctl create cluster \
  --name kubernetes-cluster-2 \
  --version 1.23 \
  --region us-east-1 \
  --nodegroup-name linux-nodes \
  --node-type t2.xlarge \
  --nodes 2

# Update kubeconfig
aws eks update-kubeconfig \
  --name kubernetes-cluster-2 \
  --region us-east-1

# Delete cluster when done
eksctl delete cluster \
  --region=us-east-1 \
  --name=kubernetes-cluster-2
```

---

## 📁 Project Structure

```
Devsecops/
├── src/main/              # EasyBuggy Java application source
├── vars/                  # Terraform variable files (tfvars)
├── Dockerfile             # Container image definition
├── Jenkinsfile            # 7-stage DevSecOps pipeline
├── deployment.yaml        # Kubernetes Deployment + LoadBalancer Service
├── main.tf                # Terraform: EC2, SG, IAM, AMI
├── outputs.tf             # Terraform outputs
├── install_jenkins.sh     # Jenkins auto-install user data script
├── catalina.policy        # Tomcat security policy
└── pom.xml                # Maven build + Snyk/Sonar plugin config
```

---

## 🔒 Security Best Practices Used

- All secrets (SonarCloud token, Snyk token, AWS credentials, Docker login) stored in **Jenkins Credentials Store** — never hardcoded
- Docker image pulled with `imagePullPolicy: Always` to ensure latest security patches
- OWASP ZAP report archived per build for audit trail
- IAM role attached to EC2 instead of using long-lived access keys on the server
- Dedicated Kubernetes namespace (`devsecops`) for workload isolation

---

## 🧹 Docker Maintenance

```bash
# Remove unused images to free disk space
docker system prune

# Remove a specific image
docker image rm <imagename>
```

---

## 👨‍💻 Author

**Sandip** — Aspiring DevOps / DevSecOps Engineer  
Transitioning from IT Admin & Network Support to Cloud & DevOps  
📍 Hyderabad, India

[![GitHub](https://img.shields.io/badge/GitHub-sforsandip-181717?logo=github)](https://github.com/sforsandip)

---

## 📌 Topics

`devsecops` `jenkins` `docker` `kubernetes` `terraform` `aws` `eks` `ecr` `sonarqube` `snyk` `owasp-zap` `sast` `dast` `sca` `ci-cd` `java` `maven` `security-automation`
