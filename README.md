<div align="center">

# 🚀 Dev-to-Prod Promotion Platform

### Production-Style CI/CD, GitOps & Kubernetes Promotion

![AWS](https://img.shields.io/badge/AWS-ECR%20%7C%20IAM%20%7C%20S3-orange?logo=amazonaws&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI-red?logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-Packaging-0F1689?logo=helm&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform&logoColor=white)
![Terragrunt](https://img.shields.io/badge/Terragrunt-Orchestration-5C4EE5)

**A hands-on DevOps platform demonstrating how application code is built once, packaged as immutable artifacts, and promoted from Dev → Staging → Production using GitOps.**

</div>

---

# 1. Project Overview

This project demonstrates a production-style application delivery workflow where changes move through controlled CI/CD and GitOps stages:

- source code is versioned in GitHub
- Jenkins checks out and tests the application
- Docker builds the microservice images
- each image receives an immutable Git commit SHA tag
- images are pushed to Amazon ECR
- Jenkins creates a Pull Request against the GitOps repository
- environment promotion happens through Git review and merge
- Argo CD reconciles the merged desired state into Kubernetes
- Dev, Staging and Production are represented by Kubernetes namespaces
- Terraform and Terragrunt manage infrastructure separately from application delivery

The implementation intentionally includes real failures and recovery scenarios encountered during the build.

---

# 2. What This Project Demonstrates

| Capability | Technology | Implementation |
|---|---|---|
| Source Control | GitHub | Application + GitOps repositories |
| Continuous Integration | Jenkins | Checkout, test, Docker build and push |
| Containerization | Docker | Three microservices |
| Artifact Registry | Amazon ECR | Immutable image tags |
| Kubernetes Packaging | Helm | Reusable service charts |
| Continuous Delivery | Argo CD | GitOps synchronization |
| Kubernetes | K3s | Dev / Staging / Production namespaces |
| Ingress | Traefik | External frontend routing |
| Infrastructure as Code | Terraform | AWS infrastructure |
| Terraform Orchestration | Terragrunt | Environment and module organization |
| Remote State | Amazon S3 | Encryption, versioning and locking |
| Access Control | AWS IAM | EC2 role and least-privilege troubleshooting |
| Change Control | GitHub Pull Requests | Promotion gates |

> **Core engineering principle:** Build once, create an immutable artifact, and promote that exact artifact through each environment.

---

# 3. Architecture

## 3.1 Complete Application Delivery Architecture

<div align="center">

<img src="docs/ci-cd-architecture.svg" alt="Application Delivery Architecture" width="100%">

</div>

The delivery responsibility is intentionally separated:

| Stage | Owner |
|---|---|
| Source code | GitHub |
| CI / build | Jenkins |
| Artifact storage | Amazon ECR |
| Deployment desired state | GitOps repository |
| Deployment / reconciliation | Argo CD |
| Runtime | Kubernetes |

---

## 3.2 Infrastructure Architecture

<div align="center">

<img src="docs/infrastructure-architecture.svg" alt="Infrastructure Architecture" width="100%">

</div>

Application delivery and infrastructure management are separate concerns.

### Terraform vs Terragrunt

**Terraform** is the infrastructure provisioning engine. It creates and manages AWS resources.

**Terragrunt** is the orchestration/configuration layer around Terraform. It helps with:

- DRY configuration
- environment organization
- remote-state configuration
- dependencies
- reusable Terraform modules
- coordinating multiple infrastructure units

### Interview answer

> Terraform defines how infrastructure is created. Terragrunt helps organize, configure and orchestrate Terraform across environments and infrastructure units.

---

# 4. Repository Structure

This platform uses three implementation repositories plus this umbrella repository.

```text
dev-to-prod-promotion/
│
├── README.md
├── docs/
│   ├── ci-cd-architecture.svg
│   ├── promotion-flow.svg
│   └── infrastructure-architecture.svg
│
├── microservices-app/
│   ├── Jenkinsfile
│   ├── README.md
│   ├── services/
│   │   ├── frontend/
│   │   ├── product-service/
│   │   └── order-service/
│   ├── scripts/
│   └── docs/
│
├── microservices-gitops/
│   ├── README.md
│   ├── argocd/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   ├── helm/
│   │   ├── frontend/
│   │   ├── product-service/
│   │   └── order-service/
│   ├── k8s/
│   └── scripts/
│
└── infrastructure/
    ├── README.md
    ├── environments/
    │   ├── dev/
    │   ├── staging/
    │   ├── production/
    │   └── shared/ecr/
    ├── modules/
    │   ├── vpc/
    │   ├── eks/
    │   ├── iam/
    │   └── ecr/
    └── bootstrap/
```

---

# 5. Application Architecture

The application consists of three microservices:

| Service | Container Port | Kubernetes Service |
|---|---:|---:|
| `frontend` | 8080 | 8080 |
| `product-service` | 8081 | 8081 |
| `order-service` | 8082 | 8082 |

The frontend communicates with the backend services through Kubernetes DNS:

```text
Frontend
   │
   ├── product-service:8081
   │
   └── order-service:8082
```

Frontend endpoints:

```text
GET /
GET /products
GET /orders
GET /health
```

---

# 6. CI/CD Pipeline

The application pipeline follows:

```text
Developer
    ↓
GitHub Application Repository
    ↓
Jenkins
    ├── Checkout
    ├── Test
    ├── Docker Build
    ├── Git SHA Tag
    └── Push to ECR
    ↓
Amazon ECR
    ↓
GitOps Pull Request
    ↓
Review / Merge
    ↓
Argo CD
    ↓
Kubernetes
```

Jenkins is deliberately **not** responsible for the final Kubernetes reconciliation.

Argo CD owns deployment because the GitOps repository is the desired-state source of truth.

---

# 7. Immutable Artifact Strategy

ECR was configured with immutable image tags.

The pipeline initially attempted to reuse:

```text
frontend:1.1.0
```

ECR correctly rejected the push because the tag already existed.

The pipeline was changed to derive the tag from the Git commit:

```groovy
env.IMAGE_TAG = sh(
    script: 'git rev-parse --short=7 HEAD',
    returnStdout: true
).trim()
```

The successful promotion artifact was:

```text
07eda39
```

The three images were therefore:

```text
frontend:07eda39
product-service:07eda39
order-service:07eda39
```

### Why Git SHA tags?

They provide:

- **Immutability**
- **Traceability**
- **Reproducibility**
- **Auditability**
- **Rollback capability**

---

# 8. Build Once, Promote the Same Artifact

<div align="center">

<img src="docs/promotion-flow.svg" alt="Dev to Production Promotion Flow" width="100%">

</div>

The same artifact was promoted through:

```text
DEV
  ↓
STAGING
  ↓
PRODUCTION
```

No environment-specific rebuild was required.

### ❌ Bad pattern

```text
Build → Dev
Build → Staging
Build → Production
```

### ✅ Preferred pattern

```text
Build Once
    ↓
Immutable Artifact
    ↓
Promote Same Artifact
```

This gives confidence that the artifact tested in Staging is the artifact deployed to Production.

---

# 9. GitOps Promotion

Promotion is controlled through Git Pull Requests.

```text
Jenkins
   ↓
Create GitOps PR
   ↓
Review
   ↓
Merge
   ↓
Argo CD
   ↓
Kubernetes
```

The completed promotion sequence was:

| PR | Purpose | Result |
|---|---|---|
| #4 | Promote `07eda39` to Dev | ✅ Healthy |
| #5 | Promote `07eda39` to Staging | ⚠️ Configuration issue discovered |
| #6 | Fix Staging ECR configuration | ✅ Healthy |
| #7 | Promote `07eda39` to Production | ✅ Healthy |

This is a useful demonstration of a real-world principle:

> **GitOps makes deployment configuration part of the reviewed change process.**

---

# 10. Kubernetes Environment Model

The lab uses one K3s cluster with namespace-based environment separation:

```text
K3s Cluster
│
├── microservices-dev
│   ├── frontend
│   ├── product-service
│   └── order-service
│
├── microservices-staging
│   ├── frontend
│   ├── product-service
│   └── order-service
│
└── microservices-prod
    ├── frontend
    ├── product-service
    └── order-service
```

This is appropriate for a lightweight lab.

A larger production platform may instead use:

- separate Kubernetes clusters
- separate AWS accounts
- separate VPCs
- stronger network isolation

> Namespaces provide environment boundaries here, but they are not equivalent to separate AWS accounts or clusters.

---

# 11. Kubernetes Networking

External frontend traffic follows:

```text
Browser / curl
      ↓
EC2 :80
      ↓
Traefik Ingress Controller
      ↓
frontend Service :8080
      ↓
frontend Pod :8080
```

Internal service communication:

```text
frontend Pod
      ├── product-service:8081
      └── order-service:8082
```

### Important distinction

- **EC2 port 80** → external entry point
- **Traefik** → Ingress Controller
- **Ingress** → Kubernetes routing resource
- **frontend Service :8080** → stable Kubernetes endpoint
- **frontend container :8080** → application listener

---

# 12. Helm

Each microservice has its own Helm chart:

```text
helm/
├── frontend/
├── product-service/
└── order-service/
```

Typical structure:

```text
Chart.yaml
values.yaml
templates/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

The reusable chart contains defaults while environment-specific files provide overrides.

```text
                 Helm Chart
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
        DEV       STAGING      PROD
       Values      Values      Values
          ↓          ↓          ↓
       Release     Release    Release
```

---

# 13. Argo CD

Argo CD continuously compares:

```text
Git Desired State
        ↕
Kubernetes Actual State
```

The environment Applications are:

```text
microservices-dev
microservices-staging
microservices-prod
```

With automated sync and self-healing enabled, Argo CD continuously works to make the cluster match Git.

### Self-healing example

If Git says:

```text
replicas = 1
```

but someone manually changes Kubernetes to:

```text
replicas = 3
```

Argo CD detects the drift and reconciles Kubernetes back toward the desired Git state.

---

# 14. ECR Authentication

K3s uses containerd, while the host Docker Engine has its own image store.

Therefore:

```text
Docker Engine image store
          ≠
K3s/containerd image store
```

An image visible with:

```bash
docker images
```

is not automatically available to K3s.

The final deployment uses Amazon ECR and:

```yaml
imagePullSecrets:
  - name: ecr-registry-secret
```

### ECR authentication incident

A deployment eventually failed with:

```text
403 Forbidden
denied: Your authorization token has expired.
```

The Kubernetes ECR registry secret contained an expired authorization token.

Refreshing the secret restored the deployment.

### Production improvement

Use automatic ECR credential management rather than manually refreshed registry secrets.

---

# 15. Terraform Remote State

Terraform state is stored remotely in Amazon S3.

The backend provides:

- encryption
- versioning
- state locking
- centralized state storage

Configuration:

```hcl
remote_state {
  backend = "s3"

  config = {
    bucket       = "dev-to-prod-promotion-terraform-state"
    key          = "${path_relative_to_include()}/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

---

# 16. Why the S3 Backend Was Bootstrapped

Terraform needs the S3 bucket before it can use the bucket as its remote backend.

Therefore:

```text
Bootstrap Terraform
        ↓
Create S3 State Bucket
        ↓
Enable Versioning / Encryption / Public Access Block
        ↓
Normal Terraform + Terragrunt
        ↓
Remote State
```

The bootstrap configuration created the state bucket separately before normal remote state was used.

---

# 17. Real Troubleshooting Scenarios

This project intentionally documents the failures encountered during implementation.

## 17.1 Immutable ECR Tag Failure

**Symptom**

```text
The image tag '1.1.0' already exists
and cannot be overwritten because the tag is immutable.
```

**Root cause**

ECR correctly prevented an existing immutable artifact from being overwritten.

**Resolution**

Use Git SHA tags.

**Lesson**

> Never build a production pipeline around overwriting release artifacts.

---

## 17.2 Docker Image Not Available to K3s

**Root cause**

Docker Engine and K3s/containerd use separate image stores.

**Resolution**

Move to registry-based deployment through ECR.

**Lesson**

> A host Docker image is not automatically a Kubernetes runtime image.

---

## 17.3 ECR Authentication Expiration

**Symptom**

```text
403 Forbidden
denied: Your authorization token has expired.
