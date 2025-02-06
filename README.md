# GitOps Deployment with EKS, ArgoCD and Helm

## Project Architecture

![Stack Diagram](https://github.com/Ashsatsan/gitops-argocd/blob/main/images/NEW%20ONE%20-%20Copy.png?raw=true) 


## IMPORTANT NOTE:
To execute this project properly first execute the IaC part of the project which will creates neccessary resources for this cicd project:
[gitops-terra](https://github.com/Ashsatsan/gitops-terra)

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
6. [CI/CD Workflow Details](#cicd-workflow-details)
7. [SonarCloud Integration](#sonarcloud-integration)
8. [ArgoCD Sync Issues and Troubleshooting](#argocd-sync-issues-and-troubleshooting)
9. [Deployment Instructions](#deployment-instructions)
10. [Testing Locally](#testing-locally)
11. [Security Considerations](#security-considerations)
12. [Verify](#Verify)
13. [Contributing](#contributing)
14. [License](#license)

---

## Overview

This project automates the deployment of a multi-component application to an Amazon EKS cluster using **GitOps principles**. The workflow includes:

- **Code Testing**: Runs unit tests, static code analysis (Checkstyle), and SonarQube/SonarCloud code quality checks.
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

![CICD Diagram](https://github.com/Ashsatsan/gitops-argocd/blob/main/images/NEW%20ONE.png?raw=true) <!-- Replace with the actual path to your stack image -->

---

## Prerequisites

Before deploying the infrastructure, ensure the following prerequisites are met:

1. **AWS Account**:
   - An active AWS account with permissions to create EKS clusters, ECR repositories, and IAM roles.
   - Configure the following secrets in GitHub Actions:
     - `AWS_ACCESS_KEY_ID`
     - `AWS_SECRET_ACCESS_KEY`
     - `ROLE_ARN`

2. **ECR Repository**:
   - Create an Amazon Elastic Container Registry (ECR) repository manually to store Docker images. This ensures that the ECR repository persists even after deleting the Terraform-managed infrastructure.
   - Example:
     ```bash
     aws ecr create-repository --repository-name vprofile-repo --region us-east-2
     ```
   - Register the ECR repository URL and name in GitHub Secrets:
     - `REGISTRY`: The URL of your ECR registry (e.g., `123456789012.dkr.ecr.us-east-2.amazonaws.com`).
     - `ECR_REPO_NAME`: The name of your ECR repository (e.g., `vprofile-repo`).

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

### 2. **OIDC and Assume Role for Security**

The OpenID Connect (OIDC) provider and IAM roles are critical components of this architecture. They ensure secure integration between GitHub Actions and AWS services.

- **Temporary Access with `role-to-assume`:**
  - The `role-to-assume` parameter in the GitHub Actions workflow ensures that only temporary AWS credentials are issued for the duration of the job.
  - Example:
    ```yaml
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: ${{ env.ROLE_ARN }}
        aws-region: ${{ env.REGION }}
    ```

### 3. **StatefulSet for Database**

A **StatefulSet** was chosen for the database (`db-statefulset.yaml`) instead of a Deployment or standalone Pod. This decision ensures data persistence, stable network identities, and ordered deployment/termination.

---

## CI/CD Workflow Details

### Full AWS Access Workflow

This workflow uses full AWS credentials (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) stored in GitHub secrets. While functional, this approach is less secure because long-term credentials are exposed.

```yaml
name: Build, Test, and Push Docker Image
on:
  push:
    branches:
      - main
```

### Secure Workflow with Role Assumption

This workflow uses AWS OIDC and role assumption (`ROLE_ARN`) for secure access. It eliminates the need to store long-term AWS credentials in GitHub secrets.

```yaml
name: Build, Test, and Push Docker Image / eks and argocd
on:
  push:
    branches:
      - main
```

---

## SonarCloud Integration

SonarCloud is integrated into the CI/CD pipeline to perform static code analysis and ensure code quality. Here’s how to set it up:

1. **Create a SonarCloud Account**:
   - Sign up at [https://sonarcloud.io](https://sonarcloud.io) using your GitHub account.
   - After that create a organization and project key
   - Here's a demonstration

   ![image alt](https://github.com/Ashsatsan/gitops-argocd/blob/main/images/gitops5.png?raw=true)

   - Also create a quality gate

   ![image alt](https://github.com/Ashsatsan/gitops-argocd/blob/main/images/gitops6.png?raw=true)
   
   
3. **Generate a Token**:
   - Go to **User Settings > Security** and generate a token. Copy the token value.

4. **Register Secrets in GitHub**:
   - Add the following secrets to your GitHub repository under **Settings > Secrets and Variables > Actions**:
     - `SONAR_URL`: `https://sonarcloud.io`
     - `SONAR_ORGANIZATION`: Your SonarCloud organization name.
     - `SONAR_PROJECT_KEY`: Your project key from SonarCloud.
     - `SONAR_TOKEN`: The token you generated.



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
- `SONAR_URL`, `SONAR_ORGANIZATION`, `SONAR_PROJECT_KEY`, `SONAR_TOKEN`: SonarCloud credentials.

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
   - Use GitHub Actions secrets to securely store sensitive information like AWS credentials and SonarCloud tokens.

3. **Image Security**:
   - Ensure Docker images are scanned for vulnerabilities before deployment.

4. **Network Security**:
   - Use private subnets and security groups to restrict access to the EKS cluster.

---

## Troubleshooting and Issue Resolution

### Common Issues and Solutions

1. **Docker Image Not Found in ECR**:
   - Ensure the ECR repository exists and the correct `REGISTRY` and `ECR_REPO_NAME` are configured in GitHub Secrets.
   - Verify that the image was successfully pushed to ECR:
     ```bash
     aws ecr describe-images --repository-name vprofile-repo --region us-east-2
     ```

2. **ArgoCD Sync Failures**:
   - Check the status of the ArgoCD application:
     ```bash
     kubectl get application custom-helm-app -n argocd -o wide
     ```
   - Review logs for debugging:
     ```bash
     kubectl logs -n argocd statefulset/argocd-application-controller --tail=100
     ```

3. **Pods Stuck in Pending State**:
   - Verify PersistentVolumeClaims (PVCs) are bound:
     ```bash
     kubectl get pvc
     ```
   - Check node availability and resource constraints.

4. **Ingress Host Not Resolving**:
   - Ensure the `ingress.host` in `values.yaml` matches your domain and DNS is configured correctly.
   - Example:
     ```yaml
     ingress:
       host: app.example.com
     ```

5. **AWS Authentication Errors**:
   - Ensure the `ROLE_ARN` is correctly configured in GitHub secrets.
   - Validate IAM role permissions for EKS and ECR.

---

##Verify

1.Verify if the images are getting pushed or not inside the AWS ECR, also check the tag name as well.

 ![image alt](https://github.com/Ashsatsan/gitops-argocd/blob/main/images/gitops6.png?raw=true)

2. Verify the pods if they are synced properly or not

 ![image alt](https://github.com/Ashsatsan/gitops-argocd/blob/main/images/gitops9.png?raw=true)

3. Lastly verify if the main application is up and running or not

![image alt](https://github.com/Ashsatsan/gitops-argocd/blob/main/images/gitops_resultwb.png?raw=true)
 
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

---

This updated `README.md` now includes a **Troubleshooting and Issue Resolution** section at the end and clarifies in the **Prerequisites** section why an ECR repository should be created manually outside of Terraform.
