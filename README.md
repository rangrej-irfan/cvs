# Privacera Manager Deployment on Google Cloud Platform (GCP)

This repository provides an automated deployment solution for Privacera Manager on Google Kubernetes Engine (GKE) using GitHub Actions. The deployment is fully automated through a GitHub Actions workflow that handles installation, configuration, and post-deployment setup.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Setup Guide](#setup-guide)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Support](#support)

## 🎯 Overview

This deployment solution automates the installation of Privacera Manager on GKE clusters using:

- **GitHub Actions Workflows**: Automated CI/CD pipeline
- **Self-Hosted Runners**: GitHub Actions runners deployed on your Kubernetes cluster
- **Ansible-based Installation**: Privacera's official installation method
- **Helm Charts**: Kubernetes-native deployment

### Key Features

- ✅ Automated installation and configuration
- ✅ GCP/GKE integration
- ✅ Secure credential management via Kubernetes Secrets
- ✅ Portal IP address discovery and reporting
- ✅ Post-deployment validation

## 🔧 Prerequisites

### 1. Infrastructure Requirements

#### **Google Kubernetes Engine (GKE) Cluster**
- ✅ **GKE Cluster**: Running GKE cluster in your GCP project
- ✅ **Node Capacity**: Minimum 3 nodes with sufficient CPU/memory resources
- ✅ **kubectl Access**: Local `kubectl` configured to access the cluster
- ✅ **Network Access**: Cluster nodes need internet access for:
  - `privacera-releases.s3.us-east-1.amazonaws.com` (Package downloads)
  - `github.com` (Repository access)
  - `hub2.privacera.com` (Privacera Hub)

#### **GitHub Actions Self-Hosted Runner**
- ✅ **Runner Controller**: `actions-runner-controller` installed on your Kubernetes cluster
- ✅ **Runner Deployment**: Runner deployed with labels `[self-hosted, k8s, linux]`
- ✅ **Resource Allocation**: 
  - CPU Request: 100m (minimum)
  - Memory Request: 256Mi (minimum)
  - CPU Limit: 500m 
  - Memory Limit: 1Gi

#### **GCP Service Account**
- ✅ **Service Account**: GCP service account with required IAM roles:
  - `roles/container.developer` (for `gcloud container clusters get-credentials`)
  - `roles/container.clusterAdmin` (for cluster management)
  - `roles/iam.serviceAccountUser` (if needed)
- ✅ **Service Account Key**: JSON key file downloaded and securely stored

### 2. Access Requirements

#### **GitHub Repository**
- ✅ **Repository Access**: Admin or write access to the repository
- ✅ **Actions Enabled**: GitHub Actions must be enabled
- ✅ **Secrets Management**: Ability to create repository secrets

#### **Privacera Hub Access**
- ✅ **Hub Account**: Valid Privacera Hub account
- ✅ **Credentials**: Username and password for hub access
- ✅ **Image Access**: Permission to pull Privacera Manager images

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Repository                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         GitHub Actions Workflow (install.yml)         │  │
│  └──────────────────────────────────────────────────────┘  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ Triggers
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              GKE Cluster (Self-Hosted Runner)                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  GitHub Actions Runner Pod                             │  │
│  │  - Executes workflow steps                            │  │
│  │  - Mounts GCP service account key (K8s Secret)        │  │
│  └──────────────────────────────────────────────────────┘  │
│                        │                                     │
│                        ▼                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Privacera Manager Namespace                         │  │
│  │  - Privacera Manager Pods                            │  │
│  │  - LoadBalancer Services                             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 📝 Setup Guide

### Step 1: Repository Setup

#### **1.1 Clone the Repository**
```bash
git clone <your-repository-url>
cd privacera-manager
```

#### **1.2 Verify Branch**
```bash
# Check current branch
git branch

# Switch to your working branch (e.g., 'main' or 'cvs')
git checkout <your-branch>
```

#### **1.3 Verify Configuration Files**
```bash
# Check variables.env exists
cat variables.env

# Verify workflow file exists
ls -la .github/workflows/install.yml
```

### Step 2: GCP Service Account Setup

#### **2.1 Create GCP Service Account**

1. Go to [GCP Console → IAM & Admin → Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts)
2. Click **CREATE SERVICE ACCOUNT**
3. Provide a name (e.g., `privacera-deployment-sa`)
4. Click **CREATE AND CONTINUE**
5. Grant the following IAM roles:
   - `Container Developer` (`roles/container.developer`)
   - `Kubernetes Engine Cluster Admin` (`roles/container.clusterAdmin`)
6. Click **DONE**

#### **2.2 Create and Download Service Account Key**

1. Click on the created service account
2. Go to the **KEYS** tab
3. Click **ADD KEY** → **Create new key**
4. Select **JSON** format
5. Click **CREATE** (JSON file will be downloaded)
6. **Save this file securely** - you'll need it for the next step

#### **2.3 Create Kubernetes Secret for GCP Service Account Key**

The service account key must be mounted as a Kubernetes secret in your runner pod.

```bash
# Set variables
SECRET_NAME="gcp-service-account-key"
NAMESPACE="actions-runner-system"
KEY_FILE="/path/to/your/downloaded-service-account-key.json"

# Create the Kubernetes secret
kubectl create secret generic $SECRET_NAME \
    --namespace=$NAMESPACE \
    --from-file=gcs.json=$KEY_FILE \
    --dry-run=client -o yaml | kubectl apply -f -
```

#### **2.4 Mount Secret in RunnerDeployment**

Update your `RunnerDeployment` to mount the secret:

```bash
kubectl patch runnerdeployment <your-runner-deployment-name> \
    -n actions-runner-system \
    --type='merge' -p='
{
  "spec": {
    "template": {
      "spec": {
        "volumes": [
          {
            "name": "gcp-secrets",
            "secret": {
              "secretName": "gcp-service-account-key"
            }
          }
        ],
        "volumeMounts": [
          {
            "name": "gcp-secrets",
            "mountPath": "/etc/gcp-secrets",
            "readOnly": true
          }
        ]
      }
    }
  }
}'
```

#### **2.5 Verify Secret Mount**

```bash
# Get runner pod name
RUNNER_POD=$(kubectl get pods -n actions-runner-system -o name | grep <your-runner-deployment-name> | head -1 | cut -d/ -f2)

# Verify secret is mounted
kubectl exec -n actions-runner-system $RUNNER_POD -- ls -la /etc/gcp-secrets/

# Verify key file content (first few lines)
kubectl exec -n actions-runner-system $RUNNER_POD -- cat /etc/gcp-secrets/gcs.json | head -5
```

### Step 3: Configure GitHub Secrets

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

#### **Required Secrets (2 secrets)**

| Secret Name | Purpose | Format/Example |
|-------------|---------|-----------------|
| `PRIVACERA_HUB_USER` | Privacera Hub username | Your Privacera Hub username |
| `PRIVACERA_HUB_PASSWORD` | Privacera Hub password | Your Privacera Hub password |

> **Note**: The GCP service account key is **NOT** stored in GitHub Secrets. It is mounted as a Kubernetes secret in the runner pod (see Step 2.3).

### Step 4: Configure Environment Variables

#### **4.1 Update `variables.env`**

Edit `variables.env` with your configuration:

```bash
# Privacera Manager version to deploy
PRIVACERA_VERSION=9.0.44.1

# Privacera Hub server URL
PRIVACERA_HUB_SERVER=hub2.privacera.com

# Deployment environment name (will be used as Kubernetes namespace)
DEPLOYMENT_ENV_NAME=your-environment-name
```

#### **4.2 Update `custom-vars/vars.gcp.yml`**

Edit `custom-vars/vars.gcp.yml` with your GCP project ID:

```yaml
# Provide GCP Project ID
PROJECT_ID: "your-gcp-project-id"
```

#### **4.3 Update `custom-vars/vars.kubernetes.yml` (if needed)**

The `K8S_CLUSTER_NAME` is typically auto-detected, but you can set it manually if needed.

## 🚀 Deployment

### Workflow Inputs

The workflow requires the following inputs when you trigger it manually:

#### **Required Inputs**

| Input Name | Description | Example | Type |
|------------|-------------|---------|------|
| `gcp_project_id` | Your GCP Project ID | `my-gcp-project-12345` | string |
| `gcp_region` | GCP region or zone for deployment | `us-central1` (regional) or `us-central1-c` (zonal) | string |
| `gke_cluster_name` | Your GKE cluster name | `my-cluster` (just the name, not the full context path) | string |

#### **Optional Inputs**

| Input Name | Description | Default | Type |
|------------|-------------|---------|------|
| `skip_helm_validation` | Skip Helm chart validation | `false` | boolean |
| `debug_mode` | Enable debug logging for verbose output | `false` | boolean |

**Important Notes:**
- **GCP Region**: 
  - For **regional clusters**: Use the region name (e.g., `us-central1`)
  - For **zonal clusters**: Use the zone name (e.g., `us-central1-c`)
- **GKE Cluster Name**: 
  - Use only the cluster name (e.g., `my-cluster`)
  - Do **NOT** use the full context path (e.g., `gke_project_region_my-cluster`)

### Step 1: Trigger the Workflow

1. Go to your GitHub repository
2. Click on the **Actions** tab
3. Select **Privacera Manager Install** workflow from the left sidebar
4. Click **Run workflow** button
5. Fill in the required inputs:
   - **GCP Project ID**: Enter your GCP project ID
   - **GCP Region**: Enter your GCP region or zone
   - **GKE Cluster Name**: Enter your GKE cluster name (just the name)
   - **Skip Helm Validation**: (Optional) Check to skip Helm validation
   - **Debug Mode**: (Optional) Check to enable verbose logging
6. Select the branch (e.g., `cvs` or `main`)
7. Click **Run workflow**

### Step 2: Monitor the Workflow

- The workflow will run on your self-hosted runner
- Monitor progress in the **Actions** tab
- Check logs for any errors or warnings

### Step 3: Review Deployment Output

After successful deployment, the workflow will:
- Display portal IP addresses from LoadBalancer services
- Show service URLs
- Display pod and service status

## ✅ Verification

### Check Deployment Status

```bash
# Set your deployment namespace
NAMESPACE="your-environment-name"  # From DEPLOYMENT_ENV_NAME in variables.env

# Check pod status
kubectl get pods -n $NAMESPACE

# Check service status
kubectl get svc -n $NAMESPACE

# Check LoadBalancer services for external IPs
kubectl get svc -n $NAMESPACE -o wide | grep LoadBalancer
```

### Access Portal

1. Get the external IP from LoadBalancer services:
```bash
   kubectl get svc -n $NAMESPACE -o jsonpath='{range .items[?(@.spec.type=="LoadBalancer")]}{.metadata.name}{"\t"}{.status.loadBalancer.ingress[0].ip}{"\n"}{end}'
   ```

2. Access the portal using the external IP address

### Verify Workflow Artifacts

After deployment, download artifacts from the workflow run:
- `portal-ips.txt`: Contains portal IP addresses
- `helm-charts-output/`: Helm chart outputs
- `privacera-manager/`: Installation artifacts

## 🔍 Troubleshooting

### Common Issues

#### **1. Workflow Not Starting**

**Problem**: Workflow doesn't appear in Actions tab or fails to start.

**Solutions**:
- Verify GitHub Actions is enabled in repository settings
- Ensure you're on the correct branch
- Check that the workflow file exists at `.github/workflows/install.yml`

#### **2. Runner Not Found**

**Problem**: `No runners found` or `Runner offline`.

**Solutions**:
- Verify runner pod is running: `kubectl get pods -n actions-runner-system`
- Check runner labels match: `[self-hosted, k8s, linux]`
- Verify runner is registered with GitHub

#### **3. GCP Authentication Failed**

**Problem**: `Failed to activate service account` or `Permission denied`.

**Solutions**:
- Verify GCP service account key is mounted: `kubectl exec -n actions-runner-system <pod> -- ls -la /etc/gcp-secrets/`
- Check service account has required IAM roles
- Verify key file is valid JSON: `kubectl exec -n actions-runner-system <pod> -- python3 -m json.tool /etc/gcp-secrets/gcs.json`

#### **4. Cluster Access Denied**

**Problem**: `403 Permission Denied` when accessing GKE cluster.

**Solutions**:
- Verify service account has `roles/container.developer` role
- Check cluster name is correct (just the name, not full context path)
- Verify region/zone is correct

#### **5. Privacera Hub Authentication Failed**

**Problem**: `Failed to authenticate to Privacera Hub`.

**Solutions**:
- Verify `PRIVACERA_HUB_USER` and `PRIVACERA_HUB_PASSWORD` secrets are set correctly
- Check credentials are valid
- Ensure network access to `hub2.privacera.com`

#### **6. Pods Not Starting**

**Problem**: Privacera Manager pods are in `Pending` or `CrashLoopBackOff` state.

**Solutions**:
- Check pod logs: `kubectl logs -n <namespace> <pod-name>`
- Verify node resources: `kubectl describe nodes`
- Check resource quotas: `kubectl describe quota -n <namespace>`

#### **7. LoadBalancer IP Not Assigned**

**Problem**: LoadBalancer services show `<pending>` external IP.

**Solutions**:
- GKE LoadBalancer provisioning can take 2-5 minutes
- Verify GKE cluster has LoadBalancer support enabled
- Check firewall rules allow traffic

### Getting Detailed Logs

Enable debug mode in the workflow:
1. When triggering the workflow, set **Debug Mode** to `true`
2. Review detailed logs in the workflow run

### Checking Workflow Logs

1. Go to **Actions** tab
2. Click on the workflow run
3. Expand each step to view logs
4. Look for error messages or warnings

## 📚 Additional Resources

### Official Documentation

- [Privacera Documentation](https://docs.privacera.com/)
- [Privacera Prerequisites](https://docs.privacera.com/get-started/base-installation/privaceracloud-data-plane/prerequisites.html)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)

### Repository Structure

```
privacera-manager/
├── .github/
│   └── workflows/
│       └── install.yml          # Main deployment workflow
├── custom-vars/                  # Configuration variables
│   ├── vars.gcp.yml             # GCP-specific configuration
│   ├── vars.kubernetes.yml       # Kubernetes configuration
│   └── ...
├── variables.env                 # Environment variables
└── README.md                     # This file
```

## 🎯 Success Criteria

Your deployment is successful when:

- ✅ All workflow steps complete without errors
- ✅ Privacera Manager pods are running (`kubectl get pods -n <namespace>`)
- ✅ LoadBalancer services have external IP addresses
- ✅ Portal is accessible via external IP
- ✅ All services are in `Running` state

## 🆘 Support

If you encounter issues:

1. **Review this README** for common solutions
2. **Check workflow logs** for specific error messages
3. **Verify prerequisites** are met
4. **Review GitHub secrets** configuration
5. **Contact Privacera Support** with:
   - Workflow run URL
   - Error messages
   - Relevant logs
   - Cluster and configuration details

---

**Last Updated**: Based on Privacera Manager version 9.0.44.1

**Note**: This deployment guide is specific to GCP/GKE. For other cloud providers or deployment methods, refer to the [official Privacera documentation](https://docs.privacera.com/).
