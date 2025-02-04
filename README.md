# GitOps Deployment with ArgoCD and Helm

This repository demonstrates a complete **CI/CD pipeline** using GitHub Actions to build, test, and deploy a custom Helm chart to an Amazon EKS cluster. The deployment leverages **ArgoCD** for GitOps-based continuous delivery and integrates **Docker**, **ECR**, **Helm**, and **Kubernetes**. The project also includes **Infrastructure as Code (IaC)** for provisioning the EKS cluster and associated resources.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Repository Structure](#repository-structure)
5. [Key Design Decisions](#key-design-decisions)
   - [Ingress Host Configuration](#ingress-host-configuration)
   - [OIDC and Assume Role for Security](#oidc-and-assume-role-for-security)
   - [StatefulSet for Database](#statefulset-for-database)
6. [Workflow Details](#workflow-details)
7. [Deployment Instructions](#deployment-instructions)
8. [Testing Locally](#testing-locally)
9. [Security Considerations](#security-considerations)
10. [Contributing](#contributing)
11. [License](#license)

---

## Overview

This project automates the deployment of a multi-component application to an Amazon EKS cluster using **GitOps principles**. The workflow includes:

- **Code Testing**: Runs unit tests, static code analysis (Checkstyle), and SonarQube code quality checks.
- **Docker Image Build and Push**: Builds a Docker image from the application code and pushes it to Amazon Elastic Container Registry (ECR).
- **EKS Deployment**: Configures Kubernetes access, installs ArgoCD, and deploys the Helm chart using ArgoCD for GitOps-based synchronization.
- **Monitoring and Debugging**: Provides detailed logs and debugging steps in case of failures.

The Helm chart defines deployments, services, and ingress rules for a multi-component application, including:
- **Application Service (`vproapp`)**
- **Database (`vprodb`)**
- **Memcached (`vpromc`)**
- **RabbitMQ (`vprormq`)**

---

## Architecture

The architecture consists of the following components:

1. **GitHub Actions Workflow**:
   - Triggers on `push` or `pull_request` to the `stage` or `main` branches.
   - Executes three jobs: **Testing**, **DockerBuildPush**, and **DeployEKS**.

2. **Amazon EKS Cluster**:
   - Hosts the Kubernetes resources deployed via Helm and managed by ArgoCD.

3. **ArgoCD**:
   - Manages the deployment of the Helm chart from the Git repository.
   - Ensures the desired state of the cluster matches the configuration in the repository.

4. **Custom Helm Chart**:
   - Defines Kubernetes resources for the application, including deployments, services, and ingress.

![Stack Diagram](path/to/stack-image.png) <!-- Replace with the actual path to your stack image -->

---

## Prerequisites

Before deploying the infrastructure, ensure the following prerequisites are met:

1. **AWS Account**:
   - An active AWS account with permissions to create EKS clusters, ECR repositories, and IAM roles.
   - Configure the following secrets in GitHub Actions:
     - `AWS_ACCESS_KEY_ID`
     - `AWS_SECRET_ACCESS_KEY`
     - `ROLE_ARN`

2. **GitHub Repository**:
   - Fork this repository and configure the required secrets in the **Settings > Secrets and Variables > Actions** section.

3. **S3 Bucket**:
   - An S3 bucket (e.g., `gitiopsbob`) in the `us-east-2` region to store Terraform state files.

4. **Tools**:
   - Install Docker, AWS CLI, kubectl, and Helm locally for testing purposes.

---

## Key Design Decisions

### 1. **Ingress Host Configuration**

The ingress resource (`vproingress.yaml`) defines the entry point for external traffic to access the application. The `host` field in the ingress configuration specifies the domain name used to route traffic to the Kubernetes services.

- **Why Change the Ingress Host?**
  - The ingress host (`eksproject1.satsan.site`) must be updated to match your organization's domain or DNS configuration.
  - Example: If your domain is `example.com`, update the `values.yaml` file:
    ```yaml
    ingress:
      host: app.example.com
    ```
  - This ensures that traffic is routed correctly to your application. Without updating the host, the ingress controller will not be able to resolve the domain, leading to service unavailability.

### 2. **OIDC and Assume Role for Security**

The OpenID Connect (OIDC) provider and IAM roles are critical components of this architecture. They ensure secure integration between GitHub Actions and AWS services.

- **How OIDC Works in This Setup:**
  - The OIDC provider (`oicd.tf`) allows GitHub Actions to authenticate with AWS without storing long-term credentials in the repository.
  - Example:
    ```hcl
    resource "aws_iam_role" "github_actions" {
      name = "github-actions-eks-role"
      assume_role_policy = jsonencode({
        Version   = "2012-10-17"
        Statement = [
          {
            Action    = "sts:AssumeRoleWithWebIdentity"
            Effect    = "Allow"
            Principal = {
              Federated = aws_iam_openid_connect_provider.github_actions.arn
            }
            Condition = {
              StringLike = {
                "token.actions.githubusercontent.com:sub" : "repo:${var.github_org}/${var.github_repo}:*"
              }
            }
          }
        ]
      })
    }
    ```
  - **Security Benefits:**
    - **Least Privilege**: The IAM role grants only the permissions required for GitHub Actions to interact with AWS services like ECR, EKS, and ArgoCD.
    - **Temporary Credentials**: OIDC uses short-lived tokens, reducing the risk of credential leakage.
    - **Granular Access Control**: The `Condition` block ensures that only specific GitHub repositories can assume the role.

### 3. **StatefulSet for Database**

A **StatefulSet** was chosen for the database (`db-statefulset.yaml`) instead of a Deployment or standalone Pod. This decision ensures data persistence, stable network identities, and ordered deployment/termination.

- **Why Use a StatefulSet?**
  - **Data Persistence**: Databases require persistent storage to retain data across pod restarts. The `volumeClaimTemplates` in the StatefulSet ensures each pod gets its own PersistentVolume (PV).
    ```yaml
    volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
        storageClassName: ""
    ```
  - **Stable Network Identity**: Each pod in a StatefulSet has a unique and stable hostname (e.g., `vprodb-0`). This is critical for databases that rely on consistent network identities for replication or failover.
  - **Ordered Deployment/Termination**: StatefulSets deploy and terminate pods in a predictable order, ensuring data integrity during scaling or updates.

---

## Workflow Details

### 1. Testing Job
- **Purpose**: Validates the application code.
- **Steps**:
  - Runs Maven tests (`mvn clean test`).
  - Performs static code analysis using Checkstyle.
  - Analyzes code quality with SonarQube.

### 2. DockerBuildPush Job
- **Purpose**: Builds and pushes the Docker image to Amazon ECR.
- **Steps**:
  - Configures AWS credentials.
  - Logs into Amazon ECR.
  - Builds the Docker image using the `Dockerfile`.
  - Pushes the image to the ECR repository.

### 3. DeployEKS Job
- **Purpose**: Deploys the Helm chart to the EKS cluster using ArgoCD.
- **Steps**:
  - Configures AWS credentials and updates the kubeconfig for the EKS cluster.
  - Installs ArgoCD if not already installed.
  - Deploys the Helm chart using ArgoCD.
  - Verifies the deployment and waits for resources to stabilize.

---

## Deployment Instructions

### 1. Clone the Repository
```bash
git clone https://github.com/<your-org>/<your-repo>.git
cd <your-repo>
```

### 2. Configure GitHub Secrets
Set the following secrets in your GitHub repository:
- `AWS_ACCESS_KEY_ID`: AWS access key ID.
- `AWS_SECRET_ACCESS_KEY`: AWS secret access key.
- `ROLE_ARN`: ARN of the IAM role for EKS access.
- `REGISTRY`: URL of the ECR registry.
- `SONAR_URL`, `SONAR_ORGANIZATION`, `SONAR_PROJECT_KEY`, `SONAR_TOKEN`: SonarQube credentials.

### 3. Trigger the Workflow
Push changes to the `stage` branch to validate locally:
```bash
git checkout stage
git add .
git commit -m "Trigger CI/CD pipeline for validation"
git push origin stage
```

Once validated, merge changes into the `main` branch for production deployment:
```bash
git checkout main
git merge stage
git push origin main
```

### 4. Verify Deployment
- Monitor the GitHub Actions workflow for success.
- Access the application using the ingress host defined in `values.yaml`.

---

## Testing Locally

To test the deployment locally:
1. **Install Tools**:
   - Install Docker, AWS CLI, kubectl, and Helm.

2. **Configure AWS CLI**:
   ```bash
   aws configure
   ```

3. **Update kubeconfig**:
   ```bash
   aws eks update-kubeconfig --region us-east-2 --name vprofile1-eks
   ```

4. **Install ArgoCD**:
   ```bash
   kubectl create namespace argocd
   helm repo add argo https://argoproj.github.io/argo-helm
   helm install argocd argo/argo-cd --namespace argocd
   ```

5. **Apply the Application Manifest**:
   ```bash
   kubectl apply -f argocd_app/application.yaml
   ```

6. **Verify Resources**:
   ```bash
   kubectl get pods -n argocd
   kubectl get all -n default
   ```

---

## Security Considerations

1. **Least Privilege**:
   - IAM roles and policies are scoped to the minimum required permissions.

2. **Secrets Management**:
   - Use GitHub Actions secrets to securely store sensitive information like AWS credentials and SonarQube tokens.

3. **Image Security**:
   - Ensure Docker images are scanned for vulnerabilities before deployment.

4. **Network Security**:
   - Use private subnets and security groups to restrict access to the EKS cluster.

---

## Contributing

Contributions are welcome! To contribute:

1. Fork the repository.
2. Create a new branch for your changes:
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. Submit a pull request with a detailed description of your changes.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

