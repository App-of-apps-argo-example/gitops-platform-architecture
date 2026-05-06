# GitOps Platform Architecture

This repository documents our GitOps setup. We use Amazon EKS and ArgoCD in a pull-based model, with OIDC for authentication and an App-of-Apps pattern to manage the `test`, `stage`, and `prod` environments.

## Repository Layout

We split the platform into four types of repositories to separate concerns and restrict access. Developers only get access to their application code, which stops them from accidentally breaking the infrastructure or bypassing security checks.

### 1. Infrastructure (`terraform`)
Contains the Terraform code that provisions the AWS VPC, EKS Cluster, and ECR registries, and installs ArgoCD.
- State is stored in S3/DynamoDB.
- `tfsec` runs on PRs to catch misconfigurations.
- **Access:** DevOps only. Developers cannot read or write to this repo.

### 2. Pipeline Templates (`pipeline-templates`)
Holds the GitHub Actions workflows (`reusable-build-deploy.yaml`).
- Enforces standardized CI/CD steps.
- Builds multi-architecture Docker images (amd64/arm64).
- Runs CodeQL (SAST) and Trivy (container scanning).
- Bumps semantic versions and tags ECR images.
- Uses `yq` to commit image tag changes to the GitOps state repository.
- **Access:** DevOps only. Developers trigger these workflows but cannot edit the steps or skip the security scans.

### 3. GitOps State (`gitops-core`)
The source of truth for the Kubernetes cluster. ArgoCD monitors this repository and applies its manifests.
- Uses the App-of-Apps pattern.
- A single Helm chart (`general-service`) is templated out for different environments using `values-test.yaml`, `values-stage.yaml`, and `values-prod.yaml`.
- **Access:** DevOps only. The CI/CD bots push image tag updates here. Developers cannot modify the cluster state.

### 4. Application Repos (`microservice-a`, `microservice-b`, `microservice-c`)
The actual business logic.
- Each repo contains the source code, a Dockerfile, and a simple `.github/workflows/ci.yaml` that calls the central pipeline template.
- **Access:** Developers have full read/write access to write code and merge branches.

## Developer vs. DevOps Responsibilities

We treat CI/CD as a black box for developers to enforce security policies. 

### The Developer Experience
Developers push code to `test`, `stage`, or `main` in their microservice repository. They wait for the GitHub Action to complete, and the code eventually appears in the target environment. They don't run `kubectl` commands, they don't manage AWS resources, and they can't disable Trivy if it flags a vulnerable package.

### The DevOps Experience
Platform engineers manage the `pipeline-templates` repository to enforce compliance rules. They modify the `terraform` repository when we need new cloud resources, and they manage ArgoCD through the `gitops-core` repository.

## Key Mechanisms

### Secretless Authentication
We don't store AWS access keys in GitHub. Instead, GitHub Actions uses OIDC to request a short-lived token from AWS (`GitHubActionsDeployRole`). If a GitHub repository is compromised, there are no long-lived credentials to steal.

### External Secrets Operator
Instead of storing base64-encoded Kubernetes Secrets in Git, we run the External Secrets Operator (ESO). ESO uses an IAM Role for Service Accounts (IRSA) to fetch passwords from AWS Secrets Manager and mounts them into Pods at runtime.

### Mandatory Security Scanning
The central pipeline enforces two scans before deploying:
- **CodeQL:** Checks the source code for logical vulnerabilities like XSS or SQL injection.
- **Trivy:** Scans the Docker image for OS and library vulnerabilities. The pipeline fails if it finds `CRITICAL` or `HIGH` severity CVEs.

### Environment Management
ArgoCD runs a root application that discovers and deploys child applications based on the Git branch:
- Pushing to `test` updates `values-test.yaml`, which deploys to the `test` namespace.
- Pushing to `stage` updates `values-stage.yaml`, which deploys to the `stage` namespace.
- Pushing to `main` updates `values-prod.yaml`, which deploys to the `prod` namespace.

### Versioning and Architecture
The pipeline determines the next semantic version based on existing Git tags. It builds images for both `linux/amd64` and `linux/arm64` using QEMU and Docker Buildx.

## The Deployment Flow

1. A developer pushes code to the `stage` branch of `microservice-a`.
2. GitHub Actions runs the central workflow from `pipeline-templates`.
3. CodeQL scans the source code.
4. Docker builds the multi-arch image, and Trivy scans it for vulnerabilities.
5. GitHub Actions authenticates with AWS via OIDC and pushes the image to ECR.
6. The CI bot clones `gitops-core`, updates the image tag in `values-stage.yaml` using `yq`, and commits the change.
7. ArgoCD detects the new commit in `gitops-core` and updates the Kubernetes cluster.
8. The External Secrets Operator fetches the database password from AWS Secrets Manager and injects it into the new Pods.