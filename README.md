<div align="center">

🚀 Dev-to-Prod Promotion Platform

Production-Style CI/CD, GitOps & Kubernetes Promotion










A production-style DevOps platform demonstrating how application code is built once, packaged as immutable artifacts, and promoted through Dev → Staging → Production using GitOps.

</div>

1. Project Overview

This project demonstrates how a DevOps team can implement a controlled application promotion workflow where changes are:

developed and versioned in GitHub

built and tested by Jenkins

packaged as Docker images

tagged with an immutable Git commit SHA

pushed to Amazon ECR

promoted through a separate GitOps repository

reviewed through Pull Requests

deployed by Argo CD

reconciled continuously against Git

verified in Kubernetes Dev, Staging and Production environments

Infrastructure is managed separately using Terraform + Terragrunt, with Terraform state stored remotely in Amazon S3.

The project intentionally includes real failure scenarios and troubleshooting rather than documenting only the successful path.

2. What This Project Demonstrates

The implementation combines:

GitHub — source control and Pull Requests

Jenkins — CI and artifact creation

Docker — containerization

Amazon ECR — immutable container registry

Kubernetes / K3s — application runtime

Helm — Kubernetes packaging and templating

Traefik — Ingress Controller

Argo CD — GitOps continuous delivery

Terraform — Infrastructure as Code

Terragrunt — Terraform orchestration

Amazon S3 — remote Terraform state

AWS IAM — identity and authorization

The central engineering principle is:

Build the application once, create an immutable artifact, and promote that exact artifact through each environment.

3. Architecture

3.1 Complete Application Delivery Architecture

                         ┌─────────────────┐
                         │    Developer    │
                         └────────┬────────┘
                                  │
                                  │ Source Code
                                  ▼
                         ┌─────────────────┐
                         │     GitHub      │
                         │  Application    │
                         │      Repo       │
                         └────────┬────────┘
                                  │
                                  │ CI
                                  ▼
                         ┌─────────────────┐
                         │     Jenkins     │
                         │                 │
                         │ Checkout        │
                         │ Test            │
                         │ Docker Build    │
                         │ Git SHA Tag     │
                         │ Push to ECR     │
                         └────────┬────────┘
                                  │
                                  │ Immutable Images
                                  ▼
                         ┌─────────────────┐
                         │    Amazon ECR   │
                         │                 │
                         │ frontend        │
                         │ product-service │
                         │ order-service   │
                         └────────┬────────┘
                                  │
                                  │ Image Reference
                                  ▼
                         ┌─────────────────┐
                         │ GitHub GitOps   │
                         │      Repo       │
                         │                 │
                         │ Helm Charts     │
                         │ Env Values      │
                         │ Argo CD Apps    │
                         └────────┬────────┘
                                  │
                             PR → Review
                             → Merge
                                  │
                                  ▼
                         ┌─────────────────┐
                         │     Argo CD     │
                         │                 │
                         │ Sync            │
                         │ Reconcile       │
                         │ Self-Heal       │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │          K3s Cluster        │
                    │                             │
                    │  ┌───────────────────────┐  │
                    │  │ microservices-dev     │  │
                    │  └───────────────────────┘  │
                    │                             │
                    │  ┌───────────────────────┐  │
                    │  │ microservices-staging │  │
                    │  └───────────────────────┘  │
                    │                             │
                    │  ┌───────────────────────┐  │
                    │  │ microservices-prod    │  │
                    │  └───────────────────────┘  │
                    │                             │
                    │       Traefik Ingress       │
                    └─────────────────────────────┘

3.2 Infrastructure Architecture

                  ┌─────────────────────┐
                  │   Terraform Modules │
                  │                     │
                  │ VPC │ IAM │ ECR     │
                  │ EKS │ etc.          │
                  └──────────┬──────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Terragrunt   │
                    │                 │
                    │ Environment     │
                    │ Organization    │
                    │ Remote State    │
                    │ Dependencies    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ AWS Infrastructure
                    └─────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
            VPC             IAM            ECR

                    ┌─────────────────┐
                    │   Amazon S3     │
                    │ Terraform State │
                    │                 │
                    │ Encryption      │
                    │ Versioning      │
                    │ State Locking   │
                    └─────────────────┘

4. Repository Structure

The platform is intentionally separated into three implementation repositories plus this umbrella project repository.

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

5. Microservices

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

The frontend communicates with the backend services through Kubernetes DNS:

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

6. CI/CD Pipeline

The complete delivery workflow is:

Developer
    │
    ▼
GitHub Application Repo
    │
    ▼
Jenkins
    │
    ├── Checkout
    ├── Test
    ├── Docker Build
    ├── Generate Git SHA
    └── Push Images
    │
    ▼
Amazon ECR
    │
    ▼
GitOps Pull Request
    │
    ▼
GitOps Repository
    │
    ▼
Argo CD
    │
    ▼
Kubernetes

Responsibility boundaries

Component

Responsibility

GitHub Application Repo

Source code

Jenkins

CI, testing, image build and publishing

Amazon ECR

Immutable artifact storage

GitHub GitOps Repo

Desired deployment state

Pull Request

Promotion/change approval

Argo CD

Deployment and reconciliation

Kubernetes

Runtime

7. Immutable Image Strategy

The ECR repositories use immutable tags.

The pipeline initially attempted to reuse:

frontend:1.1.0

ECR rejected the push because that tag already existed.

The pipeline was therefore changed to derive the image tag from the Git commit:

env.IMAGE_TAG = sh(
    script: 'git rev-parse --short=7 HEAD',
    returnStdout: true
).trim()

The resulting promotion artifact was:

07eda39

The three application images were therefore:

frontend:07eda39
product-service:07eda39
order-service:07eda39

Why Git SHA tags?

Git SHA tags provide:

Immutability

Traceability

Reproducibility

Auditability

Rollback capability

The image can be traced directly to the source revision that produced it.

8. Build Once, Promote the Same Artifact

This is one of the most important design principles in the project.

                    07eda39
                       │
                       ▼
                 ┌──────────┐
                 │  Jenkins │
                 │ Build    │
                 └────┬─────┘
                      │
                      ▼
                ┌───────────┐
                │    ECR    │
                └─────┬─────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        DEV        STAGING       PROD
         ✅           ✅           ✅

❌ Avoid

Build → Dev
Build → Staging
Build → Production

✅ Preferred

Build Once
    ↓
Immutable Artifact
    ↓
Promote Same Artifact

This prevents the artifact tested in Staging from being different from the artifact deployed to Production.

9. GitOps Promotion Model

Promotion is controlled through Git Pull Requests.

Jenkins
   │
   ▼
Create GitOps PR
   │
   ▼
Review
   │
   ▼
Merge
   │
   ▼
Argo CD
   │
   ▼
Kubernetes Environment

The completed promotion chain was:

PR #4
07eda39
    ↓
DEV
    ↓
PR #5
07eda39
    ↓
STAGING
    ↓
PR #6
Staging configuration fix
    ↓
STAGING HEALTHY
    ↓
PR #7
07eda39
    ↓
PRODUCTION

The Production Pull Request served as the final approval gate.

10. Kubernetes Environment Architecture

The lab uses a single K3s cluster with namespace-based environment separation.

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

This keeps the lab lightweight while preserving the environment promotion model.

In a larger production platform, environments may instead use:

separate Kubernetes clusters

separate AWS accounts

separate VPCs

stronger network isolation

Important: Kubernetes namespaces provide environment boundaries in this lab, but should not be described as equivalent to separate AWS accounts or clusters.

11. Kubernetes Networking

The external frontend request path is:

Browser / curl
      │
      ▼
EC2 :80
      │
      ▼
Traefik Ingress Controller
      │
      ▼
frontend Service :8080
      │
      ▼
frontend Pod :8080

Internal application traffic:

frontend Pod
      │
      ├── product-service:8081
      │
      └── order-service:8082

Important distinction

EC2 port 80 — external entry point exposed by Traefik

Frontend port 8080 — application container port

Kubernetes Service — stable internal network endpoint

Ingress — Kubernetes resource defining HTTP routing

Traefik — Ingress Controller implementing those routing rules

12. Helm

Each microservice has its own Helm chart:

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

The chart provides reusable defaults while environment-specific values provide overrides.

                    Helm Chart
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          DEV Values  STAGING    PROD
             │          │          │
             ▼          ▼          ▼
           Release    Release    Release

This avoids duplicating complete Kubernetes manifests for every environment.

13. Argo CD

Argo CD continuously compares:

Git Desired State
        │
        │ compare
        ▼
Kubernetes Actual State

Conceptually:

GitOps Repository
       │
       ▼
    Argo CD
       │
       ├── Compare
       ├── Sync
       └── Self-Heal
       │
       ▼
   Kubernetes

The Argo CD Applications are:

microservices-dev
microservices-staging
microservices-prod

Each Application represents an environment-level deployment boundary.

14. Argo CD Self-Healing

The Applications use automated synchronization with:

syncPolicy:
  automated:
    prune: true
    selfHeal: true

Example:

Git:
replicas = 1

Kubernetes:
replicas = 3

Argo CD detects the drift and reconciles the cluster back toward the desired state in Git.

Self-healing was demonstrated during the project.

15. ECR Authentication in K3s

K3s uses containerd, while the host Docker Engine has its own image store.

Therefore:

Docker Engine image store
          ≠
K3s/containerd image store

An image visible with:

docker images

is not automatically available to K3s.

The final deployment model uses Amazon ECR and:

imagePullSecrets:
  - name: ecr-registry-secret

ECR token incident

During deployment, Kubernetes reported:

403 Forbidden
denied: Your authorization token has expired.

The registry secret contained an expired ECR authorization token.

Refreshing the secret restored the deployment.

Production improvement: use automatic ECR credential management rather than manually refreshed registry secrets.

16. Terraform + Terragrunt

Infrastructure management is intentionally separated from application delivery.

Terraform
   │
   │ Infrastructure as Code
   ▼
Terraform Modules
   │
   ▼
Terragrunt
   │
   │ Organization / orchestration
   ▼
AWS Infrastructure

Terraform

Terraform is the infrastructure provisioning engine.

It manages resources such as:

VPC

IAM

ECR

Kubernetes infrastructure

Terragrunt

Terragrunt provides an orchestration/configuration layer around Terraform.

It helps with:

DRY configuration

environment organization

remote state configuration

module reuse

dependencies

multi-unit execution

Interview answer

Terraform defines how infrastructure is created. Terragrunt helps organize, configure and orchestrate Terraform across environments and infrastructure units.

17. Terraform Remote State

Terraform state is stored remotely in Amazon S3.

                 Terraform
                     │
                     ▼
              Amazon S3 Bucket
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Encryption    Versioning    Locking

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

18. Why the S3 Backend Was Bootstrapped

Terraform needs the S3 bucket in order to store remote state.

That creates a dependency:

Terraform needs S3
       ↓
S3 stores Terraform state
       ↓
The bucket must exist first

A small bootstrap Terraform configuration created the state bucket before the normal remote backend was used.

The bucket was configured with:

S3 versioning

AES256 server-side encryption

public-access blocking

19. Real Troubleshooting Scenarios

The project intentionally documents real failures encountered during implementation.

19.1 ECR Immutable Tag Failure

Symptom

The image tag '1.1.0' already exists
and cannot be overwritten because the tag is immutable.

Root Cause

ECR was correctly preventing an existing immutable artifact from being overwritten.

Resolution

Changed Jenkins to use Git SHA image tags:

07eda39

Lesson

CI/CD pipelines should never depend on overwriting immutable release artifacts.

19.2 K3s Could Not Use the Local Docker Image

Root Cause

Docker Engine and K3s/containerd use separate image stores.

Resolution

Moved to registry-based deployment using Amazon ECR.

Lesson

Kubernetes does not automatically share the host Docker Engine's image cache.

19.3 ECR Authentication Token Expired

Symptom

403 Forbidden
denied: Your authorization token has expired.

Root Cause

The ECR authorization token stored in the Kubernetes registry secret had expired.

Resolution

The registry secret was refreshed.

Production Improvement

Use automatic ECR credential management.

19.4 Staging Backend ImagePullBackOff

Symptom

frontend          → Running
product-service   → ImagePullBackOff
order-service     → ImagePullBackOff

Root Cause

The Staging product-values.yaml and order-values.yaml files were empty.

Helm therefore fell back to chart defaults instead of the intended ECR repositories.

Resolution

The missing configuration was added through Git:

Fix values
    ↓
Commit
    ↓
Pull Request
    ↓
Merge
    ↓
Argo CD reconciliation
    ↓
Staging Healthy

Lesson

Configuration is part of the deployment system. GitOps makes configuration changes reviewable, traceable and auditable.

19.5 Terraform IAM AccessDenied

Root Cause

The EC2 IAM role initially lacked some S3 permissions required by Terraform.

Troubleshooting approach

Terraform error
      ↓
Identify denied AWS API action
      ↓
Update IAM policy
      ↓
terraform plan
      ↓
Review
      ↓
terraform apply

Lesson

Prefer least privilege and identify the exact missing AWS API permission instead of immediately granting broad administrative access.

19.6 Terraform Tainted Resource

Symptom

Terraform proposed:

-/+ destroy and recreate

for an existing S3 bucket.

Action

The destructive plan was not applied.

The resource was safely untainted:

terraform untaint aws_s3_bucket.terraform_state

The plan was then reviewed again.

Lesson

Always investigate Terraform destroy/replacement actions before applying a plan.

20. Testing and Verification

The application was tested at multiple layers.

External routing

curl http://localhost/

Validates:

EC2
 → Traefik
 → frontend Service
 → frontend Pod

Internal service connectivity

A temporary curl pod was used to validate:

frontend/network
 → product-service:8081
 → order-service:8082

End-to-end flow

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

21. Rollback Strategy

Helm Rollback

Inspect release history:

helm history frontend -n microservices-dev

Helm can roll back to a known-good release revision.

GitOps Rollback

For GitOps-managed deployments, a preferred rollback mechanism is often:

Bad GitOps Commit
       ↓
Git Revert
       ↓
Argo CD
       ↓
Known-Good State

This keeps the rollback auditable in Git.

22. Security Considerations

This is a lab implementation, but the production principles are important.

IAM

Use:

Least Privilege

instead of:

AdministratorAccess

ECR

Use:

Immutable Tags
Scan on Push
Encryption

Secrets

Never commit:

Passwords
Tokens
AWS Credentials
Private Keys

Jenkins

Adding Jenkins to the Docker group provides effectively root-equivalent control over the host.

Acceptable for this lab, but a major security consideration in production.

Kubernetes

A hardened platform should also consider:

RBAC

NetworkPolicies

Pod Security Standards

Resource requests and limits

Horizontal Pod Autoscaling

PodDisruptionBudgets

Secrets management

TLS

Image signing

Vulnerability gates

Workload identity

23. Production Improvements

A real production implementation could add:

Separate AWS accounts per environment

Separate Kubernetes clusters where appropriate

Automatic ECR credential management

Production load balancer / ingress architecture

TLS through ACM or cert-manager

AWS Secrets Manager / External Secrets

Kubernetes NetworkPolicies

CPU and memory requests/limits

Horizontal Pod Autoscaling

PodDisruptionBudgets

Trivy or equivalent vulnerability gates

Image signing and verification

Jenkins ephemeral agents

Centralized logging

Prometheus / Grafana monitoring

Distributed tracing

Argo CD Projects and RBAC

Required GitHub reviewers for Production

Jira integration

Automated deployment notifications

24. Jira / Change Management

Jira can sit above the Git workflow as the work-management layer.

Jira Ticket
     │
     ▼
Feature Branch
     │
     ▼
Pull Request
     │
     ▼
Jenkins CI
     │
     ▼
GitOps PR
     │
     ▼
Production Approval
     │
     ▼
Argo CD

Jira is not responsible for running Terraform.

Its value is:

work tracking

change management

traceability

linking requirements to branches, commits and PRs

deployment/change visibility

GitHub can remain the source-control platform.

25. Senior DevOps Interview Explanation

A strong interview answer:

I designed a GitOps-based Dev-to-Production promotion workflow using Jenkins, Docker, Amazon ECR, Helm, Kubernetes/K3s and Argo CD. Jenkins handles CI by checking out the source, running tests, building the three microservice images, tagging them with the Git commit SHA and pushing immutable images to ECR. Jenkins then creates a Pull Request against a separate GitOps repository. Environment-specific Helm values determine which immutable artifact each environment should run. Promotion from Dev to Staging to Production happens through Pull Requests, providing an explicit approval and audit boundary. Argo CD watches the GitOps repository and reconciles the desired state into Kubernetes, with automated sync and self-healing enabled. Infrastructure is managed separately using Terraform modules and Terragrunt, with encrypted, versioned S3 remote state and locking.

26. Senior DevOps Interview Questions

<details>
<summary><strong>Why use Git SHA tags instead of latest?</strong></summary>

Git SHA tags are immutable and traceable to a specific source revision. latest is mutable and makes reproducibility, auditing and rollback harder.

</details>

<br>

<details>
<summary><strong>Why did ECR reject the 1.1.0 push?</strong></summary>

The ECR repository was configured with immutable tags, so an existing 1.1.0 tag could not be overwritten.

</details>

<br>

<details>
<summary><strong>Why use a separate GitOps repository?</strong></summary>

It separates application source code from deployment configuration and provides a clear desired-state repository for environments.

</details>

<br>

<details>
<summary><strong>Why doesn't Jenkins run kubectl apply?</strong></summary>

Argo CD owns deployment and reconciliation. Jenkins creates and publishes artifacts and proposes GitOps changes rather than requiring direct deployment access to the cluster.

</details>

<br>

<details>
<summary><strong>What happens if someone manually changes Production?</strong></summary>

Argo CD detects the drift and, because self-healing is enabled, reconciles the cluster back to the state defined in Git.

</details>

<br>

<details>
<summary><strong>Why use Helm?</strong></summary>

Helm provides reusable Kubernetes templates while allowing environment-specific configuration through values files.

</details>

<br>

<details>
<summary><strong>Terraform vs Terragrunt?</strong></summary>

Terraform provisions infrastructure. Terragrunt organizes and orchestrates Terraform configurations, environments, dependencies and shared configuration.

</details>

<br>

<details>
<summary><strong>Why use remote Terraform state?</strong></summary>

Remote state provides centralized, durable and shareable state while supporting encryption, versioning and locking.

</details>

<br>

<details>
<summary><strong>What should you do when Terraform wants to destroy a critical resource?</strong></summary>

Stop and investigate the plan. Identify why Terraform believes replacement is required and never blindly apply a destructive plan.

</details>

<br>

<details>
<summary><strong>How would you improve this platform for production?</strong></summary>

I would add stronger environment isolation, automatic registry authentication, secrets management, TLS, network policies, resource controls, vulnerability scanning, image signing, observability, RBAC and controlled Production approvals.

</details>

27. Final Promotion Result

The same immutable artifact successfully progressed through every environment:

                         Git Commit
                           07eda39
                              │
                              ▼
                         ┌─────────┐
                         │ Jenkins │
                         │ Build   │
                         └────┬────┘
                              │
                              ▼
                           Amazon ECR
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
              frontend     product       order
              07eda39      07eda39      07eda39
                 │            │            │
                 └────────────┼────────────┘
                              │
                              ▼
                        GitOps Promotion
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
              DEV          STAGING         PROD
                │             │             │
                ▼             ▼             ▼
            🟢 Healthy    🟢 Healthy    🟢 Healthy
            🟢 Synced     🟢 Synced     🟢 Synced

Final Production State

Component

Status

Application source

🟢 Complete

Docker

🟢 Complete

Amazon ECR

🟢 Complete

Helm

🟢 Complete

GitOps

🟢 Complete

Jenkins CI

🟢 Complete

Argo CD CD

🟢 Complete

Dev

🟢 Healthy

Staging

🟢 Healthy

Production

🟢 Healthy

Terraform / Terragrunt

🟢 Complete

Production Argo Application

🟢 Healthy

Production GitOps state

🟢 Synced

28. Key Takeaways

If you remember only ten things:

Build once, promote the same artifact.

Use immutable image tags.

Git SHA is a useful immutable image identifier.

Jenkins builds and publishes; Argo CD deploys and reconciles.

GitOps makes Git the desired-state source of truth.

Pull Requests can serve as deployment approval gates.

Helm separates reusable templates from environment-specific values.

Terraform provisions infrastructure; Terragrunt organizes Terraform.

Always investigate Terraform destroy/replacement plans.

Troubleshooting is part of DevOps engineering — not a failure of the design.

29. Project Repositories

Repository

Purpose

Application

Source code + Jenkins CI

GitOps

Helm + environment values + Argo CD

Infrastructure

Terraform + Terragrunt

Platform Overview

Complete architecture and project story

<div align="center">

🏆 Project Status

Application CI ✅ · Docker ✅ · Amazon ECR ✅ · Helm ✅ · GitOps ✅ · Jenkins ✅ · Argo CD ✅ · Dev ✅ · Staging ✅ · Production ✅ · Terraform/Terragrunt ✅

<br>

Built as a hands-on Senior DevOps portfolio project focused on CI/CD, GitOps, Kubernetes, AWS, Infrastructure as Code, promotion strategies, and real-world troubleshooting.

</div>
