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
## Initial Platform Setup

This section covers how to bootstrap the platform from scratch in a new AWS account. We use Workload Identity Federation (OIDC) so GitHub Actions can talk to AWS without us storing static `AWS_ACCESS_KEY_ID` secrets.

### 1. Configure AWS OIDC

We need to tell AWS to trust our GitHub organization. Run this locally with an admin AWS profile:

```bash
export AWS_PROFILE="aws"
export GITHUB_ORG="App-of-apps-argo-example" # Change to your org
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

# 1. Register GitHub as an OIDC provider in AWS
aws iam create-open-id-connect-provider \
  --url "https://token.actions.githubusercontent.com" \
  --client-id-list "sts.amazonaws.com" \
  --thumbprint-list "1b511abead59c6ce207077c0bf0e0043b1382612"

# 2. Create the trust policy
cat <<POLICY > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GITHUB_ORG}/*"
        },
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
POLICY

# 3. Create the GitHub Actions role
aws iam create-role \
  --role-name GitHubActionsDeployRole \
  --assume-role-policy-document file://trust-policy.json

# 4. Attach admin permissions (scope this down for production!)
aws iam attach-role-policy \
  --role-name GitHubActionsDeployRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Update your GitHub Actions workflows to use `arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsDeployRole`.

### 2. GitHub Secrets Setup

Create a Personal Access Token (PAT) with repository access and add it as an Organization Secret in GitHub:
- **Name:** `GITOPS_BOT_TOKEN`
- **Value:** `<your-pat>`

The central pipeline uses this token to clone the `gitops-core` repo, bump image tags, and push the changes back.

### 3. Provision Infrastructure

Head to the `terraform` repository on GitHub. Go to the **Actions** tab, select the **Deploy EKS and ArgoCD** workflow, and run it.

What Terraform does:
1. **State Bootstrap:** It dynamically creates an S3 bucket and a DynamoDB table for Terraform state if they don't exist.
2. **VPC & EKS:** It provisions the network and a Kubernetes 1.35 cluster.
3. **ECR:** It creates container registries for each microservice with a lifecycle policy (keeps the last 20 images).
4. **ArgoCD:** It installs ArgoCD using Helm and tells it to watch the `gitops-core` repository to bootstrap the rest of the cluster (App-of-Apps).

### 4. Deploy Applications

Once the EKS cluster is up and ArgoCD is running, push code to `microservice-a`, `microservice-b`, or `microservice-c`.
1. GitHub Actions builds a multi-arch image and pushes it to ECR.
2. It tags the release based on Git history.
3. It updates the image tag in `gitops-core` (e.g., `applications/main/values-stage.yaml`).
4. ArgoCD detects the change and rolls out the new version to Kubernetes.
