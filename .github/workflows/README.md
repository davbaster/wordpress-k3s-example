# Second Part: Automated WordPress Deployment on Azure with Kubernetes and GitHub Actions

This project demonstrates a complete CI/CD pipeline for deploying and destroying a WordPress application on a K3s (Lightweight Kubernetes) cluster. The entire infrastructure is provisioned on-demand in Microsoft Azure and managed automatically by GitHub Actions.

## Overview

The core idea is to use Infrastructure as Code (IaC) and automation to create ephemeral environments for development, testing, or demonstration purposes. The workflow is entirely tag-driven:

* **To Deploy**: Push a git tag like `v1.0` to the repository. A GitHub Action will provision an Azure VM, install a K3s cluster, and deploy a WordPress site with a MySQL database.
* **To Destroy**: Push a git tag like `destroy-1.0`. A second GitHub Action will tear down all the created Azure resources, ensuring you only pay for what you use.

## Tech Stack

* **Cloud Provider**: Microsoft Azure
* **CI/CD**: GitHub Actions
* **Container Orchestration**: K3s (A lightweight, certified Kubernetes distribution, any YAML file created for Kubernetes will work on K3s)
* **Application**: WordPress
* **Database**: MySQL
* **Containerization**: Docker (implicitly via WordPress and MySQL images)

## Features

* **Fully Automated**: No manual steps required in the Azure Portal for deployment or destruction.
* **Cost-Effective**: Resources are created on-demand and can be destroyed easily, minimizing costs.
* **Secure**: Uses Azure Service Principals and GitHub Secrets to manage credentials securely, with no secrets stored in the repository.
* **Reproducible**: The entire environment is defined in code, ensuring consistency every time it's deployed.

## Prerequisites

Before you begin, ensure you have the following:

1.  **Azure Account**: A Microsoft Azure subscription. You can [create a free account](https://azure.microsoft.com/free/) if you don't have one.
2.  **GitHub Account**: A GitHub repository to host the project files and run GitHub Actions.
3.  **Azure CLI**: The [Azure Command-Line Interface](https://learn.microsoft.com/cli/azure/install-azure-cli) installed and configured on your local machine (`az login`). However you can also use the Azure Cloud shell to execute the commands directly in Azure Portal.

## Setup Instructions

### Step 1: Configure Azure Credentials

To allow GitHub Actions to manage resources in your Azure subscription, you need to create a **Service Principal**.

1.  Log in to Azure through the CLI:
    ```bash
    az login
    ```
2.  Set the subscription you want to use:
    ```bash
    az account set --subscription "YOUR_SUBSCRIPTION_NAME_OR_ID"
    ```
3.  Create the Service Principal with the `Contributor` role. This command will output a JSON object.

    ```bash
    az ad sp create-for-rbac --name "github-wordpress-deploy" --role "Contributor" --scopes "/subscriptions/YOUR_SUBSCRIPTION_NAME_OR_ID" --sdk-auth
    ```

### Step 2: Configure GitHub Secrets

You will use the output from the previous step and other credentials to create secrets in your GitHub repository. Go to **Settings > Secrets and variables > Actions** and add the following repository secrets:

1.  **`AZURE_CREDENTIALS`**:
    * Copy the **entire JSON output** from the `az ad sp create-for-rbac` command and paste it here.

2.  **`MYSQL_ROOT_PASSWORD`**:
    * The desired root password for your MySQL database. Example: `YourSecureRootPassword123`

3.  **`MYSQL_PASSWORD`**:
    * The password for the `wordpress` user that the application will use to connect to the database. Example: `YourSecureWordpressPassword123`

## File Structure

Your repository should have the following structure:
```
├── .github/
│   └── workflows/
│       ├── deploy.yml      # Workflow to create the infrastructure and deploy WordPress
│       └── destroy.yml     # Workflow to tear down all resources
│
└── k8s/
├── 01-mysql-deployment.yml     # Manifests for MySQL (PVC, Service, Deployment)
├── 02-wordpress-deployment.yml # Manifests for WordPress (Service, Deployment)
└── 03-wordpress-ingress.yml    # Ingress rule to expose WordPress
```
## How It Works

### Deployment Workflow (`deploy.yml`)
This workflow is triggered by a push of a tag starting with `v` (e.g., `v1.0`, `v1.1`). It performs the following steps:
1.  Logs into Azure using the `AZURE_CREDENTIALS` secret.
2.  Creates a resource group to contain all the infrastructure.
3.  Provisions an Azure Virtual Machine (`Standard_B2s` running Ubuntu LTS).
4.  Opens port 80 on the VM to allow public web traffic.
5.  Installs the K3s Kubernetes distribution on the VM.
6.  Retrieves the `kubeconfig` file from the VM and configures it to be used by `kubectl` from within the GitHub Actions runner.
7.  Dynamically creates a Kubernetes secret (`mysql-secrets`) using the `MYSQL_` secrets stored in GitHub.
8.  Deploys the WordPress and MySQL manifests from the `k8s/` directory to the cluster.
9.  Outputs the public IP address of the VM to access the new WordPress site.

### Destruction Workflow (`destroy.yml`)
This workflow is triggered by a push of a tag starting with `destroy-` (e.g., `destroy-1.0`). Its job is simple:
1.  Logs into Azure.
2.  Issues a single command to **delete the entire resource group**. This action is irreversible and removes the VM, disk, public IP, and all associated resources, immediately stopping all costs.

## Usage

### To Deploy WordPress

1.  Commit and push your code to the repository's main branch.
    ```bash
    git add .
    git commit -m "Setup WordPress deployment project"
    git push origin main
    ```
2.  Create and push a `v*` tag.
    ```bash
    git tag v1.0
    git push origin v1.0
    ```
3.  Go to the **Actions** tab in your GitHub repository to monitor the `Deploy WordPress...` workflow. When it's complete, check the logs for the public IP address.

### To Destroy the Infrastructure

1.  Create and push a `destroy-*` tag.
    ```bash
    git tag destroy-1.0
    git push origin destroy-1.0
    ```
2.  Go to the **Actions** tab to watch the `Destroy Azure Infrastructure` workflow run. It will complete quickly, but the resource deletion will continue in the background in your Azure account.
