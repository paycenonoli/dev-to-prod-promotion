Dev-to-Prod Promotion Platform

A hands-on, production-style DevOps project demonstrating how
application code moves from source control → CI → container registry →
GitOps → Kubernetes Dev → Staging → Production, with infrastructure
managed separately through Terraform + Terragrunt.

The project uses one K3s cluster with separate Kubernetes namespaces for
dev, staging, and prod. This keeps the lab lightweight while
preserving the important promotion, approval, artifact immutability, and
reconciliation patterns used in larger environments.

1. Project Goals

This project demonstrates:

Git-based application development

Docker containerization

Jenkins CI/CD

Immutable container image tagging

Amazon ECR

Helm packaging

Environment-specific Helm values

GitOps with a separate repository

Pull-request based environment promotion

Argo CD continuous delivery

Argo CD automated sync and self-healing

Kubernetes namespaces as environment boundaries

Traefik Ingress

Terraform infrastructure provisioning

Terragrunt orchestration

Remote Terraform state in Amazon S3

S3 versioning, encryption, and native state locking

IAM troubleshooting and least privilege

Production-style troubleshooting and incident recovery

2. High-Level Architecture

                         APPLICATION DELIVERY

┌──────────────────────┐
│      Developer       │
│  Source Code Change  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│       GitHub         │
│  Application Repo    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│       Jenkins        │
│                      │
│  Checkout            │
│  Test                │
│  Docker Build        │
│  Git SHA Tag         │
│  Push to ECR         │
│  Create GitOps PR    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      Amazon ECR      │
│                      │
│ frontend:07eda39     │
│ product-service:...  │
│ order-service:...    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────┐
│       GitHub GitOps Repo     │
│                              │
│ Helm charts                  │
│ Environment values           │
│ Argo CD Applications         │
└──────────────┬───────────────┘
               │
               │ PR → review → merge
               ▼
        ┌───────────────┐
        │    Argo CD    │
        │ Sync + Heal   │
        └───────┬───────┘
                │
                ▼
┌──────────────────────────────────────────────┐
│                 K3s Cluster                  │
│                                              │
│  microservices-dev       ← DEV              │
│  microservices-staging   ← STAGING          │
│  microservices-prod      ← PRODUCTION       │
│                                              │
│  Traefik Ingress Controller                  │
└──────────────────────────────────────────────┘

3. Infrastructure Architecture

Application delivery and infrastructure management are separate
concerns.

Terraform Modules
       │
       ▼
   Terragrunt
       │
       ▼
 AWS Infrastructure
       │
       ├── VPC
       ├── IAM
       ├── ECR
       └── Kubernetes infrastructure

Terraform vs Terragrunt

Terraform is the infrastructure provisioning engine. It creates and
manages resources such as VPCs, IAM, ECR, and Kubernetes infrastructure.

Terragrunt is the orchestration/configuration layer around
Terraform. It helps with:

DRY configuration

Environment organization

Remote state configuration

Module reuse

Dependencies

Running infrastructure units safely

Interview answer:

Terraform defines how infrastructure is created. Terragrunt helps
organize, configure, and orchestrate Terraform across environments and
infrastructure units.

4. Repository Structure

The project is split into three repositories:

dev-to-prod-promotion/
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

5. Application Microservices

Three services are deployed:

Service

Container Port

Kubernetes Service Port

frontend

8080

8080

product-service

8081

8081

order-service

8082

8082

The frontend communicates with backend services through Kubernetes DNS:

frontend
   │
   ├── http://product-service:8081
   │
   └── http://order-service:8082

Frontend endpoints:

/
 /products
 /orders
 /health

6. Docker and ECR Image Strategy

Each service is packaged as a Docker image and pushed to Amazon ECR.

Example:

<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/frontend

ECR repositories:

frontend
product-service
order-service

Configuration:

Tag mutability: IMMUTABLE
Scan on push: enabled
Encryption: AES256

Why Immutable Tags Matter

The pipeline initially attempted to reuse:

frontend:1.1.0

ECR rejected the push because the repository was configured with
immutable tags.

The pipeline was changed to generate the image tag from the Git commit:

env.IMAGE_TAG = sh(
    script: 'git rev-parse --short=7 HEAD',
    returnStdout: true
).trim()

The resulting promotion artifact was:

07eda39

Therefore:

frontend:07eda39
product-service:07eda39
order-service:07eda39

Why Git SHA tags?

They provide:

immutability

source traceability

reproducibility

easier rollback

auditability

7. Build Once, Promote the Same Artifact

One of the most important CI/CD principles demonstrated here:

BUILD ONCE

07eda39
   │
   ├── DEV       ✅
   ├── STAGING   ✅
   └── PROD      ✅

We do not rebuild the application separately for each environment.

Bad pattern:

Build → Dev
Build → Staging
Build → Production

Preferred pattern:

Build once
   ↓
Create immutable artifact
   ↓
Promote the same artifact

This reduces the risk that the artifact tested in Staging differs from
the artifact deployed to Production.

8. Jenkins CI Pipeline

Jenkins is responsible primarily for CI and artifact creation.

GitHub
  ↓
Jenkins
  ↓
Checkout
  ↓
Test
  ↓
Docker Build
  ↓
Generate Git SHA image tag
  ↓
Push images to ECR
  ↓
Update GitOps repository
  ↓
Create Pull Request

Jenkins responsibilities

Checkout source

Run tests

Build Docker images

Generate immutable image tags

Authenticate to ECR

Push images

Update GitOps values

Create GitHub Pull Requests

Jenkins does not directly make production the desired state. The GitOps
repository and Argo CD control deployment.

9. GitOps Repository

The GitOps repository contains the desired deployment state.

Example:

environments/dev/frontend-values.yaml
environments/staging/frontend-values.yaml
environments/production/frontend-values.yaml

A values file determines which artifact should run:

image:
  repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/frontend
  tag: "07eda39"

imagePullSecrets:
  - name: ecr-registry-secret

The GitOps repository answers:

What version should this environment be running?

10. Pull Request Promotion Model

Promotion is controlled through Git.

Jenkins
   │
   ▼
GitOps PR
   │
   ▼
Review / Approval
   │
   ▼
Merge
   │
   ▼
Argo CD
   │
   ▼
Kubernetes environment

For this project:

PR #4 → 07eda39 → DEV
PR #5 → 07eda39 → STAGING
PR #6 → staging configuration fix
PR #7 → 07eda39 → PRODUCTION

The Production PR acts as the explicit approval gate.

11. Helm

Helm packages each microservice as a reusable Kubernetes deployment
unit.

helm/
├── frontend/
├── product-service/
└── order-service/

Each chart contains:

Chart.yaml
values.yaml
templates/
    deployment.yaml
    service.yaml
    ingress.yaml

The chart provides reusable defaults while environment-specific values
provide overrides.

Helm chart
     │
     ├── dev values
     ├── staging values
     └── production values

This avoids duplicating complete Kubernetes manifests for every
environment.

12. Kubernetes Architecture

The lab uses one K3s cluster with namespace-based environment
separation:

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

This is a lightweight lab architecture.

A larger production platform could use:

separate Kubernetes clusters

separate AWS accounts

separate VPCs

stronger network isolation

Namespaces provide useful environment boundaries in this lab, but should
not be described as equivalent to separate AWS accounts or clusters.

13. Kubernetes Networking

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

Inside Kubernetes:

frontend Pod
     │
     ├── product-service:8081
     │
     └── order-service:8082

Important distinctions:

EC2 port 80 is the external entry point exposed by Traefik.

Frontend container port is 8080.

Kubernetes Services provide stable internal endpoints.

Ingress is a Kubernetes API resource containing routing rules.

Traefik is the Ingress Controller that implements those rules.

14. Argo CD

Argo CD continuously compares:

Git desired state
        VS
Kubernetes actual state

Architecture:

GitHub GitOps
      │
      ▼
   Argo CD
      │
      ├── Compare
      ├── Sync
      └── Self-heal
      │
      ▼
 Kubernetes

The Applications are:

microservices-dev
microservices-staging
microservices-prod

An environment is treated as the deployment boundary for the three
services.

15. Argo CD Automated Sync and Self-Healing

The Applications use:

syncPolicy:
  automated:
    prune: true
    selfHeal: true

If Kubernetes is manually changed while Git says something different:

Git:
replicas = 1

Kubernetes:
replicas = 3

Argo CD detects drift and restores the Git-defined state.

Self-healing was demonstrated during the project by manually changing
the frontend replica count and observing Argo restore the desired state.

16. ECR Authentication in K3s

K3s uses containerd rather than the host Docker Engine.

Therefore:

Docker Engine image store
        ≠
K3s/containerd image store

A Docker image visible with:

docker images

is not automatically available to K3s.

The project initially demonstrated this with a local image and later
moved to ECR, which is the appropriate registry-based deployment model.

Kubernetes uses:

imagePullSecrets:
  - name: ecr-registry-secret

The ECR authorization token is temporary.

During the project, deployment failed with:

403 Forbidden
denied: Your authorization token has expired.

Refreshing the Kubernetes registry secret fixed the deployment.

For a hardened production platform, automatic ECR credential management
should be preferred over manually refreshing registry secrets.

17. Terraform Remote State

Terraform state is stored remotely in S3.

Terraform
    │
    ▼
S3 bucket
    │
    ├── encryption
    ├── versioning
    └── native state locking

The root Terragrunt configuration uses an S3 remote state configuration
similar to:

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

Remote state provides centralized, durable state storage for
infrastructure management.

18. Why the Backend Bucket Is Bootstrapped Separately

There is a chicken-and-egg problem:

Terraform needs S3
       ↓
S3 stores Terraform state
       ↓
The S3 bucket must exist first

A small bootstrap Terraform configuration created the state bucket
before the normal remote backend was used.

The bucket was configured with:

versioning

AES256 server-side encryption

public-access blocking

19. Terraform IAM Troubleshooting Incident

The bootstrap initially failed because the EC2 IAM role did not have
every S3 permission Terraform required.

The important lesson:

Terraform may call several AWS APIs to manage one resource, so an IAM
policy can appear broadly correct while still missing a required API
action.

Troubleshooting flow:

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

The preferred approach is to add the specific missing permission rather
than immediately granting broad administrator access.

20. Terraform Taint Incident

During the S3 bootstrap, the bucket was created but a later operation
failed. Terraform marked the bucket as tainted.

A subsequent plan proposed:

-/+ destroy and recreate

That plan was not applied.

Instead:

terraform untaint aws_s3_bucket.terraform_state

Then:

terraform plan

was reviewed again.

The final safe plan avoided destroying the existing bucket.

Operational lesson:

Always inspect Terraform plans, especially destroy/replacement
actions, before applying them.

This is particularly important for:

S3 buckets

databases

production networking

persistent storage

IAM resources

21. Production-Style Incident: Empty Staging Values

During Staging promotion, Argo CD initially showed:

STAGING
Degraded

The frontend was healthy, but:

product-service → ImagePullBackOff
order-service   → ImagePullBackOff

The root cause was configuration, not Kubernetes.

The staging values files for the backend services were empty.

Because the environment-specific values were missing, Helm fell back to
chart defaults:

product-service
order-service

instead of the ECR repositories.

The fix was made through Git:

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

This incident demonstrates why configuration is part of the application
delivery system and why GitOps makes configuration changes auditable.

22. Final Production Promotion

The same immutable artifact progressed through every environment:

07eda39
   │
   ├── DEV       ✅ Healthy
   │
   ├── STAGING   ✅ Healthy
   │
   └── PROD      ✅ Healthy

The Production GitOps PR was:

promote 07eda39 to production

After merge, Argo CD reported:

APP HEALTH:  Healthy
SYNC STATUS: Synced to main

The Production environment contained:

frontend
product-service
order-service

with the frontend configured for three replicas.

23. Why GitOps Instead of Jenkins kubectl apply?

Traditional deployment:

Jenkins
   ↓
kubectl apply
   ↓
Kubernetes

GitOps deployment:

Jenkins
   ↓
GitOps PR
   ↓
Merge
   ↓
Argo CD
   ↓
Kubernetes

GitOps benefits:

audit trail

pull-request approval

Git-based rollback

drift detection

reconciliation

clear separation of CI and CD

reduced direct cluster access for Jenkins

24. Why Jenkins Does Not Deploy Directly

The responsibilities are intentionally separated:

Jenkins
  = Build / Test / Package / Publish

Argo CD
  = Deploy / Reconcile / Self-heal

Jenkins creates and publishes the artifact and proposes a GitOps change.

Argo CD observes the GitOps repository and makes Kubernetes match the
approved desired state.

This reduces the amount of Kubernetes access required by Jenkins.

25. Rollback Strategy

Helm provides release history:

helm history frontend -n microservices-dev

A rollback moves the release back to a known-good revision.

In a GitOps environment, another preferred rollback mechanism is to
revert the GitOps change:

Bad GitOps commit
       ↓
Git revert
       ↓
Argo CD
       ↓
Known-good Kubernetes state

The key idea is to make rollback reproducible and auditable.

26. Verification and Testing

Three levels of testing were used.

External routing

curl http://localhost/

Validates:

EC2
 → Traefik
 → frontend Service
 → frontend Pod

Internal service connectivity

A temporary curl pod was used to test:

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

27. Useful Troubleshooting Commands

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

28. Security Considerations

This is a lab, but the production lessons are important.

IAM

Use least privilege rather than:

AdministratorAccess

ECR

Use:

immutable tags
scan on push

Secrets

Never commit:

passwords
tokens
AWS credentials
private keys

Jenkins

Adding Jenkins to the Docker group effectively provides root-equivalent
control over the host. Acceptable for this lab, but a serious security
boundary in production.

Kubernetes

A hardened platform should also consider:

RBAC

NetworkPolicies

Pod Security Standards

resource requests/limits

Horizontal Pod Autoscaling

PodDisruptionBudgets

secrets management

TLS

image signing

vulnerability gates

workload identity

29. Production Improvements

A real production implementation could add:

Separate AWS accounts per environment.

Separate Kubernetes clusters where appropriate.

Automatic ECR credential management.

Production ingress/load balancer architecture.

TLS certificates through ACM or cert-manager.

AWS Secrets Manager / External Secrets.

Kubernetes NetworkPolicies.

CPU/memory requests and limits.

Horizontal Pod Autoscaling.

PodDisruptionBudgets.

Trivy or another vulnerability gate.

Image signing and verification.

Jenkins ephemeral agents.

Centralized logging.

Prometheus/Grafana monitoring.

Distributed tracing.

Argo CD Projects and RBAC.

Required GitHub reviewers for production.

Jira integration for change tracking.

Automated deployment notifications.

30. Jira / Change Management

Jira can sit above the Git workflow as the work-management layer.

Example:

Jira ticket
    │
    ▼
Feature branch
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
Production approval
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

GitHub can remain the source-control platform. There is no requirement
to move to Bitbucket simply because Jira is being used.

31. Senior DevOps Interview Explanation

A concise interview answer:

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

32. Senior DevOps Interview Questions

CI/CD

Why use Git SHA tags?

They create immutable, traceable artifacts that map directly to a source
commit.

Why not use latest?

latest is mutable and makes deployments difficult to reproduce and
audit.

Why did ECR reject the 1.1.0 push?

The repository was configured with immutable tags, so an existing tag
could not be overwritten.

What is the difference between CI and CD?

CI validates and packages changes. CD delivers approved artifacts to
environments.

GitOps

Why use a separate GitOps repository?

It separates source code from deployment configuration and provides a
clear desired-state repository.

Why doesn’t Jenkins run kubectl apply?

Argo CD owns deployment and reconciliation. Jenkins does not need direct
deployment access to the cluster.

What happens if someone manually changes Production?

Argo CD detects the drift and, with self-healing enabled, reconciles it
back to Git.

Kubernetes

What does a Service do?

It provides a stable network endpoint and directs traffic to matching
Pods.

What does an Ingress do?

It defines HTTP/HTTPS routing rules into the cluster. The Ingress
Controller implements those rules.

What is Traefik?

The Ingress Controller used by this K3s installation.

Why can frontend reach product-service:8081?

Kubernetes DNS resolves the Service name inside the cluster.

Helm

Why use Helm?

It packages Kubernetes resources into reusable templates and allows
environment-specific values without duplicating complete manifests.

What is the difference between chart defaults and environment values?

values.yaml contains defaults. Environment-specific values override
those defaults.

Terraform

Terraform vs Terragrunt?

Terraform provisions infrastructure. Terragrunt organizes and
orchestrates Terraform configurations and environments.

Why use remote state?

It centralizes state and makes it durable and shareable while supporting
encryption, versioning and locking.

Why S3 versioning?

It provides historical versions of Terraform state and can help recover
from accidental state changes.

What should you do when Terraform proposes destroying a critical resource?

Stop and investigate. Never blindly apply the plan.

33. Failure Scenarios Demonstrated

Problem

Root Cause

Resolution

ECR tag push rejected

Immutable tag already existed

Use Git SHA tags

K3s couldn’t use Docker image

Docker and K3s containerd use separate image stores

Use a registry such as ECR

ECR 403

Token expired

Refresh registry authentication

Staging backend ImagePullBackOff

Empty Helm values files

Configure ECR repository/tag/pull secret through GitOps

Terraform AccessDenied

Missing IAM permission

Identify required AWS API permission

Terraform planned replacement

Resource was tainted after partial failure

Inspect plan and untaint safely

Argo deployment initially stale

GitOps change had not yet reconciled

Refresh/wait for Argo reconciliation

34. What This Project Demonstrates

This project demonstrates understanding of the complete delivery system:

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

35. Final State

The completed promotion:

                    Git Commit
                     07eda39
                         │
                         ▼
                  ┌─────────────┐
                  │   Jenkins   │
                  └──────┬──────┘
                         │
                         ▼
                       ECR
             ┌───────────┼───────────┐
             ▼           ▼           ▼
         frontend     product      order
          07eda39      07eda39     07eda39
             │           │           │
             └───────────┼───────────┘
                         ▼
                     GitOps PRs
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
           DEV       STAGING       PROD
             │           │           │
             ▼           ▼           ▼
          Healthy      Healthy      Healthy
             │           │           │
             └───────────┼───────────┘
                         ▼
                      Argo CD
                         │
                         ▼
                       K3s

Result: the exact same immutable application artifact successfully
progressed from Dev → Staging → Production through GitOps-controlled
promotion.

36. Repository Links

Application repository:

https://github.com/paycenonoli/dev-to-prod-microservices-app

GitOps repository:

https://github.com/paycenonoli/dev-to-prod-microservices-gitops

Infrastructure repository:

https://github.com/paycenonoli/dev-to-prod-infrastructure

37. Ten Key Takeaways

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

Project Status

Area

Status

Application source

Complete

Docker

Complete

Amazon ECR

Complete

Helm

Complete

GitOps

Complete

Jenkins CI

Complete

Argo CD CD

Complete

Dev promotion

Complete

Staging promotion

Complete

Production promotion

Complete

Terraform/Terragrunt

Complete

Production GitOps health

Healthy
