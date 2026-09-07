🚀 Dev-to-Prod Promotion Platform

Production-style CI/CD + GitOps platform demonstrating application
promotion from source code to production.










Core delivery path

GitHub → Jenkins → Docker → Amazon ECR → GitOps PR → Argo CD → Kubernetes

The project demonstrates how the same immutable application artifact
is promoted through Development → Staging → Production, while
infrastructure is managed separately with Terraform + Terragrunt.

📌 Project at a Glance

Area

Technology

Purpose

Source control

GitHub

Application and GitOps repositories

CI

Jenkins

Build, test, package and publish

Containers

Docker

Package microservices

Registry

Amazon ECR

Store immutable images

Packaging

Helm

Reusable Kubernetes deployment templates

CD / GitOps

Argo CD

Reconcile Git state to Kubernetes

Kubernetes

K3s

Lightweight Kubernetes platform

Ingress

Traefik

External HTTP routing

IaC

Terraform

Provision AWS infrastructure

IaC orchestration

Terragrunt

Organize Terraform environments/modules

State

Amazon S3

Remote Terraform state

Change control

GitHub PRs

Promotion and approval gates

🏗️ Architecture

Application Delivery

flowchart LR
    A["👨‍💻 Developer"] --> B["GitHub<br/>Application Repo"]
    B --> C["Jenkins CI"]
    C --> D["Docker Build & Test"]
    D --> E["Amazon ECR<br/>Immutable Images"]
    E --> F["GitOps PR"]
    F --> G["GitHub<br/>GitOps Repo"]
    G --> H["Argo CD"]
    H --> I["K3s Cluster"]

    I --> J["DEV<br/>microservices-dev"]
    I --> K["STAGING<br/>microservices-staging"]
    I --> L["PRODUCTION<br/>microservices-prod"]

Infrastructure

flowchart LR
    A["Terraform Modules"] --> B["Terragrunt"]
    B --> C["AWS Infrastructure"]

    C --> D["VPC"]
    C --> E["IAM"]
    C --> F["Amazon ECR"]
    C --> G["Kubernetes Infrastructure"]

    B --> H["Amazon S3<br/>Remote State + Locking"]

Key separation: Jenkins creates/publishes application artifacts.
Argo CD deploys and reconciles them. Terraform/Terragrunt manages
infrastructure.

🎯 Project Goals

This project was built to demonstrate practical Senior DevOps concepts:

CI/CD pipeline design

Docker image lifecycle

Immutable artifacts

AWS ECR

Kubernetes

Helm

GitOps

Argo CD

Pull-request based promotion

Environment separation

Infrastructure as Code

Terraform remote state

Terragrunt orchestration

IAM troubleshooting

Deployment troubleshooting

Rollback strategy

Production change control

📦 Repository Structure

The project is intentionally separated into three implementation
repositories plus this umbrella repository.

dev-to-prod-promotion/
│
├── README.md
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

Repositories

Application: paycenonoli/dev-to-prod-microservices-app

GitOps: paycenonoli/dev-to-prod-microservices-gitops

Infrastructure: paycenonoli/dev-to-prod-infrastructure

Platform overview: paycenonoli/dev-to-prod-promotion

🧩 Microservices

The application contains three services:

Service

Container Port

Kubernetes Service

frontend

8080

8080

product-service

8081

8081

order-service

8082

8082

The frontend calls the backend services using Kubernetes DNS:

frontend
   │
   ├── http://product-service:8081
   │
   └── http://order-service:8082

Frontend endpoints:

GET /
GET /products
GET /orders
GET /health

🔄 CI/CD + GitOps Flow

flowchart TD
    A["Developer pushes source change"] --> B["GitHub Application Repo"]
    B --> C["Jenkins"]
    C --> D["Checkout"]
    D --> E["Test"]
    E --> F["Docker Build"]
    F --> G["Generate Git SHA"]
    G --> H["Push immutable images to ECR"]
    H --> I["Update GitOps values"]
    I --> J["Create GitOps PR"]
    J --> K["Review / Merge"]
    K --> L["Argo CD detects Git change"]
    L --> M["Sync Kubernetes"]

Ownership model

Component

Primary responsibility

GitHub App Repo

Source code

Jenkins

CI + artifact creation

Amazon ECR

Artifact storage

GitHub GitOps Repo

Desired deployment state

GitHub PR

Promotion/change approval

Argo CD

Deployment + reconciliation

Kubernetes

Runtime platform

🔐 Immutable Image Strategy

ECR was configured with immutable image tags.

The pipeline initially attempted to reuse:

frontend:1.1.0

ECR rejected the push because the tag already existed.

Instead, the pipeline generates a tag from the Git commit:

env.IMAGE_TAG = sh(
    script: 'git rev-parse --short=7 HEAD',
    returnStdout: true
).trim()

The successful promotion artifact was:

07eda39

Therefore the application artifacts were:

frontend:07eda39
product-service:07eda39
order-service:07eda39

Why this matters

A Git SHA tag provides:

✅ Immutability

✅ Traceability

✅ Reproducibility

✅ Auditability

✅ Easier rollback

🚀 Build Once, Promote the Same Artifact

This is one of the most important principles demonstrated by the
project.

flowchart LR
    A["Git Commit<br/>07eda39"] --> B["Build Once"]
    B --> C["ECR"]
    C --> D["DEV"]
    D --> E["STAGING"]
    E --> F["PRODUCTION"]

    D --> D1["✅ Healthy"]
    E --> E1["✅ Healthy"]
    F --> F1["✅ Healthy"]

We do not rebuild the application for each environment.

❌ Bad pattern

Build → Dev
Build → Staging
Build → Production

✅ Preferred pattern

Build once
    ↓
Immutable artifact
    ↓
Promote the same artifact

This ensures the artifact tested in Staging is the artifact deployed to
Production.

🌎 Kubernetes Environment Model

The lab uses one K3s cluster with namespace-based environment
separation.

flowchart TB
    K["K3s Cluster"]

    K --> D["microservices-dev"]
    K --> S["microservices-staging"]
    K --> P["microservices-prod"]

    D --> D1["frontend"]
    D --> D2["product-service"]
    D --> D3["order-service"]

    S --> S1["frontend"]
    S --> S2["product-service"]
    S --> S3["order-service"]

    P --> P1["frontend"]
    P --> P2["product-service"]
    P --> P3["order-service"]

This is a lightweight lab architecture.

A larger production platform could use:

Separate Kubernetes clusters

Separate AWS accounts

Separate VPCs

Stronger network isolation

Kubernetes namespaces are environment boundaries in this lab, but they
should not be described as equivalent to separate AWS accounts or
clusters.

🌐 Kubernetes Networking

The external frontend request path is:

flowchart LR
    A["Browser / curl"] --> B["EC2 :80"]
    B --> C["Traefik Ingress Controller"]
    C --> D["frontend Service :8080"]
    D --> E["frontend Pod :8080"]

Inside Kubernetes:

flowchart LR
    F["frontend Pod"] --> P["product-service :8081"]
    F --> O["order-service :8082"]

Important distinction

EC2 port 80 → external entry point exposed by Traefik

Frontend container port 8080 → application port

Kubernetes Service → stable internal endpoint

Ingress → Kubernetes API resource containing routing rules

Traefik → Ingress Controller implementing those rules

📜 Helm

Each microservice has a Helm chart:

helm/
├── frontend/
├── product-service/
└── order-service/

Typical chart structure:

Chart.yaml
values.yaml
templates/
├── deployment.yaml
├── service.yaml
└── ingress.yaml

The reusable chart contains defaults while environment files override
them:

flowchart LR
    A["Helm Chart<br/>Reusable Defaults"] --> B["DEV Values"]
    A --> C["STAGING Values"]
    A --> D["PRODUCTION Values"]

    B --> E["Dev Release"]
    C --> F["Staging Release"]
    D --> G["Production Release"]

This avoids copying complete Kubernetes manifests for every environment.

🔁 GitOps Promotion

Promotion is controlled through Git.

flowchart LR
    A["Jenkins"] --> B["GitOps PR"]
    B --> C["Review"]
    C --> D["Merge"]
    D --> E["Argo CD"]
    E --> F["Environment"]

The actual promotion chain completed during the project was:

PR #4 → 07eda39 → DEV
      ↓
PR #5 → 07eda39 → STAGING
      ↓
PR #6 → Staging configuration fix
      ↓
PR #7 → 07eda39 → PRODUCTION

The Production PR was the final Git-based approval gate.

🤖 Argo CD

Argo CD continuously compares:

Git Desired State
        ↕
Kubernetes Actual State

Architecture:

flowchart LR
    A["GitOps Repository"] --> B["Argo CD"]
    B --> C["Compare"]
    C --> D["Sync"]
    D --> E["Kubernetes"]
    E --> F["Actual State"]
    F --> C

Applications:

microservices-dev
microservices-staging
microservices-prod

Each environment is treated as the deployment boundary for the three
services.

❤️ Argo CD Self-Healing

The Applications use:

syncPolicy:
  automated:
    prune: true
    selfHeal: true

Example:

Git:
replicas = 1

Kubernetes:
replicas = 3

Argo CD detects the drift and reconciles Kubernetes back toward Git.

Self-healing was demonstrated during the project.

🔑 ECR Authentication in K3s

K3s uses containerd, not the host Docker Engine.

Therefore:

Docker image store
        ≠
K3s/containerd image store

An image visible with:

docker images

is not automatically available to K3s.

The registry-based deployment uses:

imagePullSecrets:
  - name: ecr-registry-secret

ECR token incident

During deployment, a registry token expired and Kubernetes reported:

403 Forbidden
denied: Your authorization token has expired.

Refreshing the ECR registry secret resolved the problem.

For a hardened production platform, automatic ECR credential
management should replace manually refreshed registry secrets.

🏗️ Terraform + Terragrunt

Infrastructure is managed separately from application delivery.

flowchart LR
    A["Terraform Modules"] --> B["Terragrunt"]
    B --> C["AWS"]

    C --> D["VPC"]
    C --> E["IAM"]
    C --> F["ECR"]
    C --> G["Kubernetes Infrastructure"]

    B --> H["S3 Remote State"]

Terraform

Terraform is the infrastructure provisioning engine.

It manages resources such as:

VPC

IAM

ECR

Kubernetes infrastructure

Terragrunt

Terragrunt provides orchestration and configuration around Terraform:

DRY configuration

Environment organization

Remote state configuration

Module reuse

Dependencies

Multi-unit execution

Interview answer

Terraform defines how infrastructure is created. Terragrunt helps
organize, configure, and orchestrate Terraform across environments and
infrastructure units.

🪣 Terraform Remote State

Terraform state is stored in Amazon S3.

Terraform
    │
    ▼
Amazon S3
    ├── Encryption
    ├── Versioning
    └── Native State Locking

The remote-state configuration uses:

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

🥚 Why the S3 Backend Was Bootstrapped Separately

Terraform needs the S3 bucket in order to store remote state.

Therefore:

Terraform needs S3
       ↓
S3 stores Terraform state
       ↓
Bucket must exist first

A small bootstrap Terraform configuration created the bucket before the
normal remote backend was used.

The bucket was configured with:

S3 versioning

AES256 encryption

Public access blocking

🛠️ Real Troubleshooting Incidents

These incidents are part of the value of this project.

1. ECR Immutable Tag Failure

Symptom

The image tag '1.1.0' already exists
and cannot be overwritten because the tag is immutable.

Root cause

ECR was correctly preventing an existing artifact tag from being
overwritten.

Fix

Changed Jenkins to use Git SHA tags:

07eda39

Lesson

Never design a CI/CD pipeline around overwriting immutable production
artifacts.

2. K3s Could Not Use Local Docker Image

Root cause

Docker Engine and K3s/containerd maintain separate image stores.

Fix

Move to registry-based deployment using ECR.

Lesson

Container images exist inside a runtime’s image store or registry;
Docker Engine and Kubernetes’ container runtime are not automatically
the same image store.

3. ECR Authentication Expired

Symptom

403 Forbidden
denied: Your authorization token has expired.

Root cause

The ECR authorization token used by the Kubernetes registry secret had
expired.

Fix

Refresh the registry secret.

Production improvement

Use automatic credential management rather than manually refreshed
tokens.

4. Staging Backend ImagePullBackOff

Symptom

frontend          → Running
product-service   → ImagePullBackOff
order-service     → ImagePullBackOff

Root cause

The staging product-values.yaml and order-values.yaml files were
empty.

Helm therefore fell back to chart defaults instead of the intended ECR
repositories.

Fix

Configured:

image:
  repository: <ECR repository>
  tag: "07eda39"

imagePullSecrets:
  - name: ecr-registry-secret

Then:

Commit
  ↓
PR
  ↓
Merge
  ↓
Argo CD
  ↓
Staging Healthy

Lesson

Configuration is part of the deployment artifact. GitOps makes
configuration changes visible, reviewable and auditable.

5. Terraform IAM AccessDenied

Root cause

The EC2 IAM role did not initially have every S3 API permission
Terraform required.

Troubleshooting approach

Terraform error
      ↓
Identify denied AWS API
      ↓
Add specific permission
      ↓
terraform plan
      ↓
Review
      ↓
terraform apply

Lesson

Prefer least privilege and troubleshoot the exact AWS API action
instead of immediately granting AdministratorAccess.

6. Terraform Tainted Resource

Symptom

Terraform proposed:

-/+ destroy and recreate

for the existing S3 bucket.

Action

The destructive plan was not applied.

The resource was safely untainted:

terraform untaint aws_s3_bucket.terraform_state

Then the plan was reviewed again.

Lesson

Always investigate Terraform destroy/replacement actions before
applying a plan.

↩️ Rollback Strategy

Helm

Inspect release history:

helm history frontend -n microservices-dev

Helm can roll back to a known-good release revision.

GitOps

For GitOps deployments, a preferred rollback approach is often:

Bad GitOps Commit
       ↓
Git Revert
       ↓
Argo CD
       ↓
Known-Good Kubernetes State

This preserves an auditable change history.

🧪 Testing and Verification

The project used three levels of validation.

1. External routing

curl http://localhost/

Validates:

EC2
 → Traefik
 → frontend Service
 → frontend Pod

2. Internal service connectivity

A temporary curl pod validated:

frontend/network
 → product-service:8081
 → order-service:8082

3. End-to-end application flow

curl http://localhost/products
curl http://localhost/orders

This validates:

Client
  ↓
Traefik
  ↓
Frontend
  ↓
Backend Services
  ↓
Backend Pods

🔎 Useful Troubleshooting Commands

Kubernetes

kubectl get pods -n microservices-dev
kubectl get pods -n microservices-staging
kubectl get pods -n microservices-prod

kubectl get svc -n microservices-dev
kubectl get ingress -n microservices-dev

Pod details

kubectl describe pod <pod-name> -n <namespace>

Logs

kubectl logs <pod-name> -n <namespace>

Helm

helm ls -n microservices-dev
helm history frontend -n microservices-dev

ECR

aws ecr describe-repositories --region us-east-1

aws ecr list-images \
  --repository-name frontend \
  --region us-east-1

Terraform

terraform plan
terraform apply
terraform state list

Terragrunt

terragrunt init
terragrunt plan
terragrunt apply

🔒 Security Considerations

This is a lab implementation, but the following production principles
apply.

IAM

Use least privilege instead of:

AdministratorAccess

ECR

Use:

Immutable tags
Scan on push
Encryption

Secrets

Never commit:

Passwords
Tokens
AWS credentials
Private keys

Jenkins

Adding Jenkins to the Docker group effectively gives Jenkins
root-equivalent control over the host.

Acceptable for this lab; a major security consideration in production.

Kubernetes

A hardened platform should also consider:

RBAC

NetworkPolicies

Pod Security Standards

Resource requests/limits

Horizontal Pod Autoscaling

PodDisruptionBudgets

Secrets management

TLS

Image signing

Vulnerability gates

Workload identity

🚀 Production Improvements

If this lab were expanded into a real production platform:

Separate AWS accounts per environment

Separate Kubernetes clusters where appropriate

Automatic ECR credential management

Production ingress/load balancer architecture

TLS through ACM or cert-manager

AWS Secrets Manager / External Secrets

Kubernetes NetworkPolicies

CPU/memory requests and limits

Horizontal Pod Autoscaling

PodDisruptionBudgets

Trivy or equivalent vulnerability gates

Image signing and verification

Jenkins ephemeral agents

Centralized logging

Prometheus/Grafana monitoring

Distributed tracing

Argo CD Projects and RBAC

Required GitHub reviewers for production

Jira integration

Automated deployment notifications

📋 Jira / Change Management

Jira can sit above the Git workflow as the work-management layer.

flowchart LR
    A["Jira Ticket"] --> B["Feature Branch"]
    B --> C["Pull Request"]
    C --> D["Jenkins CI"]
    D --> E["GitOps PR"]
    E --> F["Production Approval"]
    F --> G["Argo CD"]

Jira is not responsible for running Terraform.

Its value is:

Work tracking

Change management

Traceability

Linking requirements to branches/commits/PRs

Deployment/change visibility

GitHub can remain the source-control platform.

🎤 Senior DevOps Interview Answer

A strong concise explanation:

I designed a GitOps-based Dev-to-Production promotion workflow using
Jenkins, Docker, Amazon ECR, Helm, Kubernetes/K3s, and Argo CD.
Jenkins handles CI by checking out the source, running tests, building
the three microservice images, tagging them with the Git commit SHA,
and pushing immutable images to ECR. It then creates a PR against a
separate GitOps repository. Environment-specific Helm values determine
which immutable artifact each environment should run. Promotion from
Dev to Staging to Production happens through pull requests, providing
an explicit approval and audit boundary. Argo CD watches the GitOps
repository and reconciles the desired state into Kubernetes, with
automated sync and self-healing enabled. Infrastructure is managed
separately with Terraform modules and Terragrunt, using an encrypted,
versioned S3 remote state backend with locking.

💬 Interview Questions You Should Be Ready For

<details>
<summary>
<strong>Why use Git SHA tags instead of latest?</strong>
</summary>
Git SHA tags are immutable and traceable to a specific source revision.
`latest` is mutable and makes reproducibility, auditing and rollback
harder.
</details>
<details>
<summary>
<strong>Why did ECR reject the 1.1.0 push?</strong>
</summary>
The repository was configured with immutable tags, so an existing
`1.1.0` tag could not be overwritten.
</details>
<details>
<summary>
<strong>Why use a separate GitOps repository?</strong>
</summary>
It separates application source code from deployment configuration and
establishes a clear desired-state repository for environments.
</details>
<details>
<summary>
<strong>Why doesn’t Jenkins run kubectl apply?</strong>
</summary>
Argo CD owns deployment and reconciliation. Jenkins creates and
publishes artifacts and proposes GitOps changes rather than requiring
direct cluster deployment privileges.
</details>
<details>
<summary>
<strong>What happens if someone manually changes Production?</strong>
</summary>
Argo CD detects the drift and, because self-healing is enabled,
reconciles the cluster back to the state defined in Git.
</details>
<details>
<summary>
<strong>Why use Helm?</strong>
</summary>
Helm provides reusable Kubernetes templates while allowing
environment-specific configuration through values files.
</details>
<details>
<summary>
<strong>Terraform vs Terragrunt?</strong>
</summary>
Terraform provisions infrastructure. Terragrunt organizes and
orchestrates Terraform configurations, environments, dependencies and
shared configuration.
</details>
<details>
<summary>
<strong>Why use remote Terraform state?</strong>
</summary>
Remote state provides centralized, durable and shareable state while
supporting encryption, versioning and locking.
</details>
<details>
<summary>
<strong>What do you do when Terraform wants to destroy a critical
resource?</strong>
</summary>
Stop and investigate the plan. Identify why Terraform believes
replacement is required and never blindly apply a destructive plan.
</details>
<details>
<summary>
<strong>How would you improve the lab for production?</strong>
</summary>
I would add stronger environment isolation, automatic registry
authentication, secrets management, TLS, network policies, resource
controls, vulnerability scanning, image signing, observability, RBAC and
controlled production approvals.
</details>

🏆 What This Project Demonstrates

This project is intentionally more than:

“I know Docker, Kubernetes and Jenkins.”

It demonstrates understanding of the complete software delivery
lifecycle:

SOURCE
  ↓
CI
  ↓
ARTIFACT
  ↓
REGISTRY
  ↓
GITOPS
  ↓
APPROVAL
  ↓
DEPLOYMENT
  ↓
RECONCILIATION
  ↓
KUBERNETES
  ↓
OPERATIONS

Clear ownership:

GitHub Application Repo
        ↓
      Jenkins
        ↓
       ECR
        ↓
GitHub GitOps Repo
        ↓
      Argo CD
        ↓
    Kubernetes

🥇 Final Promotion Result

The exact same immutable artifact successfully progressed through all
environments:

flowchart LR
    A["Git Commit<br/><b>07eda39</b>"]
    B["Jenkins<br/>Build Once"]
    C["Amazon ECR<br/>Immutable Images"]
    D["DEV<br/>✅ Healthy"]
    E["STAGING<br/>✅ Healthy"]
    F["PRODUCTION<br/>✅ Healthy"]

    A --> B --> C --> D --> E --> F

Final Production State

Component

Status

Application source

✅ Complete

Docker

✅ Complete

Amazon ECR

✅ Complete

Helm

✅ Complete

GitOps

✅ Complete

Jenkins CI

✅ Complete

Argo CD CD

✅ Complete

Dev promotion

✅ Healthy

Staging promotion

✅ Healthy

Production promotion

✅ Healthy

Terraform/Terragrunt

✅ Complete

Production Argo Application

🟢 Healthy

Production GitOps state

🟢 Synced

🧠 Ten Key Takeaways

If you remember only ten things:

Build once, promote the same artifact.

Use immutable image tags.

Git SHA is a useful immutable image identifier.

Jenkins builds and publishes; Argo CD deploys and reconciles.

GitOps makes Git the desired-state source of truth.

Pull requests can serve as deployment approval gates.

Helm separates reusable templates from environment-specific
values.

Terraform provisions infrastructure; Terragrunt organizes
Terraform.

Always investigate Terraform destroy/replacement plans.

Troubleshooting is part of DevOps engineering—not a failure of the
design.

🔗 Project Repositories

Repository

Purpose

Application

Source code + Jenkins CI

GitOps

Helm + environment values + Argo CD

Infrastructure

Terraform + Terragrunt

Platform Overview

Architecture + complete project story

📊 Project Status

Application CI: ✅ Complete
Docker: ✅ Complete
Amazon ECR: ✅ Complete
Helm: ✅ Complete
GitOps: ✅ Complete
Jenkins CI: ✅ Complete
Argo CD CD: ✅ Complete
Dev: ✅ Healthy
Staging: ✅ Healthy
Production: ✅ Healthy
Terraform/Terragrunt: ✅ Complete

Built as a hands-on Senior DevOps portfolio project focused on
CI/CD, GitOps, Kubernetes, AWS, Infrastructure as Code, promotion
strategies, and real-world troubleshooting.
