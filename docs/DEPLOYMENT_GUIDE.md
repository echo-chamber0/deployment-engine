# Data Commons Accelerator - GCP Marketplace User Guide

A complete guide to deploying and managing Data Commons Accelerator through Google Cloud Marketplace.

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

Data Commons Accelerator is a ready-to-deploy instance of [Custom Data Commons](https://docs.datacommons.org/custom_dc/) on Google Kubernetes Engine (GKE). [Data Commons](https://datacommons.org) is an open knowledge repository providing unified access to public datasets and statistics, enabling your organization to explore data without manually aggregating from multiple sources.

### What Problems Does It Solve?

Data Commons addresses these common data exploration challenges:

- **Data Fragmentation**: Public datasets scattered across multiple sources with different formats
- **Time-to-Insight**: Analysts spend weeks aggregating and standardizing data manually
- **Lack of Context**: Difficulty understanding relationships and connections between datasets
- **Scalability**: Traditional approaches break down with large datasets and many concurrent users
- **Customization**: Need to combine public data with proprietary internal datasets

### Who Should Use It?

Data Commons Accelerator is ideal for:

- **Data analysts** exploring public datasets and statistics
- **Researchers** studying demographic, economic, or environmental trends
- **Government agencies** publishing and analyzing public statistics
- **Non-profit organizations** working with community data
- **Academic institutions** teaching data analysis and visualization
- **Enterprise teams** integrating public data with business intelligence

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

The Data Commons Accelerator solution consists of four primary components:

**1. GKE Application Container**

Your Data Commons Accelerator application runs as Kubernetes pods in your existing GKE cluster. The application:

- Provides the web UI for data exploration
- Handles API requests from users and external clients
- Processes statistical queries and visualizations
- Integrates with the Google Maps API for geospatial features

The application runs in a dedicated Kubernetes namespace (default: `datacommons`) to keep it isolated from other workloads on your cluster.

**2. CloudSQL Database**

A managed MySQL database stores:

- Statistical datasets and curated public data
- Metadata describing available datasets
- User-created custom datasets
- Query history and saved visualizations

The database is deployed to a private IP address (via VPC Private Service Access) for security. It never exposes a public IP to the internet. Database replicas and backups are automatically managed by Google Cloud.

**3. Cloud Storage Bucket**

A GCS bucket stores:

- Custom datasets you import
- Exported data in various formats
- Query results and visualizations
- Temporary files during data processing

You control who can access the bucket via GCP IAM permissions.

**4. Data Ingestion**

The solution includes automated data processing:

- **Database Initialization (db-init)**: A one-time Kubernetes Job that initializes the CloudSQL database schema on first deployment
- **Data Sync (db-sync)**: A recurring CronJob that synchronizes custom data from your GCS bucket to CloudSQL every 3 hours

Both use the import service container image and run Python-based data processing. The sync schedule is configurable.

**5. Workload Identity**

A Google Cloud service account authenticates the application to cloud resources:

- CloudSQL access via Workload Identity binding
- GCS bucket access via IAM roles
- Google Maps API calls via API key
- All credentials managed securely (no keys stored in pods)

### How Components Interact

```
User Browser
    │
    ├─> GKE Pod (Data Commons Application)
    │       │
    │       ├─> CloudSQL Database (private IP)
    │       │   └─> Dataset storage, queries
    │       │
    │       ├─> GCS Bucket
    │       │   └─> Custom data, exports
    │       │
    │       └─> Google Maps API
    │           └─> Geospatial visualization
    │
    └─> Infrastructure Manager
        └─> Manages Kubernetes resources
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

1. Go to **Solution deployments**
2. Find your deployment and click the **three-dot menu** (⋮)
3. Click **Delete**
4. Confirm the deletion
