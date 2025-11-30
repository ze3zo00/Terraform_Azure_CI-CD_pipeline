# Complete Guide to Terraform Pipelines with Azure and GitLab

Automating your Azure infrastructure with a Terraform pipeline in GitLab transforms manual deployments into consistent, auditable, team-collaborative workflows. This guide walks you through the entire process—from creating your first GitLab project to running production-grade infrastructure pipelines—with every command and configuration explained for someone new to GitLab.

A well-designed Terraform pipeline eliminates the "it works on my machine" problem by ensuring every infrastructure change runs in the same environment, follows the same validation steps, and leaves a complete audit trail. By the end of this guide, you'll have a working pipeline that validates your Terraform code, shows you exactly what will change before anything happens, requires approval for production deployments, and stores state securely in Azure.

---

## Table of Contents

1. [Why Pipeline Automation Matters for Azure Infrastructure](#why-pipeline-automation-matters-for-azure-infrastructure)
2. [Understanding GitLab Runners for Enterprise Deployments](#understanding-gitlab-runners-for-enterprise-deployments)
   - [What is a GitLab Runner?](#what-is-a-gitlab-runner)
   - [Shared Runners vs Self-Hosted Runners](#shared-runners-vs-self-hosted-runners)
     - [Shared Runners (GitLab.com SaaS)](#shared-runners-gitlabcom-saas)
     - [Self-Hosted Runners (Enterprise Standard)](#self-hosted-runners-enterprise-standard)
   - [Runner Executor Types](#runner-executor-types)
   - [Enterprise Runner Architecture Patterns](#enterprise-runner-architecture-patterns)
     - [Pattern 1: Dedicated Runner VMs in Azure](#pattern-1-dedicated-runner-vms-in-azure)
     - [Pattern 2: Kubernetes-Based Runners (AKS)](#pattern-2-kubernetes-based-runners-aks)
     - [Pattern 3: Hybrid - Shared Runners + Self-Hosted for Sensitive Jobs](#pattern-3-hybrid---shared-runners--self-hosted-for-sensitive-jobs)
   - [Installing a Self-Hosted GitLab Runner on Azure](#installing-a-self-hosted-gitlab-runner-on-azure)
     - [Step 1: Create the Runner VM](#step-1-create-the-runner-vm)
     - [Step 2: Install Docker on the VM](#step-2-install-docker-on-the-vm)
     - [Step 3: Install GitLab Runner](#step-3-install-gitlab-runner)
     - [Step 4: Register the Runner with GitLab](#step-4-register-the-runner-with-gitlab)
     - [Step 5: Configure Runner for Enterprise Requirements](#step-5-configure-runner-for-enterprise-requirements)
     - [Step 6: Start and Enable the Runner](#step-6-start-and-enable-the-runner)
   - [Securing Self-Hosted Runners](#securing-self-hosted-runners)
     - [Network Security](#network-security)
     - [Runner Authentication with Azure](#runner-authentication-with-azure)
     - [Least Privilege for Docker Executor](#least-privilege-for-docker-executor)
   - [Routing Jobs to Specific Runners](#routing-jobs-to-specific-runners)
   - [Monitoring and Maintenance](#monitoring-and-maintenance)
     - [Health Checks](#health-checks)
     - [Log Management](#log-management)
   - [Runner Scaling Strategies](#runner-scaling-strategies)
     - [Manual Scaling](#manual-scaling)
     - [Auto-Scaling with Docker Machine (Advanced)](#auto-scaling-with-docker-machine-advanced)
   - [Decision Matrix: Which Runner Setup for Your Enterprise?](#decision-matrix-which-runner-setup-for-your-enterprise)
3. [Prerequisites and Requirements](#prerequisites-and-requirements)
   - [Azure Requirements](#azure-requirements)
   - [Create Azure Storage for Terraform State](#create-azure-storage-for-terraform-state)
   - [GitLab Requirements](#gitlab-requirements)
   - [Required Tools for Local Development](#required-tools-for-local-development)
4. [Step-by-Step Implementation Guide](#step-by-step-implementation-guide)
   - [Step 1: Set Up Your GitLab Project Structure](#step-1-set-up-your-gitlab-project-structure)
   - [Step 2: Configure the Terraform Provider and Backend](#step-2-configure-the-terraform-provider-and-backend)
   - [Step 3: Add Azure Credentials to GitLab](#step-3-add-azure-credentials-to-gitlab)
   - [Step 4: Create Your First Terraform Configuration](#step-4-create-your-first-terraform-configuration)
   - [Step 5: Create the GitLab CI/CD Pipeline](#step-5-create-the-gitlab-cicd-pipeline)
   - [Step 6: Commit and Push Your Code](#step-6-commit-and-push-your-code)
   - [Step 7: Run Your First Pipeline](#step-7-run-your-first-pipeline)
5. [Multi-Environment Pipeline Configuration](#multi-environment-pipeline-configuration)
   - [Directory Structure for Multiple Environments](#directory-structure-for-multiple-environments)
   - [Environment-Specific Variable Files](#environment-specific-variable-files)
   - [Multi-Environment Pipeline](#multi-environment-pipeline)
6. [Sample Terraform Code for Common Azure Resources](#sample-terraform-code-for-common-azure-resources)
   - [Virtual Network with Subnets](#virtual-network-with-subnets)
   - [Storage Account with Security Settings](#storage-account-with-security-settings)
   - [Using Local Values for Consistent Tagging](#using-local-values-for-consistent-tagging)
7. [Common Pitfalls and Troubleshooting](#common-pitfalls-and-troubleshooting)
   - [Authentication Failures](#authentication-failures)
   - [State Locking Errors](#state-locking-errors)
   - [GitLab Runner Issues](#gitlab-runner-issues)
   - [Variables Not Available in Jobs](#variables-not-available-in-jobs)
   - [Backend Initialization Failures](#backend-initialization-failures)
   - [Debugging Failed Pipelines](#debugging-failed-pipelines)
   - [Quick Reference for Error Resolution](#quick-reference-for-error-resolution)
8. [Best Practices for Production Pipelines](#best-practices-for-production-pipelines)
9. [Conclusion](#conclusion)

---

## Why Pipeline Automation Matters for Azure Infrastructure

Running Terraform locally works fine for learning, but production infrastructure demands more rigor. Pipeline automation solves five critical problems that manual execution cannot address.

First, **consistency becomes automatic**. Every deployment uses the same Terraform version, provider versions, and execution environment. No more debugging why something worked on your laptop but failed when your colleague ran it.

Second, **credentials stay secure**. Service principal secrets live in GitLab's encrypted variable storage, not in shell history or configuration files on developer machines. When credentials need rotation, you update one place instead of notifying the entire team.

Third, **collaboration improves dramatically**. Infrastructure changes go through merge requests just like application code. Team members review Terraform plans before they execute, catching misconfigured resources or unintended deletions before they cause outages.

Fourth, **audit trails emerge naturally**. Git history shows who changed what and when. Pipeline logs capture exactly which Terraform commands ran and what they output. Compliance reviews become straightforward because everything is recorded.

Fifth, **human error decreases**. The pipeline enforces validation, formatting checks, and approval gates. You cannot accidentally run `terraform apply` against production because you forgot to switch contexts—the pipeline handles environment selection automatically.

---

## Understanding GitLab Runners for Enterprise Deployments

Before diving into prerequisites, it's essential to understand **GitLab Runners**—the execution engines that actually run your pipeline jobs. This is especially critical for enterprise environments where security, network access, and compliance requirements shape your infrastructure decisions.

### What is a GitLab Runner?

Think of a GitLab Runner as the "worker" that executes your pipeline jobs. When you push code to GitLab and trigger a pipeline:

1. GitLab (the server) reads your `.gitlab-ci.yml` file
2. GitLab creates jobs and queues them
3. A GitLab Runner picks up the job
4. The runner executes the commands (like `terraform plan`) in an isolated environment
5. The runner sends results back to GitLab

The key question for enterprises: **Where does this runner execute, and what can it access?**

### Shared Runners vs Self-Hosted Runners

GitLab offers two runner deployment models, each with different implications for enterprise use:

#### Shared Runners (GitLab.com SaaS)

**What they are:** Cloud-based runners provided by GitLab on their infrastructure. When you use GitLab.com (the SaaS version), you get access to these for free within usage limits.

**Advantages:**
- Zero infrastructure to manage
- No setup required—works immediately
- Automatically scaled and maintained by GitLab
- Free tier includes 400 compute minutes/month

**Limitations for Enterprise:**
- Jobs run on GitLab's infrastructure (outside your network perimeter)
- Cannot access private Azure resources without public endpoints
- Limited control over execution environment
- Usage limits may be insufficient for large teams
- Compliance concerns—your code and credentials exist outside your infrastructure

**When to use:** Small teams, public cloud resources only, no strict compliance requirements, proof-of-concept work.

#### Self-Hosted Runners (Enterprise Standard)

**What they are:** Runner software you install and manage on your own infrastructure—whether that's Azure VMs, on-premises servers, Kubernetes clusters, or container platforms.

**Advantages:**
- Full control over execution environment
- Runners operate inside your network perimeter
- Can access private VNets, internal APIs, and on-premises systems
- No usage limits beyond your infrastructure capacity
- Meets compliance requirements for data sovereignty
- Custom configurations (specific tools, libraries, caching strategies)

**Considerations:**
- You manage the infrastructure (VMs, updates, scaling)
- Initial setup and configuration required
- Ongoing maintenance responsibility

**When to use:** Enterprise deployments, private Azure resources, compliance requirements, high-volume pipelines, need for specialized tools or configurations.

### Runner Executor Types

Self-hosted runners support multiple **executor** types that determine how jobs run:

| Executor | Description | Best For | Enterprise Use Case |
|----------|-------------|----------|---------------------|
| **Docker** | Runs each job in a fresh Docker container | Most CI/CD workloads | Standard choice—isolates jobs, easy cleanup, consistent environments |
| **Kubernetes** | Runs jobs as pods in a Kubernetes cluster | Large scale, dynamic scaling | High-volume pipelines, teams already on AKS |
| **Shell** | Runs jobs directly on the runner's host OS | Legacy systems, specific OS requirements | Windows-specific tasks, specialized tools |
| **Docker Machine** | Auto-scales Docker hosts on demand | Variable load | Cost optimization—scales up during work hours, down at night |

For Terraform pipelines with Azure, **Docker executor** is the recommended choice for most enterprises because:
- Each job gets a clean container with the exact Terraform version you specify
- No state pollution between jobs
- Easy to reproduce issues locally
- Simple cleanup—containers are destroyed after each job

### Enterprise Runner Architecture Patterns

Here are three common enterprise deployment patterns:

#### Pattern 1: Dedicated Runner VMs in Azure

```
┌─────────────────────────────────────────────────┐
│ Azure Subscription (Enterprise)                 │
│                                                  │
│  ┌──────────────┐         ┌──────────────┐     │
│  │ Runner VM 1  │         │ Runner VM 2  │     │
│  │ (Linux)      │         │ (Linux)      │     │
│  │              │         │              │     │
│  │ Docker       │         │ Docker       │     │
│  │ Executor     │         │ Executor     │     │
│  └──────┬───────┘         └──────┬───────┘     │
│         │                        │              │
│         └────────┬───────────────┘              │
│                  │                               │
│         Connected to VNet                        │
│         Can access private resources             │
└─────────┬────────────────────────────────────────┘
          │
          │ Polls GitLab for jobs
          │ (outbound HTTPS only)
          ▼
┌─────────────────────┐
│ GitLab.com or       │
│ Self-Hosted GitLab  │
└─────────────────────┘
```

**Advantages:** Simple, predictable, works with existing VM management processes.

**Setup:** Deploy 2-3 Linux VMs, install GitLab Runner, configure tags for job routing.

#### Pattern 2: Kubernetes-Based Runners (AKS)

```
┌─────────────────────────────────────────────────┐
│ Azure Kubernetes Service (AKS)                  │
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │ GitLab Runner Pod (Orchestrator)       │    │
│  │                                         │    │
│  │  Creates job pods on demand:           │    │
│  │                                         │    │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  │    │
│  │  │ Job 1  │  │ Job 2  │  │ Job 3  │  │    │
│  │  │ Pod    │  │ Pod    │  │ Pod    │  │    │
│  │  └────────┘  └────────┘  └────────┘  │    │
│  └────────────────────────────────────────┘    │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Advantages:** Auto-scaling, efficient resource utilization, native Kubernetes integration.

**Setup:** Deploy runner using Helm chart to AKS cluster.

#### Pattern 3: Hybrid - Shared Runners + Self-Hosted for Sensitive Jobs

```
┌──────────────────────────────────────────┐
│ GitLab Pipeline                          │
│                                          │
│  Validate Job ─────► Shared Runner      │  (Public cloud)
│                      (GitLab.com)        │
│                                          │
│  Plan Job     ─────► Self-Hosted Runner │  (Your Azure)
│                      (Private access)    │
│                                          │
│  Apply Job    ─────► Self-Hosted Runner │  (Your Azure)
│                      (Credentials safe)  │
└──────────────────────────────────────────┘
```

**Advantages:** Cost optimization, security for sensitive operations, leverage free tier for validation.

**Setup:** Use job tags to route sensitive jobs to self-hosted runners.

### Installing a Self-Hosted GitLab Runner on Azure

Here's the complete process for deploying a self-hosted runner on an Azure Linux VM.

#### Step 1: Create the Runner VM

```bash
# Create resource group for runners
az group create --name gitlab-runners-rg --location eastus

# Create a Linux VM for the runner
az vm create \
  --resource-group gitlab-runners-rg \
  --name gitlab-runner-01 \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# Get the public IP for SSH access
az vm show -d -g gitlab-runners-rg -n gitlab-runner-01 --query publicIps -o tsv
```

#### Step 2: Install Docker on the VM

SSH into your VM and install Docker:

```bash
# SSH into the VM
ssh azureuser@<VM_PUBLIC_IP>

# Update package index
sudo apt-get update

# Install Docker
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verify Docker installation
sudo docker run hello-world

# Add current user to docker group (optional, for convenience)
sudo usermod -aG docker $USER
```

#### Step 3: Install GitLab Runner

```bash
# Download the GitLab Runner binary
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

# Install GitLab Runner
sudo apt-get install gitlab-runner

# Verify installation
gitlab-runner --version
```

#### Step 4: Register the Runner with GitLab

You need a **registration token** from GitLab. Get it from:
- **GitLab.com:** Your project → Settings → CI/CD → Runners → Expand
- **Self-Hosted GitLab:** Admin Area → Runners

```bash
# Register the runner (interactive)
sudo gitlab-runner register

# You'll be prompted for:
# 1. GitLab instance URL: https://gitlab.com (or your self-hosted URL)
# 2. Registration token: [paste from GitLab UI]
# 3. Runner description: azure-terraform-runner-01
# 4. Runner tags: azure,terraform,docker (comma-separated)
# 5. Executor: docker
# 6. Default Docker image: hashicorp/terraform:1.10.5
```

**Example interactive session:**

```
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
https://gitlab.com

Please enter the gitlab-ci token for this runner:
<your-token-here>

Please enter the gitlab-ci description for this runner:
azure-terraform-runner-01

Please enter the gitlab-ci tags for this runner (comma separated):
azure,terraform,docker

Please enter the executor: docker, shell, ssh, docker+machine, kubernetes:
docker

Please enter the default Docker image (e.g. ruby:2.6):
hashicorp/terraform:1.10.5

Runner registered successfully. Feel free to start it!
```

#### Step 5: Configure Runner for Enterprise Requirements

Edit the runner configuration file:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

Modify the configuration for production use:

```toml
concurrent = 4  # Number of jobs that can run simultaneously

check_interval = 0  # How often to check GitLab for new jobs (0 = default)

[session_server]
  session_timeout = 1800  # Session timeout for interactive debugging

[[runners]]
  name = "azure-terraform-runner-01"
  url = "https://gitlab.com"
  token = "<runner-token>"
  executor = "docker"
  
  [runners.docker]
    tls_verify = false
    image = "hashicorp/terraform:1.10.5"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    
  [runners.cache]
    Type = "local"
    Path = "/cache"
    Shared = true
    
    [runners.cache.local]
      BaseURL = "http://localhost:9000"
```

**Key configuration options explained:**

- **concurrent**: How many jobs run in parallel (match to VM size)
- **volumes**: Mount Docker socket for Docker-in-Docker scenarios
- **cache**: Local cache shared between jobs for faster execution

#### Step 6: Start and Enable the Runner

```bash
# Start the runner
sudo gitlab-runner start

# Enable auto-start on boot
sudo systemctl enable gitlab-runner

# Verify runner status
sudo gitlab-runner status

# View runner logs
sudo journalctl -u gitlab-runner -f
```

### Securing Self-Hosted Runners

Enterprise runners need additional security hardening:

#### Network Security

```bash
# Create Network Security Group rules
az network nsg rule create \
  --resource-group gitlab-runners-rg \
  --nsg-name gitlab-runner-01NSG \
  --name AllowSSH \
  --priority 1000 \
  --source-address-prefixes '<your-corporate-ip-range>' \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

# Deny all other inbound traffic (rule usually exists by default)
az network nsg rule create \
  --resource-group gitlab-runners-rg \
  --nsg-name gitlab-runner-01NSG \
  --name DenyAllInbound \
  --priority 4096 \
  --source-address-prefixes '*' \
  --destination-port-ranges '*' \
  --access Deny \
  --protocol '*'
```

**Note:** GitLab Runners only need **outbound** HTTPS (443) access to poll GitLab for jobs. They don't require inbound access from GitLab.

#### Runner Authentication with Azure

Give the runner's VM a **Managed Identity** so it can authenticate to Azure without storing credentials:

```bash
# Enable system-assigned managed identity on the VM
az vm identity assign \
  --resource-group gitlab-runners-rg \
  --name gitlab-runner-01

# Get the managed identity's principal ID
PRINCIPAL_ID=$(az vm identity show \
  --resource-group gitlab-runners-rg \
  --name gitlab-runner-01 \
  --query principalId -o tsv)

# Grant Contributor access to the subscription
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role Contributor \
  --scope /subscriptions/<SUBSCRIPTION_ID>
```

Now modify your pipeline to use managed identity instead of service principal:

```yaml
# .gitlab-ci.yml
plan:
  stage: plan
  tags:
    - azure
    - terraform
  before_script:
    - cd ${TF_ROOT}
    - export ARM_USE_MSI=true
    - export ARM_SUBSCRIPTION_ID=$ARM_SUBSCRIPTION_ID
    - export ARM_TENANT_ID=$ARM_TENANT_ID
    - terraform init
  script:
    - terraform plan -out=tfplan
```

This eliminates the need to store `ARM_CLIENT_ID` and `ARM_CLIENT_SECRET` in GitLab variables.

#### Least Privilege for Docker Executor

Run the GitLab Runner service with minimal privileges:

```bash
# Edit runner configuration to use unprivileged containers
sudo nano /etc/gitlab-runner/config.toml

# Set privileged = false (should already be false)
[runners.docker]
  privileged = false
```

### Routing Jobs to Specific Runners

Use **tags** to control which runners execute which jobs. This is critical in enterprises with multiple runner types.

**Tag your runners during registration:**

```bash
# Terraform-specific runner
Tags: terraform,azure,docker,prod

# Development runner  
Tags: terraform,azure,docker,dev

# Windows runner for special cases
Tags: terraform,azure,shell,windows
```

**Route jobs using tags in your pipeline:**

```yaml
plan:prod:
  stage: plan
  tags:
    - terraform
    - azure
    - prod  # Only runs on production runners
  script:
    - terraform plan
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

plan:dev:
  stage: plan
  tags:
    - terraform
    - azure
    - dev  # Only runs on development runners
  script:
    - terraform plan
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
```

### Monitoring and Maintenance

#### Health Checks

Create a script to verify runner health:

```bash
#!/bin/bash
# /usr/local/bin/check-runner-health.sh

# Check if runner service is running
systemctl is-active --quiet gitlab-runner || exit 1

# Check if Docker is running
systemctl is-active --quiet docker || exit 1

# Check disk space (alert if >80% full)
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
  echo "WARNING: Disk usage is ${DISK_USAGE}%"
  exit 1
fi

echo "Runner health: OK"
exit 0
```

Schedule this with cron or Azure Monitor:

```bash
# Add to crontab
*/5 * * * * /usr/local/bin/check-runner-health.sh
```

#### Log Management

Configure log rotation to prevent disk exhaustion:

```bash
sudo nano /etc/logrotate.d/gitlab-runner
```

Add:

```
/var/log/gitlab-runner/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifexist
    copytruncate
}
```

### Runner Scaling Strategies

As your pipeline usage grows, you'll need to scale runners:

#### Manual Scaling

Deploy additional runner VMs and register them with the same tags:

```bash
# Create runner VM 2
az vm create \
  --resource-group gitlab-runners-rg \
  --name gitlab-runner-02 \
  --image Ubuntu2204 \
  --size Standard_D2s_v3

# Install Docker and GitLab Runner (same steps as above)
# Register with identical tags
```

GitLab automatically load-balances jobs across runners with matching tags.

#### Auto-Scaling with Docker Machine (Advanced)

For variable workloads, use GitLab's Docker Machine executor to auto-scale:

```toml
[[runners]]
  name = "azure-autoscale-runner"
  url = "https://gitlab.com"
  token = "..."
  executor = "docker+machine"
  
  [runners.machine]
    IdleCount = 1
    IdleTime = 600
    MaxBuilds = 100
    MachineDriver = "azure"
    MachineName = "gitlab-runner-%s"
    MachineOptions = [
      "azure-subscription-id=...",
      "azure-size=Standard_D2s_v3",
      "azure-location=eastus"
    ]
```

This creates VMs on-demand when jobs queue and destroys them after idle time.

### Decision Matrix: Which Runner Setup for Your Enterprise?

| Requirement | Recommended Setup |
|-------------|-------------------|
| Small team (<10 people), public cloud only | GitLab.com shared runners |
| Private Azure resources, moderate usage | 2-3 dedicated VMs with Docker executor |
| High volume (>100 jobs/day), variable load | AKS with Kubernetes executor or Docker Machine auto-scaling |
| Windows-specific tasks | Dedicated Windows VM with shell executor |
| Maximum security, compliance requirements | VM in private subnet with managed identity, no internet access except GitLab |
| Multi-region deployments | Runners in each region with regional tags |

---

## Prerequisites and Requirements

Before building your pipeline, you need accounts and access configured across three platforms. This section covers everything you need—gather these items before proceeding.

### Azure Requirements

You need an Azure subscription where you have permission to create a service principal with Contributor access. The service principal authenticates Terraform to Azure, and you'll also create a storage account to hold Terraform's state file.

**Create a service principal using Azure CLI:**

```bash
# Login to Azure (if not already authenticated)
az login

# Get your subscription ID
az account show --query id -o tsv

# Create a service principal with Contributor role
az ad sp create-for-rbac \
  --name "terraform-gitlab-pipeline" \
  --role Contributor \
  --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID>
```

This command outputs four values you must save securely:

```json
{
  "appId": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx",     
  "displayName": "terraform-gitlab-pipeline",
  "password": "xxxxxx~xxxxxx~xxxxx",              
  "tenant": "xxxxx-xxxx-xxxx-xxxx-xxxxx"         
}
```

**Map these values to Terraform variables:**
- `appId` → `ARM_CLIENT_ID`
- `password` → `ARM_CLIENT_SECRET`
- `tenant` → `ARM_TENANT_ID`
- Your subscription ID → `ARM_SUBSCRIPTION_ID`

The **Contributor role** allows Terraform to create, modify, and delete most Azure resources within the subscription. For tighter security, scope the service principal to specific resource groups:

```bash
az ad sp create-for-rbac \
  --name "terraform-pipeline" \
  --role Contributor \
  --scopes /subscriptions/<SUB_ID>/resourceGroups/my-resource-group
```

### Create Azure Storage for Terraform State

Terraform tracks the resources it manages in a state file. For CI/CD pipelines, this state must be stored remotely so multiple pipeline runs can access it, and it must support locking to prevent concurrent modifications.

```bash
# Set variables
RESOURCE_GROUP_NAME=terraform-state-rg
STORAGE_ACCOUNT_NAME=tfstate$RANDOM  # Must be globally unique
CONTAINER_NAME=tfstate
LOCATION=eastus

# Create resource group
az group create --name $RESOURCE_GROUP_NAME --location $LOCATION

# Create storage account with security settings
az storage account create \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $STORAGE_ACCOUNT_NAME \
  --sku Standard_LRS \
  --encryption-services blob \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

# Create blob container
az storage container create \
  --name $CONTAINER_NAME \
  --account-name $STORAGE_ACCOUNT_NAME

# Note the storage account name - you'll need it later
echo "Storage account: $STORAGE_ACCOUNT_NAME"
```

**Grant the service principal access to the state storage:**

```bash
# Get the service principal's object ID
SP_OBJECT_ID=$(az ad sp show --id <APP_ID> --query id -o tsv)

# Assign Storage Blob Data Contributor role
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id $SP_OBJECT_ID \
  --scope /subscriptions/<SUB_ID>/resourceGroups/$RESOURCE_GROUP_NAME/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT_NAME
```

### GitLab Requirements

Create a free account at [gitlab.com](https://gitlab.com) if you don't have one. GitLab.com provides shared runners that execute your pipelines at no cost for the free tier, meaning you don't need to set up any infrastructure to run CI/CD jobs.

**Create a new project:**
1. Click **New project** → **Create blank project**
2. Enter a project name (e.g., `azure-infrastructure`)
3. Set visibility to **Private** (recommended for infrastructure code)
4. Initialize with a README (optional but helpful)
5. Click **Create project**

### Required Tools for Local Development

Install these tools on your development machine for writing and testing Terraform code locally:

| Tool | Version | Purpose |
|------|---------|---------|
| **Terraform** | 1.9+ (1.10.5 recommended) | Infrastructure provisioning |
| **Azure CLI** | Latest | Azure authentication, service principal management |
| **Git** | Latest | Version control |

Verify installations:

```bash
terraform --version    # Should show 1.9.0 or higher
az --version          # Should show 2.x
git --version
```

---

## Step-by-Step Implementation Guide

This section walks through building a complete Terraform pipeline from scratch. Follow these steps in order—each builds on the previous.

### Step 1: Set Up Your GitLab Project Structure

Clone your GitLab project and create the following directory structure:

```
azure-infrastructure/
├── .gitlab-ci.yml           # Pipeline configuration
├── terraform/
│   ├── main.tf              # Main resource definitions
│   ├── variables.tf         # Variable declarations
│   ├── outputs.tf           # Output values
│   ├── providers.tf         # Provider configuration
│   ├── backend.tf           # State backend configuration
│   └── terraform.tfvars     # Variable values (non-sensitive)
└── README.md
```

Create the directories:

```bash
git clone <your-gitlab-project-url>
cd azure-infrastructure
mkdir -p terraform
```

### Step 2: Configure the Terraform Provider and Backend

Create `terraform/providers.tf` to configure the Azure provider:

```hcl
terraform {
  required_version = ">= 1.9.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
  
  # Authentication via environment variables:
  # ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_SUBSCRIPTION_ID, ARM_TENANT_ID
  # These are set as CI/CD variables in GitLab
}
```

Create `terraform/backend.tf` for remote state storage. Use partial configuration so credentials pass through GitLab CI/CD:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "YOUR_STORAGE_ACCOUNT_NAME"  # Replace with actual name
    container_name       = "tfstate"
    key                  = "infrastructure.tfstate"
    use_azuread_auth     = true
  }
}
```

### Step 3: Add Azure Credentials to GitLab

Navigate to your project in GitLab: **Settings → CI/CD → Variables** (expand the Variables section).

Add each variable by clicking **Add variable**:

| Key | Value | Protected | Masked |
|-----|-------|-----------|--------|
| `ARM_CLIENT_ID` | Your service principal appId | ✓ | ✓ |
| `ARM_CLIENT_SECRET` | Your service principal password | ✓ | ✓ |
| `ARM_SUBSCRIPTION_ID` | Your Azure subscription ID | ✓ | ✓ |
| `ARM_TENANT_ID` | Your Azure tenant ID | ✓ | ✓ |

**Understanding the checkboxes:**
- **Protected**: Variable only available to pipelines running on protected branches (like `main`). Enable this for production credentials.
- **Masked**: Value hidden in job logs, shown as `[MASKED]`. Enable for all secrets.

For the `ARM_CLIENT_SECRET`, also check **Hidden** (available in GitLab 17.4+) so the value cannot be viewed after creation.

### Step 4: Create Your First Terraform Configuration

Create `terraform/variables.tf`:

```hcl
variable "resource_group_name" {
  description = "Name of the Azure Resource Group"
  type        = string
  default     = "terraform-demo-rg"
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "eastus"
}

variable "environment" {
  description = "Environment name for tagging"
  type        = string
  default     = "dev"
}
```

Create `terraform/main.tf` with a simple resource group to test the pipeline:

```hcl
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Source      = "GitLab-Pipeline"
  }
}
```

Create `terraform/outputs.tf`:

```hcl
output "resource_group_id" {
  description = "ID of the created resource group"
  value       = azurerm_resource_group.main.id
}

output "resource_group_name" {
  description = "Name of the created resource group"
  value       = azurerm_resource_group.main.name
}
```

### Step 5: Create the GitLab CI/CD Pipeline

Create `.gitlab-ci.yml` in your project root. This is the complete pipeline configuration:

```yaml
# Terraform Pipeline for Azure Infrastructure
# This pipeline validates, plans, and applies Terraform configurations

image:
  name: hashicorp/terraform:1.10.5
  entrypoint: [""]  # Required for shell access in GitLab CI

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/terraform
  # Disable interactive prompts
  TF_INPUT: "false"
  # Reduce log noise
  TF_IN_AUTOMATION: "true"

stages:
  - validate
  - plan
  - apply

# Cache Terraform plugins between pipeline runs
cache:
  key: terraform-plugins-${CI_COMMIT_REF_SLUG}
  paths:
    - ${TF_ROOT}/.terraform

# Template for common Terraform initialization
.terraform-init: &terraform-init
  before_script:
    - cd ${TF_ROOT}
    - terraform --version
    - terraform init -reconfigure

# STAGE 1: Validate
# Checks Terraform syntax and configuration validity
validate:
  stage: validate
  <<: *terraform-init
  script:
    - terraform fmt -check -recursive -diff
    - terraform validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# STAGE 2: Plan
# Shows what changes Terraform will make
plan:
  stage: plan
  <<: *terraform-init
  script:
    - terraform plan -out=tfplan
    - terraform show -json tfplan > tfplan.json
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
      - ${TF_ROOT}/tfplan.json
    reports:
      terraform: ${TF_ROOT}/tfplan.json
    expire_in: 7 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# STAGE 3: Apply
# Executes the planned changes (manual approval required)
apply:
  stage: apply
  <<: *terraform-init
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan
  when: manual
  allow_failure: false
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  environment:
    name: production
```

**What each section does:**

The **image** block specifies the Docker container where jobs run. HashiCorp's official Terraform image includes the CLI. The `entrypoint: [""]` override is required because the default entrypoint doesn't work with GitLab's shell executor.

The **variables** block sets configuration that applies to all jobs. `TF_INPUT: "false"` prevents Terraform from waiting for user input that never comes in CI/CD.

The **stages** block defines the execution order. Jobs in the same stage run in parallel; stages run sequentially.

The **cache** block stores downloaded Terraform providers between pipeline runs, speeding up initialization from minutes to seconds.

The **validate** job runs `terraform fmt -check` (fails if code isn't formatted) and `terraform validate` (checks for syntax errors).

The **plan** job creates an execution plan and saves it as an artifact. The `reports: terraform:` line enables GitLab's merge request widget that shows resource changes.

The **apply** job executes the saved plan. `when: manual` means someone must click a button to start it—this is your approval gate. `allow_failure: false` means the pipeline shows as blocked until approved.

### Step 6: Commit and Push Your Code

```bash
git add .
git commit -m "Add Terraform pipeline for Azure infrastructure"
git push origin main
```

After pushing, navigate to **CI/CD → Pipelines** in GitLab. You should see a pipeline running with three stages.

### Step 7: Run Your First Pipeline

The validate and plan stages run automatically. Watch them in the GitLab UI:

1. Click on the pipeline to see job details
2. Click on **validate** to see the job log
3. Click on **plan** to see what Terraform will create

The plan output shows something like:

```
Terraform will perform the following actions:

  # azurerm_resource_group.main will be created
  + resource "azurerm_resource_group" "main" {
      + id       = (known after apply)
      + location = "eastus"
      + name     = "terraform-demo-rg"
      + tags     = {
          + "Environment" = "dev"
          + "ManagedBy"   = "Terraform"
          + "Source"      = "GitLab-Pipeline"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**To apply the changes:**
1. Go to the pipeline view
2. Find the **apply** job (it shows a play button because it's manual)
3. Click the play button
4. Confirm by clicking **Trigger this manual action**

The apply job runs and creates your resource group in Azure.

---

## Multi-Environment Pipeline Configuration

Real infrastructure typically spans multiple environments—development, staging, and production. This section shows how to structure your pipeline for environment-specific deployments.

### Directory Structure for Multiple Environments

Organize your Terraform code with separate directories per environment:

```
azure-infrastructure/
├── .gitlab-ci.yml
├── modules/                    # Shared, reusable modules
│   └── resource-group/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── terraform.tfvars
    ├── staging/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── backend.tf
        └── terraform.tfvars
```

Each environment has its own state file (defined in `backend.tf`) and configuration values (in `terraform.tfvars`). The shared module in `modules/` contains reusable resource definitions.

### Environment-Specific Variable Files

`environments/dev/terraform.tfvars`:
```hcl
environment         = "dev"
resource_group_name = "myapp-dev-rg"
location            = "eastus"
vm_size             = "Standard_B1s"
instance_count      = 1
```

`environments/prod/terraform.tfvars`:
```hcl
environment         = "prod"
resource_group_name = "myapp-prod-rg"
location            = "eastus"
vm_size             = "Standard_D2s_v3"
instance_count      = 3
```

### Multi-Environment Pipeline

Here's a complete `.gitlab-ci.yml` that deploys different environments based on branches:

```yaml
image:
  name: hashicorp/terraform:1.10.5
  entrypoint: [""]

variables:
  TF_INPUT: "false"
  TF_IN_AUTOMATION: "true"

stages:
  - validate
  - plan
  - apply

# YAML anchor for reusable init template
.terraform-init-template: &init-template
  before_script:
    - cd ${TF_ROOT}
    - terraform init -reconfigure

# ============== DEV ENVIRONMENT ==============
validate:dev:
  stage: validate
  variables:
    TF_ROOT: ${CI_PROJECT_DIR}/environments/dev
  <<: *init-template
  script:
    - terraform validate
    - terraform fmt -check
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - environments/dev/**/*

plan:dev:
  stage: plan
  variables:
    TF_ROOT: ${CI_PROJECT_DIR}/environments/dev
  <<: *init-template
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 7 days
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

apply:dev:
  stage: apply
  variables:
    TF_ROOT: ${CI_PROJECT_DIR}/environments/dev
  <<: *init-template
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan:dev
  environment:
    name: development
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

# ============== PROD ENVIRONMENT ==============
validate:prod:
  stage: validate
  variables:
    TF_ROOT: ${CI_PROJECT_DIR}/environments/prod
  <<: *init-template
  script:
    - terraform validate
    - terraform fmt -check
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - environments/prod/**/*

plan:prod:
  stage: plan
  variables:
    TF_ROOT: ${CI_PROJECT_DIR}/environments/prod
  <<: *init-template
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 7 days
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

apply:prod:
  stage: apply
  variables:
    TF_ROOT: ${CI_PROJECT_DIR}/environments/prod
  <<: *init-template
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan:prod
  when: manual  # Requires approval
  allow_failure: false
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

This configuration automatically deploys to dev when you push to the `develop` branch, and requires manual approval for production deployments from `main`.

---

## Sample Terraform Code for Common Azure Resources

These examples demonstrate properly structured Terraform configurations for Azure resources you'll commonly provision.

### Virtual Network with Subnets

```hcl
# vnet.tf
resource "azurerm_virtual_network" "main" {
  name                = "${var.project_name}-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  tags = local.common_tags
}

resource "azurerm_subnet" "public" {
  name                 = "public-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "private" {
  name                 = "private-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
  
  service_endpoints = ["Microsoft.Storage", "Microsoft.Sql"]
}

resource "azurerm_network_security_group" "public" {
  name                = "${var.project_name}-public-nsg"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = local.common_tags
}

resource "azurerm_subnet_network_security_group_association" "public" {
  subnet_id                 = azurerm_subnet.public.id
  network_security_group_id = azurerm_network_security_group.public.id
}
```

### Storage Account with Security Settings

```hcl
# storage.tf
resource "azurerm_storage_account" "main" {
  name                     = "${lower(replace(var.project_name, "-", ""))}storage"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = var.environment == "prod" ? "GRS" : "LRS"
  
  # Security hardening
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
  shared_access_key_enabled       = true
  
  blob_properties {
    versioning_enabled = true
    
    delete_retention_policy {
      days = 7
    }
  }

  tags = local.common_tags
}

resource "azurerm_storage_container" "data" {
  name                  = "appdata"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}
```

### Using Local Values for Consistent Tagging

```hcl
# locals.tf
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    Pipeline    = "GitLab-CI"
  }
}
```

---

## Common Pitfalls and Troubleshooting

Even well-designed pipelines fail sometimes. This section covers the most common issues and their solutions.

### Authentication Failures

**Error:** `AADSTS7000215: Invalid client secret provided`

**Cause:** The `ARM_CLIENT_SECRET` value is expired or incorrect.

**Solution:** 
1. In Azure Portal, go to **App registrations** → your app → **Certificates & secrets**
2. Create a new client secret
3. Update the `ARM_CLIENT_SECRET` variable in GitLab

**Error:** `AuthorizationFailed: does not have authorization to perform action`

**Cause:** The service principal lacks permissions for the operation.

**Solution:** Verify the service principal has Contributor role:
```bash
az role assignment list --assignee <APP_ID> --output table
```

If missing, add it:
```bash
az role assignment create \
  --assignee <APP_ID> \
  --role Contributor \
  --scope /subscriptions/<SUBSCRIPTION_ID>
```

### State Locking Errors

**Error:** `Error acquiring the state lock`

**Cause:** A previous pipeline run crashed or was cancelled while holding the lock.

**Solution:** 
1. Check if another pipeline is running—if so, wait for it
2. If no pipeline is running, force-unlock the state:
```bash
terraform force-unlock <LOCK_ID>
```
The lock ID appears in the error message.

### GitLab Runner Issues

**Error:** `Terraform has no command named "sh"`

**Cause:** The HashiCorp Terraform Docker image's default entrypoint conflicts with GitLab CI.

**Solution:** Add the entrypoint override in your pipeline:
```yaml
image:
  name: hashicorp/terraform:1.10.5
  entrypoint: [""]
```

### Variables Not Available in Jobs

**Error:** Jobs can't access CI/CD variables you defined.

**Cause:** Variables are set as "Protected" but the branch isn't protected.

**Solution:**
1. **Option A:** Protect your branch: **Settings → Repository → Protected branches** → Add `main`
2. **Option B:** Uncheck "Protected" on the variable (less secure)

### Backend Initialization Failures

**Error:** `Error loading state: access denied`

**Cause:** The service principal lacks permissions on the state storage account.

**Solution:** Assign the Storage Blob Data Contributor role:
```bash
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee <APP_ID> \
  --scope /subscriptions/<SUB_ID>/resourceGroups/terraform-state-rg/providers/Microsoft.Storage/storageAccounts/<STORAGE_ACCOUNT>
```

### Debugging Failed Pipelines

When a pipeline fails:

1. **Check the job log:** CI/CD → Pipelines → Click the failed job
2. **Look for error patterns:** Authentication errors appear early; resource errors appear during plan/apply
3. **Test locally:** Run the same Terraform commands with the same credentials to reproduce the issue

**Enable Terraform debug logging** (temporarily, for troubleshooting only):
```yaml
variables:
  TF_LOG: DEBUG
```

Remove this after debugging—verbose logs can expose sensitive information.

### Quick Reference for Error Resolution

| Symptom | Likely Cause | First Step |
|---------|--------------|------------|
| "Invalid client secret" | Expired credentials | Regenerate secret in Azure Portal |
| "AuthorizationFailed" | Missing role assignment | Check service principal permissions |
| "Error acquiring lock" | Stale lock from failed run | Force-unlock or wait |
| Variables not found | Protected variable on unprotected branch | Protect the branch or unprotect the variable |
| "404" on state file | Incorrect backend configuration | Verify storage account name and container |

---

## Best Practices for Production Pipelines

Following these practices ensures your Terraform pipeline remains reliable, secure, and maintainable as your infrastructure grows.

**Version-pin everything.** Lock Terraform and provider versions to prevent unexpected changes:
```hcl
terraform {
  required_version = "~> 1.10.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}
```

**Commit the lock file.** Always commit `.terraform.lock.hcl` to ensure consistent provider versions across all environments and team members.

**Use separate state files per environment.** Never share state between dev and prod—isolation prevents accidental cross-environment changes.

**Require merge request reviews.** Protect your main branch and require at least one approval before merging infrastructure changes.

**Implement scheduled drift detection.** Add a scheduled pipeline that runs `terraform plan` nightly to detect manual changes made outside Terraform:
```yaml
drift-detection:
  stage: validate
  script:
    - terraform plan -detailed-exitcode
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  allow_failure: true
```

**Rotate credentials regularly.** Set calendar reminders to rotate service principal secrets before they expire. Azure defaults to 6-month expiration for client secrets.

---

## Conclusion

Building a Terraform pipeline with GitLab and Azure requires coordinating authentication, state management, runner infrastructure, and CI/CD configuration—but once established, it provides immense value through consistency, security, and collaboration.

The key components are:
- **GitLab Runners** (execute your pipeline jobs—choose shared runners for simplicity or self-hosted for enterprise control)
- **Service Principal or Managed Identity** (authenticates Terraform to Azure)
- **Remote State Storage** (enables team collaboration and state locking)
- **GitLab CI/CD Pipeline** (automates validation, planning, and controlled deployment)

For enterprise deployments, invest time in setting up self-hosted runners properly. The initial effort pays dividends in security, compliance, and access to private resources. Start with 2-3 VM-based runners using the Docker executor, then scale as your team grows.

Start with the basic single-environment pipeline, verify it works end-to-end, then expand to multi-environment configurations as your needs grow. The directory-per-environment approach provides the clearest separation for teams new to infrastructure as code.

Your next steps: 
1. Decide on your runner strategy (shared vs self-hosted)
2. If enterprise, deploy and register your first self-hosted runner
3. Push this configuration to GitLab
4. Run your first pipeline and create that resource group in Azure
5. Extend the Terraform configuration to provision the infrastructure your applications actually need

Once you see the flow working, you'll have a solid foundation for managing your entire Azure ecosystem as code.
