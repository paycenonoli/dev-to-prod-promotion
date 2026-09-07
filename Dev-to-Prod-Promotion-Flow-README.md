<div align="center">

# 🚀 Dev-to-Production Promotion Flow

### Detailed CI/CD + GitOps Runbook with Commands, Pull Requests & Merge Points

![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)
![Jenkins](https://img.shields.io/badge/Jenkins-CI-red?logo=jenkins)
![Docker](https://img.shields.io/badge/Docker-Container-blue?logo=docker)
![Amazon ECR](https://img.shields.io/badge/Amazon%20ECR-Registry-orange?logo=amazonaws)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes)
![Helm](https://img.shields.io/badge/Helm-Packaging-0F1689?logo=helm)
![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-EF7B4D?logo=argo)
![GitHub](https://img.shields.io/badge/GitHub-Source%20%26%20GitOps-181717?logo=github)
![Terraform](https://img.shields.io/badge/Terraform-IaC-844FBA?logo=terraform)
![Terragrunt](https://img.shields.io/badge/Terragrunt-Orchestration-65C5C9)

**Build once → push once → review → promote → reconcile → validate**

</div>

---

# 1. Purpose

This README is the **step-by-step runbook** for the Dev-to-Production promotion flow used in this project.

It deliberately shows:

- where commands are run
- which repository is being changed
- when a branch is created
- when a Pull Request is opened
- when the PR is merged
- when Jenkins runs
- when ECR receives the image
- when Argo CD deploys
- when validation occurs
- how the same artifact moves from Dev → Staging → Production

> **Golden rule:** Jenkins builds the artifact. GitOps controls the desired deployment state. Argo CD performs the deployment.

---

# 2. The Complete Flow

<div align="center">

<img src="docs/promotion-flow.svg" alt="Dev to Production Promotion Flow" width="100%">

</div>

The complete logical chain is:

```text
Application Code
      │
      ▼
Application GitHub Repository
      │
      │ PR / merge
      ▼
main
      │
      ▼
Jenkins
      │
      ├── test
      ├── build
      ├── tag with Git SHA
      └── push
            │
            ▼
         Amazon ECR
            │
            │ immutable image
            ▼
      GitOps Repository
            │
            │ PR
            ▼
       review / approval
            │
            │ merge
            ▼
          main
            │
            ▼
         Argo CD
            │
            ▼
     Kubernetes / K3s
            │
      ┌─────┼─────┐
      ▼     ▼     ▼
     Dev  Staging Prod
```

---

# 3. Repositories

The project uses three implementation repositories.

| Repository | Purpose |
|---|---|
| `dev-to-prod-microservices-app` | Application source code |
| `dev-to-prod-microservices-gitops` | Helm, environment values and Argo CD |
| `dev-to-prod-infrastructure` | Terraform/Terragrunt infrastructure |

The umbrella repository documents the platform.

---

# 4. Kubernetes Environment Model

This project uses one K3s cluster with three namespaces:

```text
                         K3s
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
 microservices-dev  microservices-staging  microservices-prod
          │               │               │
      frontend        frontend          frontend
      product         product           product
      order           order             order
```

This is a deliberate lab design.

It allows us to demonstrate promotion boundaries without creating three separate Kubernetes clusters.

---

# 5. Before Starting

The working directory is:

```bash
cd ~/dev-to-prod-promotion
```

The three repositories are:

```text
~/dev-to-prod-promotion/microservices-app
~/dev-to-prod-promotion/microservices-gitops
~/dev-to-prod-promotion/infrastructure
```

Check:

```bash
cd ~/dev-to-prod-promotion/microservices-app
git status
```

```bash
cd ~/dev-to-prod-promotion/microservices-gitops
git status
```

```bash
cd ~/dev-to-prod-promotion/infrastructure
git status
```

All repositories should be clean before beginning a controlled promotion.

---

# 6. Stage 0 — Infrastructure Foundation

Infrastructure is managed separately from application promotion.

```text
Terraform
   │
   ▼
AWS infrastructure
   │
   ├── VPC
   ├── IAM
   ├── ECR
   └── other infrastructure
```

Terragrunt organizes Terraform units and environments.

The shared ECR configuration lives at:

```text
infrastructure/environments/shared/ecr/
```

The ECR repositories are:

```text
frontend
product-service
order-service
```

Terraform state is stored remotely in S3.

> Application promotion does **not** run Terraform after every application build. Infrastructure changes and application changes are separate concerns.

---

# 7. Stage 1 — Developer Changes the Application

## Repository

Run from:

```bash
cd ~/dev-to-prod-promotion/microservices-app
```

Create a feature branch:

```bash
git switch main
git pull origin main

git switch -c feature/frontend-update
```

Make the application change.

For example:

```text
services/frontend/app.py
```

Test locally if appropriate.

Then:

```bash
git status
git add .
git commit -m "feat: update frontend"
git push -u origin feature/frontend-update
```

---

# 8. PR #1 — Application Pull Request

At this point:

```text
feature/frontend-update
          │
          ▼
      GitHub PR
          │
          ▼
       Review
          │
          ▼
      Merge → main
```

### Important

**This is the first PR/merge boundary.**

The application code should be reviewed before it becomes the source that Jenkins builds.

After approval, merge the PR into:

```text
main
```

Then:

```bash
git switch main
git pull origin main
```

---

# 9. Stage 2 — Jenkins Starts CI

Once the application change is merged to `main`, Jenkins builds the application.

Conceptually:

```text
GitHub main
     │
     ▼
  Jenkins
     │
     ├── Checkout
     ├── Test
     ├── Build
     ├── Tag
     └── Push to ECR
```

The Jenkinsfile lives in:

```text
microservices-app/Jenkinsfile
```

---

# 10. Stage 3 — Jenkins Creates the Image Tag

The image tag is derived from the Git commit:

```groovy
env.IMAGE_TAG = sh(
    script: 'git rev-parse --short=7 HEAD',
    returnStdout: true
).trim()
```

Example:

```text
07eda39
```

That gives us:

```text
frontend:07eda39
product-service:07eda39
order-service:07eda39
```

The tag identifies the source revision that produced the image.

---

# 11. Stage 4 — Jenkins Builds the Images

Conceptually Jenkins executes:

```bash
docker build \
  -t frontend:${IMAGE_TAG} \
  services/frontend
```

And similarly:

```bash
docker build \
  -t product-service:${IMAGE_TAG} \
  services/product-service
```

```bash
docker build \
  -t order-service:${IMAGE_TAG} \
  services/order-service
```

The actual Jenkinsfile is the authoritative implementation.

The important architecture is:

```text
One Git commit
      │
      ├── frontend image
      ├── product image
      └── order image
```

---

# 12. Stage 5 — Jenkins Pushes to ECR

Jenkins authenticates to ECR:

```bash
aws ecr get-login-password --region us-east-1 \
| docker login \
    --username AWS \
    --password-stdin \
    417521971848.dkr.ecr.us-east-1.amazonaws.com
```

Then pushes the images.

Example:

```bash
docker push \
  417521971848.dkr.ecr.us-east-1.amazonaws.com/frontend:${IMAGE_TAG}
```

The same happens for the backend services.

---

# 13. Immutable Artifact

ECR uses immutable tags.

Therefore:

```text
frontend:07eda39
```

cannot be overwritten by another image.

The artifact chain becomes:

```text
Git commit
   │
   ▼
07eda39
   │
   ▼
ECR
   │
   ├── frontend:07eda39
   ├── product-service:07eda39
   └── order-service:07eda39
```

This is the artifact that will move through all environments.

---

# 14. Stage 6 — Jenkins Updates the GitOps Repository

Jenkins now moves from **CI** into **promotion automation**.

It does not need to rebuild the application for every environment.

Instead, it changes the desired state in the GitOps repository.

Example:

```yaml
image:
  repository: 417521971848.dkr.ecr.us-east-1.amazonaws.com/frontend
  tag: "07eda39"
```

The same tag is used for the corresponding services.

---

# 15. PR #2 — Jenkins Creates the Dev GitOps PR

Jenkins creates a branch similar to:

```text
jenkins/promote-3
```

The branch contains the desired Dev image version.

Example commit:

```text
ci: promote 07eda39 to dev
```

Then Jenkins opens a GitHub PR:

```text
jenkins/promote-3
        │
        ▼
      PR #2
        │
        ▼
     Review
        │
        ▼
   Merge → main
```

### Important

**This is the second PR/merge boundary.**

This PR changes **deployment state**, not application source.

---

# 16. Stage 7 — Argo CD Deploys Dev

After the GitOps PR is merged:

```text
GitOps main
    │
    ▼
Argo CD
    │
    ▼
microservices-dev
```

Argo CD detects that Git now declares:

```text
frontend:07eda39
product-service:07eda39
order-service:07eda39
```

Argo CD renders the Helm charts using:

```text
helm/frontend
helm/product-service
helm/order-service
```

and:

```text
environments/dev/
```

---

# 17. Stage 8 — Validate Dev

Run:

```bash
kubectl get pods -n microservices-dev
```

Expected:

```text
frontend          Running
product-service   Running
order-service     Running
```

Check Argo:

```bash
kubectl get applications -n argocd
```

The Dev Application should become:

```text
Synced
Healthy
```

---

# 18. Validate the Application End-to-End

Frontend:

```bash
curl http://localhost/
```

Products:

```bash
curl http://localhost/products
```

Orders:

```bash
curl http://localhost/orders
```

The important test is not merely:

```text
Pod = Running
```

We also validate application behavior:

```text
Client
  │
  ▼
Traefik
  │
  ▼
Frontend
  │
  ├────────► product-service
  │
  └────────► order-service
```

---

# 19. Dev Promotion Checkpoint

At this point we have:

```text
Application code
      │
      ▼
Merged application PR
      │
      ▼
Jenkins
      │
      ▼
ECR: 07eda39
      │
      ▼
Merged GitOps PR
      │
      ▼
Argo CD
      │
      ▼
Dev
      │
      ▼
Validation PASS
```

Only after Dev validation do we promote the artifact.

---

# 20. Stage 9 — Promote Dev → Staging

## Repository

Run from:

```bash
cd ~/dev-to-prod-promotion/microservices-gitops
```

Start from current `main`:

```bash
git switch main
git pull origin main
```

Create the promotion branch:

```bash
git switch -c promote-07eda39-to-staging
```

Update the staging values.

For example:

```bash
sed -i 's/tag: ".*"/tag: "07eda39"/' \
  environments/staging/frontend-values.yaml
```

```bash
sed -i 's/tag: ".*"/tag: "07eda39"/' \
  environments/staging/product-values.yaml
```

```bash
sed -i 's/tag: ".*"/tag: "07eda39"/' \
  environments/staging/order-values.yaml
```

Review:

```bash
git diff
```

The expected change is the image tag.

---

# 21. Commit the Staging Promotion

```bash
git add environments/staging/
git commit -m "promote 07eda39 to staging"
git push -u origin promote-07eda39-to-staging
```

---

# 22. PR #3 — Staging Promotion PR

Create the PR:

```text
promote-07eda39-to-staging
              │
              ▼
          GitHub PR
              │
              ▼
           Review
              │
              ▼
         Merge → main
```

### Important

**This is the third PR/merge boundary.**

Notice what did **not** happen:

```text
❌ docker build
❌ docker push
❌ create a new image
```

We are promoting the existing:

```text
07eda39
```

---

# 23. Stage 10 — Argo CD Deploys Staging

After the staging PR merges:

```text
GitOps main
    │
    ▼
Argo CD
    │
    ▼
microservices-staging
```

Check:

```bash
kubectl get pods -n microservices-staging
```

Check Argo:

```bash
kubectl get applications -n argocd
```

The staging Application should become:

```text
Synced
Healthy
```

---

# 24. Validate Staging

Validate the workloads:

```bash
kubectl get deployments -n microservices-staging
```

```bash
kubectl get pods -n microservices-staging
```

Check the image:

```bash
kubectl get pods -n microservices-staging \
  -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.status.containerStatuses[0].image}{"\n"}{end}'
```

The expected image tag is:

```text
07eda39
```

This confirms Staging received the same artifact.

---

# 25. Stage 11 — Promote Staging → Production

Only after Staging validation passes do we promote Production.

## Repository

```bash
cd ~/dev-to-prod-promotion/microservices-gitops
```

Update from main:

```bash
git switch main
git pull origin main
```

Create the promotion branch:

```bash
git switch -c promote-07eda39-to-production
```

Update Production values:

```bash
sed -i 's/tag: ".*"/tag: "07eda39"/' \
  environments/production/frontend-values.yaml
```

```bash
sed -i 's/tag: ".*"/tag: "07eda39"/' \
  environments/production/product-values.yaml
```

```bash
sed -i 's/tag: ".*"/tag: "07eda39"/' \
  environments/production/order-values.yaml
```

Review:

```bash
git diff
```

---

# 26. Commit the Production Promotion

```bash
git add environments/production/
git commit -m "promote 07eda39 to production"
git push -u origin promote-07eda39-to-production
```

---

# 27. PR #4 — Production Promotion PR

Create the Production PR:

```text
promote-07eda39-to-production
              │
              ▼
          GitHub PR
              │
              ▼
        Production Review
              │
              ▼
         Merge → main
```

### Important

**This is the fourth PR/merge boundary.**

Production deployment is therefore explicitly controlled by a Git change.

---

# 28. Stage 12 — Argo CD Deploys Production

After the Production PR is merged:

```text
GitOps main
     │
     ▼
  Argo CD
     │
     ▼
microservices-prod
```

Check:

```bash
kubectl get pods -n microservices-prod
```

Check:

```bash
kubectl get deployments -n microservices-prod
```

Check Argo:

```bash
kubectl get applications -n argocd
```

Expected:

```text
microservices-prod   Synced   Healthy
```

---

# 29. Final Production Validation

Confirm the Production pods are using the expected tag:

```bash
kubectl get pods -n microservices-prod \
  -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{.status.containerStatuses[0].image}{"\n"}{end}'
```

Expected:

```text
frontend-...         => .../frontend:07eda39
product-service-...  => .../product-service:07eda39
order-service-...    => .../order-service:07eda39
```

Confirm:

```bash
kubectl get pods -n microservices-prod
```

All expected Pods should be:

```text
Running
```

and the Argo Application should be:

```text
Synced
Healthy
```

---

# 30. The Complete PR / Merge Timeline

<div align="center">

<img src="docs/pr-merge-flow.svg" alt="Pull Request and Merge Flow" width="100%">

</div>

The four major merge boundaries are:

| Point | Branch | PR purpose | Merge destination |
|---|---|---|---|
| **PR #1** | `feature/frontend-update` | Application code review | App `main` |
| **PR #2** | `jenkins/promote-3` | Promote image to Dev | GitOps `main` |
| **PR #3** | `promote-07eda39-to-staging` | Promote same image to Staging | GitOps `main` |
| **PR #4** | `promote-07eda39-to-production` | Promote same image to Production | GitOps `main` |

The exact PR numbers may differ on another run; the **four control points** are what matter.

---

# 31. Environment Promotion Diagram

<div align="center">

<img src="docs/environment-promotion.svg" alt="Environment Promotion Flow" width="100%">

</div>

The artifact does not change:

```text
                    SAME IMAGE
                       │
                       ▼
                    07eda39
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
            Dev     Staging   Production
```

What changes is the GitOps declaration:

```yaml
image:
  tag: "07eda39"
```

inside the appropriate environment values file.

---

# 32. Why We Use Separate GitOps PRs

The promotion PR gives us a clear audit trail.

For example:

```text
PR #3
promote 07eda39 to staging
```

tells us:

- what artifact was promoted
- where it was promoted
- who reviewed it
- when it was merged
- what files changed
- what Argo CD subsequently deployed

For Production, this becomes a formal change-control record.

---

# 33. Why Jenkins Does Not Deploy Directly

A simpler pipeline could do:

```bash
kubectl apply ...
```

But this project deliberately uses:

```text
Jenkins
   │
   ▼
GitOps PR
   │
   ▼
Git
   │
   ▼
Argo CD
   │
   ▼
Kubernetes
```

This gives us:

- Git as the source of truth
- auditable deployment changes
- PR approval
- Git-based rollback
- Argo reconciliation
- separation of CI and CD

---

# 34. Why Helm Is Used

Helm separates the reusable Kubernetes template from environment-specific configuration.

Example chart:

```text
helm/frontend/
```

Environment values:

```text
environments/dev/frontend-values.yaml
environments/staging/frontend-values.yaml
environments/production/frontend-values.yaml
```

The same chart can therefore produce:

```text
Dev:
replicas = 1

Staging:
replicas = 2

Production:
replicas = 3
```

while using the same application template.

---

# 35. Argo CD's Role

Argo CD follows this model:

```text
Git
 │
 │ desired state
 ▼
Argo CD
 │
 │ reconcile
 ▼
Kubernetes
```

If someone manually changes Kubernetes:

```bash
kubectl scale deployment frontend --replicas=3 \
  -n microservices-dev
```

Argo CD can detect the drift and restore the Git-defined state when self-healing is enabled.

Therefore:

```text
Git = desired state
Kubernetes = actual state
Argo CD = reconciliation
```

---

# 36. ECR Authentication in Kubernetes

Kubernetes needs permission to pull private ECR images.

The lab uses:

```text
ecr-registry-secret
```

Example:

```bash
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=417521971848.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)" \
  -n microservices-dev
```

Repeat for Staging:

```bash
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=417521971848.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)" \
  -n microservices-staging
```

And Production:

```bash
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=417521971848.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)" \
  -n microservices-prod
```

> **Lab note:** ECR authorization tokens expire. A production implementation should use a more automated AWS-native credential mechanism rather than manually refreshing long-lived Kubernetes secrets.

---

# 37. Important Troubleshooting Example — Expired ECR Token

A real failure occurred during promotion.

The Pod showed:

```text
ImagePullBackOff
```

The underlying error was:

```text
403 Forbidden
denied: Your authorization token has expired.
```

The Dev secret was refreshed with:

```bash
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=417521971848.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)" \
  -n microservices-dev \
  --dry-run=client -o yaml | kubectl apply -f -
```

Argo CD then reconciled the workload and the deployment recovered.

### Senior-level lesson

When you see:

```text
ImagePullBackOff
```

do not immediately assume the image does not exist.

Check:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

and identify the actual image-pull error.

---

# 38. Important Troubleshooting Example — Empty GitOps Values

During Staging promotion, the Product and Order values files were empty.

The attempted:

```bash
sed -i ...
```

commands did not help because there was no existing `tag:` line to replace.

The correct solution was to populate the complete environment values files.

Example:

```yaml
replicaCount: 1

image:
  repository: 417521971848.dkr.ecr.us-east-1.amazonaws.com/product-service
  tag: "07eda39"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: ecr-registry-secret

environment:
  name: staging

service:
  type: ClusterIP
  port: 8081

ingress:
  enabled: false
```

### Senior-level lesson

A successful Git merge does not necessarily mean the desired Kubernetes configuration is correct.

Validate:

```text
Git diff
→ Helm rendering
→ Argo status
→ Kubernetes Pods
→ application behavior
```

---

# 39. Important Troubleshooting Example — Immutable ECR Tag

An earlier Jenkins build attempted to push:

```text
frontend:1.1.0
```

again.

ECR rejected it because the repository is immutable.

The error was essentially:

```text
The image tag '1.1.0' already exists
and cannot be overwritten because the tag is immutable.
```

The solution was to use the Git commit SHA:

```text
07eda39
```

instead.

### Lesson

Do not design CI around mutable tags when artifact traceability matters.

Prefer:

```text
frontend:<commit-sha>
```

over:

```text
frontend:latest
```

---

# 40. Rollback Strategy

Because deployments are represented in Git, rollback can be performed by reverting the GitOps change.

Example:

```text
Production
   │
   ▼
07eda39
```

If the previous known-good version was:

```text
06abc12
```

the GitOps repository can be changed back to:

```yaml
tag: "06abc12"
```

Then:

```text
Git
 │
 ▼
Argo CD
 │
 ▼
Kubernetes
```

Argo CD reconciles Production back to the previous artifact.

---

# 41. Helm Rollback vs GitOps Rollback

There are two different rollback concepts.

### Helm rollback

Helm can maintain release history:

```bash
helm history frontend -n microservices-dev
```

and:

```bash
helm rollback frontend <REVISION> \
  -n microservices-dev
```

This is useful for understanding Helm itself.

### GitOps rollback

In an Argo CD GitOps model, the preferred long-term mechanism is usually:

```text
Git revert
   ↓
Argo CD
   ↓
Kubernetes
```

because Git remains the source of truth.

---

# 42. Final State

After the complete promotion:

```text
                  Git Commit
                      │
                      ▼
                    07eda39
                      │
                      ▼
                 Amazon ECR
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
         Dev       Staging    Production
          │           │           │
       Healthy     Healthy      Healthy
          │           │           │
          └───────────┴───────────┘
                SAME ARTIFACT
```

This is the key outcome of the project.

---

# 43. What Each Tool Actually Does

| Tool | Job |
|---|---|
| GitHub | Source control, PRs, review and audit |
| Jenkins | Continuous Integration and promotion automation |
| Docker | Container image creation |
| ECR | Immutable image storage |
| Helm | Kubernetes application packaging |
| GitOps repo | Desired deployment state |
| Argo CD | Continuous deployment/reconciliation |
| K3s/Kubernetes | Container orchestration |
| Terraform | Infrastructure provisioning |
| Terragrunt | Terraform orchestration and DRY configuration |
| Traefik | Kubernetes ingress controller |

---

# 44. CI vs CD in This Project

## CI

```text
GitHub
  ↓
Jenkins
  ↓
Test
  ↓
Build
  ↓
Docker image
  ↓
ECR
```

## CD / GitOps

```text
GitOps PR
   ↓
Merge
   ↓
Argo CD
   ↓
Helm
   ↓
Kubernetes
```

The separation is intentional.

---

# 45. The Interview Explanation

A strong Senior DevOps explanation is:

> “We use GitHub for source control and pull-request governance. Once application code is merged, Jenkins checks out the revision, runs CI, builds the microservice images, tags them with the Git commit SHA, and pushes immutable images to Amazon ECR. Jenkins then creates a GitOps promotion PR that updates the desired image version in the environment values. After approval and merge, Argo CD detects the Git change and reconciles the Kubernetes environment. We validate Dev first, then promote the exact same immutable image tag to Staging and finally Production through separate GitOps PRs. We don't rebuild the artifact for each environment.”

That is the core architecture.

---

# 46. Promotion Checklist

## Application

```text
[ ] Feature branch created
[ ] Application code changed
[ ] Tests pass
[ ] Application PR opened
[ ] Application PR reviewed
[ ] Application PR merged to main
```

## CI

```text
[ ] Jenkins triggered
[ ] Source checked out
[ ] Tests pass
[ ] Image built
[ ] Git SHA used as tag
[ ] Image pushed to ECR
```

## Dev

```text
[ ] Dev GitOps PR created
[ ] Dev GitOps PR reviewed
[ ] Dev GitOps PR merged
[ ] Argo CD synced
[ ] Dev healthy
[ ] Application tested
```

## Staging

```text
[ ] Staging promotion branch created
[ ] Image tag updated
[ ] GitOps PR opened
[ ] PR reviewed
[ ] PR merged
[ ] Argo CD synced
[ ] Staging healthy
[ ] Same image tag verified
```

## Production

```text
[ ] Production promotion branch created
[ ] Image tag updated
[ ] Production PR opened
[ ] Production approval
[ ] PR merged
[ ] Argo CD synced
[ ] Production healthy
[ ] Same image tag verified
```

---

# 47. The One Diagram to Remember

```text
                 APPLICATION DELIVERY
                 ====================

 Developer
     │
     ▼
Application Repo
     │
     │ PR #1 + MERGE
     ▼
   main
     │
     ▼
  Jenkins
     │
     ├── Test
     ├── Build
     ├── Tag
     └── Push
          │
          ▼
        ECR
          │
          │ 07eda39
          ▼
     GitOps Repo
          │
          │ PR #2 + MERGE
          ▼
         DEV
          │
       Validate
          │
          │ PR #3 + MERGE
          ▼
       STAGING
          │
       Validate
          │
          │ PR #4 + MERGE
          ▼
     PRODUCTION
          │
          ▼
       Healthy
```

**The artifact is built once. The environment changes through Git.**

---

# 48. Final Takeaways

The most important concepts demonstrated by this project are:

1. **Build once, promote many times.**
2. **Use immutable image tags.**
3. **Separate application source from deployment configuration.**
4. **Use Pull Requests as change-control boundaries.**
5. **Keep Git as the desired-state source of truth.**
6. **Let Argo CD reconcile Kubernetes.**
7. **Use Helm for reusable Kubernetes templates.**
8. **Validate each environment before promotion.**
9. **Promote the exact same artifact to Production.**
10. **Keep infrastructure lifecycle separate from application delivery.**

---

<div align="center">

### 🚀 Build Once. Review. Promote. Reconcile. Validate.

**Dev → Staging → Production**

</div>
