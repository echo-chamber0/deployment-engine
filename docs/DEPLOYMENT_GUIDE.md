# Data Commons Accelerator - GCP Marketplace Deployment Guide

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Deployment via GCP Marketplace](#deployment-via-gcp-marketplace)
5. [Accessing Your Deployment](#accessing-your-deployment)
6. [Using Data Commons Accelerator](#using-data-commons-accelerator)
7. [Troubleshooting](#troubleshooting)
8. [Deleting Your Deployment](#deleting-your-deployment)

---

## Overview

### What is Data Commons Accelerator?

Data Commons Accelerator is a ready-to-deploy instance of **Custom Data Commons** on Google Kubernetes Engine (GKE). [Data Commons](https://docs.datacommons.org/what_is.html) is an open knowledge repository providing unified access to public datasets and statistics. For more details on custom Data Commons and its benefits, see [Custom Data Commons documentation](https://docs.datacommons.org/custom_dc/).

### What Problems Does It Solve?

Data Commons Accelerator simplifies deploying a custom Data Commons instance—removing the complexity of infrastructure setup, Kubernetes configuration, and cloud resource provisioning. It empowers domain experts and data analysts to quickly create a custom Data Commons, integrate their datasets, and leverage the public Data Commons knowledge graph.

**Value Proposition:**

- **Simplicity**: Deploy a complex data solution with a few clicks, eliminating manual setup and configuration
- **Efficiency**: Streamline custom Data Commons adoption via Marketplace, bypassing traditional sales cycles
- **Accelerated Time-to-Insight**: Quickly join proprietary datasets with public data (census, economic, weather) to unlock new correlations
- **Empowerment**: Natural language interface allows non-technical users to query data without code

### Who Should Use It?

See [Custom Data Commons: When do I need a custom instance?](https://docs.datacommons.org/custom_dc/) for use cases and audience guidance.

### What Gets Deployed?

This solution provisions a complete data exploration platform:

- **Data Commons Accelerator Web Application**: Interactive interface for data exploration and visualization
- **CloudSQL MySQL Database**: Persistent storage for datasets and metadata (with optional high availability)
- **Cloud Storage Bucket**: Scalable storage for custom data imports
- **Kubernetes Workload**: Application deployed to your existing GKE cluster (not created by this solution) with Workload Identity authentication
- **Service Account**: Secure identity for accessing cloud resources

No additional infrastructure setup is required—everything integrates with your existing GCP project.

---

## Architecture

### Components

This Marketplace solution deploys the following GCP resources:

| Component | Description |
|-----------|-------------|
| **GKE Workload** | Data Commons application pods running in your existing cluster (namespace: `datacommons`) |
| **CloudSQL MySQL** | Managed database with private IP (via VPC Private Service Access) for dataset storage |
| **GCS Bucket** | Cloud Storage for custom data imports |
| **Service Account** | Workload Identity-enabled SA for secure access to CloudSQL, GCS, and Maps API |
| **db-init Job** | One-time Kubernetes Job that initializes the database schema |
| **db-sync CronJob** | Recurring job that syncs custom data from GCS to CloudSQL (every 3 hours) |

For details on Data Commons architecture and how the application works internally, see [Custom Data Commons documentation](https://docs.datacommons.org/custom_dc/).

### How Components Interact

```
User Browser                         GCS Bucket (Custom Data)
    │                                     │
    ├─> GKE Pod (Data Commons App)        ├─> db-sync CronJob (every 3 hours)
    │       │                             │
    │       ├─> CloudSQL Database <───────┘
    │       │   └─> Dataset storage, queries
    │       │
    │       ├─> GCS Bucket
    │       │   └─> Custom data, exports
    │       │
    │       └─> Google Maps API
    │           └─> Geospatial visualization
    │
    └─> Infrastructure Manager
        └─> Creates all related components/infrastructure of Data Commons (k8s resources, CloudSQL, IAM, GCS etc.)
```

**Deployment Workflow:**
1. You fill out the GCP Marketplace form with your preferences
2. Infrastructure Manager uses Terraform to provision CloudSQL, create GCS bucket, and bind service account
3. Helm deploys the Data Commons application to your GKE cluster
4. All resources are linked via Workload Identity and VPC Private Service Access

---

## Prerequisites

Before deploying Data Commons Accelerator, ensure you have the following:

### Infrastructure Requirements

| Requirement | Details |
|-------------|---------|
| **GKE Cluster** | Kubernetes 1.27+, Standard or Autopilot cluster |
| **Workload Identity** | Must be enabled (default on GKE 1.27+) |
| **VPC Network** | VPC with Private Service Access configured (PSA could be created via Marketplace form if it's not existing) |

**GKE Cluster:** If you need to create a cluster first, use the [GCP Kubernetes Engine creation form](https://console.cloud.google.com/kubernetes/add?).

### Required IAM Roles

Your Google Cloud user account must have these roles assigned on the GCP project:

| Role | Purpose |
|------|---------|
| Cloud Infrastructure Manager Admin | Deploy and manage Infrastructure Manager deployments |
| Infrastructure Administrator | Manage infrastructure resources provisioned by Terraform |
| Kubernetes Engine Developer | Deploy workloads to GKE clusters |
| Project IAM Admin | Assign IAM roles to service accounts |
| Service Account Admin | Create and manage GCP service accounts |
| Service Account User | Act as service accounts for Workload Identity |

Contact your GCP administrator if you are missing any roles.

### Data Commons API Key

Data Commons Accelerator requires an API key to access the Data Commons knowledge graph.

To obtain your API key, visit [Data Commons API documentation](https://docs.datacommons.org/custom_dc/quickstart.html/) and follow the "Get a Data Commons API Key" section. Save the key securely—you will need it during deployment.

**Security:** Keep your API key confidential and never commit it to version control.

---

## Deployment via GCP Marketplace

This section walks through deploying Data Commons Accelerator via GCP Marketplace.

### Step 1: Navigate to GCP Marketplace

1. Open [Google Cloud Console](https://console.cloud.google.com)
2. In the top search bar, type "Marketplace"
3. Click **Marketplace** from the results
4. In the Marketplace search bar, search for "Data Commons Accelerator"
5. Click the **Data Commons Accelerator** solution
6. Review the solution overview, pricing information, and documentation
7. Click the **Deploy** button (or **Get Started**, depending on UI)

### Step 2: Complete the Deployment Configuration Form

The Marketplace will open a deployment configuration form organized into several sections: **Basic** (deployment name and project), **GKE** (cluster details), **CloudSQL** (database settings), **Cloud Storage** (bucket configuration), **API** (Data Commons API key), and **Application** (pod replicas and resource sizing).

Each field has built-in tooltips with detailed guidance—hover over or click the help icon next to any field for clarification. The form validates your inputs and shows clear error messages if anything is incorrect.

**Before you start, gather these from Prerequisites:**
- Your **GKE cluster name and location**
- Your **Data Commons API key**

#### Private Service Access (PSA)

The CloudSQL section of the form asks how to configure Private Service Access for database connectivity:

| Option | When to Use | Configuration |
|--------|------------|----------------|
| **Create New PSA** | First deployment, no existing PSA | The form will create a /20 PSA range automatically |
| **Use Existing PSA** | PSA already configured, multiple deployments in same VPC | Provide your existing PSA range name |

**To find your existing PSA range:**

```bash
gcloud compute addresses list --global \
  --filter="purpose=VPC_PEERING AND network=YOUR_VPC_NAME" \
  --format="table(name,address,prefixLength,network)"
```

### Step 3: Review and Deploy

Once you've completed all sections:

1. **Review your selections** by scrolling through the form
2. **Accept the terms** by checking the Terms checkbox
3. **Click the Deploy button**

Deployment takes approximately **10–15 minutes**. A progress indicator will appear. **Do not close the browser tab** during deployment.

When the status shows **"Active"**, your deployment is complete. Proceed to the next section for accessing your application.

---

## Accessing Your Deployment

All deployment outputs—resource names, connection strings, and commands—are available in:
**Infrastructure Manager** > **Deployments** > your deployment > **Outputs** tab.

### Quick Access via Cloud Shell (Recommended)

The easiest way to access your deployment—no local tools needed:

1. Go to **GKE** > click your cluster > click **Connect**
2. Click **Run in Cloud Shell**
3. Run the port-forward command:
   ```bash
   until kubectl port-forward -n NAMESPACE svc/datacommons 8080:8080; do echo "Port-forward crashed. Respawning..." >&2; sleep 1; done
   ```
   (Replace `NAMESPACE` with your namespace from the Outputs tab; default: `datacommons`)
4. In the Cloud Shell toolbar, click **Web Preview** > **Preview on port 8080**

### Local Access via kubectl

If you have `gcloud` and `kubectl` installed locally:

1. Configure kubectl:
   ```bash
   gcloud container clusters get-credentials CLUSTER --location=LOCATION --project=PROJECT
   ```
2. Port-forward:
   ```bash
   until kubectl port-forward -n NAMESPACE svc/datacommons 8080:8080; do echo "Port-forward crashed. Respawning..." >&2; sleep 1; done
   ```
3. Open http://localhost:8080 in your browser

### Production Access

For external access, follow:
**[GCP Guide: Exposing Applications in GKE](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps)**

---

## Using Data Commons Accelerator

For comprehensive guides on using Data Commons, refer to the official documentation:

- [Custom Data Commons Documentation](https://docs.datacommons.org/custom_dc/)
- [Data Commons API Documentation](https://docs.datacommons.org/api)
- [Knowledge Graph Browser](https://datacommons.org/browser)

---

## Troubleshooting

### Deployment Logs

1. Go to **Solution deployments**
2. Click the three-dot menu (⋮) next to your deployment
3. Select **Cloud Build log**
4. Review the Terraform execution log for provisioning errors

**Common deployment errors:**
- **"GKE cluster not found"** — Verify cluster name and project match
- **"Insufficient permissions"** — Check [Required IAM Roles](#required-iam-roles)
- **"PSA not configured"** — See [PSA Issues](#private-service-access-issues) below

### Pod Status and Logs

**GKE Console:** Kubernetes Engine > Workloads > filter by namespace `datacommons`

**Quick diagnostics from Cloud Shell:**

```bash
kubectl get pods -n datacommons
kubectl describe pod POD_NAME -n datacommons
kubectl logs -n datacommons -l app.kubernetes.io/name=datacommons
```

**Common pod issues:**
- **Pending** — Cluster needs more capacity
- **CrashLoopBackOff** — Check logs; often CloudSQL still initializing (wait 2–3 min)
- **ImagePullBackOff** — Verify `dc_api_key` is correct

### Private Service Access Issues

```bash
# Check existing PSA ranges
gcloud compute addresses list --global --filter="purpose=VPC_PEERING"
```

- **"Couldn't find free blocks"** — Use `psa_range_configuration: "create_16"` for more IPs
- **"Peering already exists"** — Use `psa_range_configuration: "existing"` with your existing range name

---

## Deleting Your Deployment

If you no longer need the Data Commons Accelerator, delete the deployment to stop incurring costs.

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Search for "Solution deployments"
3. Find your deployment and click the **three-dot menu** (⋮)
4. Click **Delete**
5. Confirm the deletion
