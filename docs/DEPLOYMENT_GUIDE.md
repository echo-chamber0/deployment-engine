# Data Commons Accelerator - GCP Marketplace Deployment Guide

A complete guide to deploying and managing Data Commons Accelerator through Google Cloud Marketplace.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Deployment via GCP Marketplace](#deployment-via-gcp-marketplace)
5. [Accessing Your Deployment](#accessing-your-deployment)
6. [Using Data Commons Accelerator](#using-data-commons-accelerator)
7. [Managing Your Deployment](#managing-your-deployment)
8. [Troubleshooting](#troubleshooting)
9. [Deleting Your Deployment](#deleting-your-deployment)

---

## Overview

### What is Data Commons Accelerator?

Data Commons Accelerator is a ready-to-deploy instance of the Data Commons platform on Google Kubernetes Engine (GKE). Data Commons is an open knowledge repository providing unified access to public datasets and statistics, enabling your organization to explore data without manually aggregating from multiple sources.

### What Problems Does It Solve?

Data Commons Accelerator addresses these common data exploration challenges:

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
- **Cloud Storage Bucket**: Scalable storage for custom data imports and exports
- **Kubernetes Workload**: Application deployed to your existing GKE cluster with Workload Identity authentication
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

**4. Workload Identity**

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

### Required Before Deployment

**1. Existing GKE Cluster**

You must have an existing GKE cluster in your GCP project where Data Commons will be deployed.

- **Minimum version**: Kubernetes 1.27 or higher
- **Cluster type**: Standard or Autopilot (both supported)
- **Workload Identity**: Must be enabled (enabled by default on GKE 1.27+)
- **Network**: VPC with Private Service Access configured

To verify your cluster exists and meets requirements:

1. Go to **Kubernetes Engine** in the GCP Console
2. Click **Clusters**
3. Find your cluster in the list
4. Click the cluster name to see details
5. Verify the **Cluster version** is 1.27 or higher
6. Check that **Workload Identity** is shown as "Enabled"

If you need to create a cluster first, see the [GCP Kubernetes Engine documentation](https://docs.cloud.google.com/kubernetes-engine/docs/resources/autopilot-standard-feature-comparison).

**Important**: The deployment form will ask for your cluster name and location.

**2. Private Service Access (PSA) Configuration**

Data Commons Accelerator uses CloudSQL with private IP connectivity, which requires Private Service Access (PSA) to be configured between your VPC network and Google's service producer network.

**Option A: Automatic PSA Creation (Recommended for New Users)**

**Default behavior** - The solution will automatically:
- Allocate a /20 IP range (4,096 addresses) in your VPC
- Create Private Service Access connection for CloudSQL
- Configure VPC peering with Google's service producer network

**Requirements:**
- Service Networking API must be enabled (solution enables automatically)
- User must have `compute.networkAdmin` role or equivalent
- Sufficient IP space in your VPC (solution uses /20 by default)

**Configuration:**
```yaml
create_psa_connection: true  # default - no action needed
psa_range_prefix_length: 20  # 4,096 IPs for production
```

**Option B: Use Existing PSA (Recommended for Enterprise)**

**When to use:**
- Your organization already has PSA configured for the VPC
- Multiple deployments in the same project/VPC
- Centralized network management by dedicated team
- Existing CloudSQL or other services using PSA

**Find your existing PSA range:**
```bash
gcloud compute addresses list --global \
  --filter="purpose=VPC_PEERING AND network~YOUR_VPC_NAME" \
  --format="table(name,address,prefixLength,network)"
```

**Verify PSA connection exists:**
```bash
gcloud services vpc-peerings list \
  --network=YOUR_VPC_NAME \
  --project=YOUR_PROJECT_ID
```

**Configuration:**
```yaml
create_psa_connection: false
existing_psa_range_name: "your-psa-range-name"  # from gcloud command above
```

**Important:** The PSA range must be in the same VPC network as your GKE cluster. The solution automatically derives the VPC from your GKE cluster configuration.

**Checking Your GKE Cluster's VPC**

To verify which VPC network your GKE cluster uses:
```bash
gcloud container clusters describe YOUR_CLUSTER_NAME \
  --location=YOUR_LOCATION \
  --format="value(network)"
```

The solution automatically uses this VPC for PSA configuration.

**3. Required IAM Permissions**

Your Google Cloud user account must have these roles on the GCP project:

| Role | Purpose |
|------|---------|
| Marketplace Admin | Deploy solutions from GCP Marketplace |
| Kubernetes Engine Admin | Create and manage GKE resources |
| Cloud SQL Admin | Create and manage CloudSQL instances |
| Storage Admin | Create and manage GCS buckets |
| Service Account Admin | Create and manage service accounts |
| Service Account User | Bind identities to workloads |

To verify your permissions:

1. Go to **IAM & Admin** in the GCP Console
2. Click **IAM**
3. Find your user email in the list
4. Click the role(s) next to your email
5. Verify the required roles are listed

If you're missing roles, contact your GCP administrator to grant them.

**4. Data Commons API Key**

Data Commons Accelerator requires an API key to access the Data Commons knowledge graph.

To get your API key:

1. Visit https://docs.datacommons.org/custom_dc/quickstart.html
2. Follow the "Get a Data Commons API Key" section
3. You'll receive your API key
4. Save it somewhere secure—you'll need it during deployment

**Security note**: This key authenticates your instance to Data Commons. Keep it confidential and never commit it to version control.

### Pre-Deployment Checklist

Before proceeding to the GCP Marketplace form, verify:

- [ ] GKE cluster exists and version is 1.27+
- [ ] Workload Identity is enabled on your cluster
- [ ] Private Service Access is configured
- [ ] You have required IAM roles
- [ ] You have your Data Commons API key saved
- [ ] You've estimated monthly costs and have budget approved
- [ ] You have 15-20 minutes available for deployment
- [ ] You know your GKE cluster name and location

---

## Deployment via GCP Marketplace

This section walks through the complete GCP Marketplace deployment process, field by field.

### Step 1: Navigate to GCP Marketplace

1. Open [Google Cloud Console](https://console.cloud.google.com)
2. In the top search bar, type "Marketplace"
3. Click **Marketplace** from the results
4. In the Marketplace search bar, search for "Data Commons Accelerator"
5. Click the **Data Commons Accelerator** solution
6. Review the solution overview, pricing information, and documentation
7. Click the **Deploy** button (or **Get Started**, depending on UI)

### Step 2: Deployment Configuration Form

The Marketplace will open a deployment configuration form. This form has multiple sections. We'll walk through each field.

#### Section 1: Basic Configuration

This section identifies your deployment and project.

**Deployment Name**

- **Field name**: `deployment_name`
- **What it means**: A friendly identifier for this deployment in GCP Marketplace and Infrastructure Manager
- **Format**: Lowercase letters, numbers, hyphens only (no spaces or special characters)
- **Examples**: `datacommons-prod`, `data-commons-team-analytics`, `dc-staging`
- **Why it matters**: Used for tracking multiple deployments and resource naming
- **Recommendation**: Use a descriptive name that includes environment and purpose (e.g., `datacommons-prod` not `deploy123`)

**Project**

- **Field name**: `project_id`
- **What it means**: Your GCP project where all resources will be created
- **Format**: Project ID (find in **Settings** > **Project** > **Project ID**)
- **Examples**: `my-project-123456`, `data-analytics-prod`
- **Why it matters**: All resources (database, storage, service accounts) are created in this project
- **Recommendation**: Must be the same project as your GKE cluster. If you don't see your project in the dropdown, you may lack Marketplace Admin permissions—contact your GCP administrator

**Region**

- **Field name**: `region`
- **What it means**: Default region for regional resources (CloudSQL, networking)
- **Format**: Region code (see [GCP Regions](https://cloud.google.com/compute/docs/regions-zones))
- **Examples**: `us-central1`, `us-east1`, `europe-west1`, `asia-southeast1`
- **Why it matters**: Affects latency (choose region near your users) and pricing (slight variations by region)
- **Recommendation**:
  - Choose region closest to your users
  - Must have same VPC connectivity as your GKE cluster
  - Can't be changed after deployment (don't change later)

#### Section 2: GKE Cluster Configuration

This section specifies which existing GKE cluster to deploy to.

**GKE Cluster Name**

- **Field name**: `gke_cluster_name`
- **What it means**: Name of your existing GKE cluster (not created by this solution)
- **Format**: Cluster name as shown in **Kubernetes Engine** > **Clusters**
- **Examples**: `prod-gke-cluster`, `analytics-cluster-1`, `us-central1-cluster`
- **Why it matters**: The application pods will run on this cluster
- **Requirement**: Cluster must exist, be running, and have Workload Identity enabled
- **Recommendation**: Use dropdown to select from existing clusters in your project. Don't type manually—use the selection dropdown to avoid typos

**GKE Cluster Location**

- **Field name**: `gke_cluster_location`
- **What it means**: Geographic location of your GKE cluster (region or zone)
- **Format**: Region (e.g., `us-central1`) or Zone (e.g., `us-central1-a`)
- **Examples**: `us-central1` (regional cluster), `us-central1-a` (zonal cluster)
- **Why it matters**: Must match actual cluster location for deployment to work
- **Recommendation**: If you selected a cluster name above, this may auto-populate. Verify it matches the cluster's actual location from the **Kubernetes Engine** console

**Kubernetes Namespace**

- **Field name**: `namespace`
- **What it means**: Kubernetes namespace where the application pods will run
- **What is a namespace?**: A logical partition of your Kubernetes cluster, like a folder. Namespaces allow multiple applications to run on the same cluster without interfering
- **Format**: Lowercase letters and hyphens only
- **Default**: `datacommons`
- **Examples**: `datacommons`, `datacommons-prod`, `analytics`
- **Why it matters**: Keeps this deployment isolated from other applications on your cluster
- **Recommendation**: Use the default `datacommons` unless your organization has a naming standard (e.g., `namespace-{environment}`). The namespace will be created automatically if it doesn't exist

#### Section 3: CloudSQL Database Configuration

This section configures the managed MySQL database that stores datasets.

**CloudSQL Machine Tier**

- **Field name**: `cloudsql_tier`
- **What it means**: The processing power and memory of your database server
- **Why it matters**: Directly impacts query performance and cost. Higher tiers = faster queries but higher monthly cost
- **Default**: `db-n1-standard-1`

**Recommendation:**
- **New deployments**: Choose `db-n1-standard-1` (good balance of cost and performance)
- **Proof-of-concept**: Use `db-f1-micro` to minimize costs
- **High-traffic production**: Use `db-n1-standard-2` or higher
- **You can upgrade later**: After deployment, you can change the tier via Infrastructure Manager update (5-10 minutes downtime)

**CloudSQL Disk Size**

- **Field name**: `cloudsql_disk_size`
- **What it means**: Storage space available for the database
- **Unit**: Gigabytes (GB)
- **Minimum**: 10 GB
- **Default**: 20 GB
- **Maximum**: 65,536 GB (65 TB)
- **Why it matters**: Stores datasets and metadata. Database automatically expands as you add data

**Recommendation:**
- **Default (20 GB)** is suitable for most deployments
- **Start small**: You don't need to predict exact size. Disk auto-grows
- **Monitor usage**: Check **Cloud SQL** > **Instances** > [your instance] > **Overview** tab for current size
- **Estimate**:
  - Empty installation: ~2 GB
  - With public Data Commons data: +5–10 GB
  - Custom datasets: Add estimated size
- **Example**: If you plan to import 50 GB of custom data, allocate 60 GB initially

**CloudSQL Region (Optional)**

- **Field name**: `cloudsql_region`
- **What it means**: Geographic region where the database is created (default: uses the `region` parameter from Section 1)
- **Format**: Region code, or leave empty to use default
- **Examples**: `us-central1`, `us-east1`, `europe-west1`
- **Why it matters**: Affects latency and must align with your VPC's Private Service Access configuration
- **Recommendation**: Leave empty unless you have a specific reason to override. The default region aligns with your application region

**High Availability**

- **Field name**: `cloudsql_ha_enabled`
- **What it means**: Enables automatic database replication to a different availability zone
- **How it works**:
  - **Disabled (default)**: Single database instance in one zone. If zone fails, data is lost
  - **Enabled**: Two instances (primary + replica) in different zones. If one zone fails, automatically switches to replica with zero downtime
- **Cost impact**: Approximately doubles the monthly database cost
- **Downtime**: 0 minutes (automatic failover)

**Recommendation:**
- **Production deployments**: Enable High Availability. The extra cost is worth the reliability
- **Non-production**: Disable to save costs
- **Can be changed later**: Update via Infrastructure Manager

#### Section 4: Cloud Storage Configuration

This section configures the GCS bucket for data storage.

**GCS Bucket Name**

- **Field name**: `gcs_bucket_name`
- **What it means**: Name of the Cloud Storage bucket (like a drive on the cloud)
- **Format**: Must be globally unique across all GCP projects. Only lowercase letters, numbers, hyphens, periods
- **Pattern**: `{project}-{purpose}-{random}` to ensure uniqueness
- **Examples**:
  - `datacommons-prod`
  - `mycompany-dc-analytics`
  - `datacommons-team-project1`
- **Why it matters**: Stores custom datasets, exports, and query results. Must be unique
- **Recommendation**: Include your project name and environment to make it identifiable. Avoid generic names like `data` or `bucket1`

**GCS Storage Class**

- **Field name**: `gcs_storage_class`
- **What it means**: How your data is stored (affects cost and access speed)
- **Default**: `STANDARD`
- **Monthly cost per GB**: Varies by class (see table below)

**Recommendation:**
- **New deployments**: Use `STANDARD` (best performance, reasonable cost)
- **Cost optimization**: Use `NEARLINE` if you access archived data less than monthly
- **Can't be changed later**: Choose carefully. To change, you'd need to migrate data to a new bucket
- **Best practice**: Use `STANDARD` initially; if you have archived data, create separate buckets with different storage classes

**GCS Location**

- **Field name**: `gcs_location`
- **What it means**: Geographic region or multi-region where the bucket is stored
- **Default**: `US`
- **Multi-region options**: `US`, `EU`, `ASIA` (replicates across multiple regions for redundancy)
- **Single-region options**: `us-central1`, `us-east1`, `europe-west1`, etc.

**Recommendation:**
- **Most deployments**: Use `US`, `EU`, or `ASIA` based on your users' location
- **Cost optimization**: Use single-region locations only if cost is critical and you accept single-point-of-failure risk
- **Data residency**: EU if you have European data requiring GDPR compliance

#### Section 5: API Configuration

This section provides the Data Commons API key that authenticates your instance.

**Data Commons API Key**

- **Field name**: `dc_api_key`
- **What it means**: Authentication token for accessing the Data Commons knowledge graph
- **Where to get it**: You obtained this in the Prerequisites section (https://docs.datacommons.org/custom_dc/quickstart.html)
- **Why it matters**: Authenticates your instance to Data Commons for accessing the knowledge graph
- **Security**: The key is stored as a Kubernetes secret. Never commit to version control

**Recommendation:**
- **Paste carefully**: Copy from your secure note
- **Verify**: Double-check there are no extra spaces or characters
- **Can't be changed via form**: If you enter the wrong key, you'll need to manually update the Kubernetes secret after deployment. Contact your GCP administrator if needed

#### Section 6: Application Configuration

This section controls how the application runs on your cluster.

**Application Replicas**

- **Field name**: `app_replicas`
- **What it means**: Number of application pods running simultaneously
- **Default**: `1`
- **Range**: 1–10 pods
- **How it affects cost**: Each replica uses cloud resources. 2 replicas ≈ 2x resource cost
- **How it affects reliability**: More replicas = better resilience to pod failures

**Recommendation:**
- **Development**: Use 1 replica
- **Production**: Use 2 replicas
- **High-traffic**: Use 3–4 replicas
- **Can be changed later**: Update via Infrastructure Manager

**Resource Tier**

- **Field name**: `resource_tier`
- **What it means**: CPU and memory allocated to each application pod
- **Default**: `medium`
- **Options**: `small`, `medium`, `large`
- **How it affects cost**: Each pod's resources are billed separately

**Resource Tier Specifications:**

| Tier | CPU | Memory | Best For |
|------|-----|--------|--------|
| `small` | 1.0 | 2 GB| Light workloads, <10 concurrent users |
| `medium` | 2.0 | 4 GB | Standard workloads, 10–100 concurrent users |
| `large` | 4.0 | 8 GB | Heavy workloads, >100 concurrent users, complex queries |


**Recommendation:**
- **Default (`medium`)** works for most deployments
- **Start with medium**: If you experience slow queries or high CPU, upgrade to `large`
- **Reduce to small**: If you're under-utilizing resources
- **Can be changed later**: Update via Infrastructure Manager (requires pod restart, 1–2 minutes downtime)

**Enable Natural Language Queries**

- **Field name**: `enable_natural_language`
- **What it means**: Allow users to ask questions in plain English (e.g., "Show me healthcare spending by state")
- **Default**: Enabled
- **How it works**: System translates natural language to statistical queries

**Recommendation:**
- **Most deployments**: Keep enabled. Users find it helpful
- **Specialized deployments**: Disable only if you have specific technical requirements
- **Can be changed later**: Update via Infrastructure Manager

**Enable Data Sync**

- **Field name**: `enable_data_sync`
- **What it means**: Automatically synchronize custom data from GCS bucket to CloudSQL database
- **Default**: Enabled
- **What it does**: Enables scheduled import of custom datasets you upload to GCS into CloudSQL for querying. Without it, you manage data imports manually
- **Performance impact**: Runs during off-peak hours; minimal performance impact

**Recommendation:**
- **Most deployments**: Keep enabled. You get fresh data automatically
- **Custom-only deployments**: Disable only if using only proprietary data and never want public data
- **Can be changed later**: Update via Infrastructure Manager (background process, no downtime)

### Step 3: Review and Deploy

**Review Deployment Configuration**

Before submitting:

1. Scroll to the top of the form
2. Review each section:
   - Deployment name is descriptive
   - Project ID is correct
   - GKE cluster name and location match your cluster
   - CloudSQL tier is appropriate for your use case
   - GCS bucket name is globally unique
   - Data Commons API key is correct
3. If any field is wrong, click on the section and correct it

**Accept Terms of Service**

1. Scroll to the bottom of the form
2. Read and check the checkbox: "I accept the Google Cloud Marketplace Terms"
3. Check any additional terms specific to Data Commons

**Click Deploy**

1. Click the **Deploy** button
2. A progress indicator appears
3. **Do not close the browser tab** during deployment

### Step 4: Monitor Deployment Progress

The deployment takes 10–15 minutes. Here's what's happening:

1. **Infrastructure Manager creates resources** (2–3 minutes):
   - CloudSQL instance being provisioned
   - GCS bucket being created
   - Service account being created
   - VPC network connections being established

2. **Terraform applies configuration** (3–5 minutes):
   - Resources are configured with your settings
   - Workload Identity bindings are created
   - Kubernetes secrets are created

3. **Helm deploys application** (3–5 minutes):
   - Container images are pulled from registry
   - Application pods are scheduled on your GKE cluster
   - Readiness probes verify the application is healthy

**To monitor progress:**

1. You should see a progress page in your browser showing deployment status
2. Alternatively, go to **Infrastructure Manager** > **Deployments**
3. Click on your deployment name
4. View the status indicator:
   - **Creating**: Deployment in progress
   - **Active**: Deployment succeeded (you can now access the application)
   - **Error**: Something failed (see Troubleshooting section)

**If deployment fails:**

1. Click on the deployment in Infrastructure Manager
2. Click **Details** or **Logs**
3. Review the error message
4. Common issues and solutions are in the [Troubleshooting](#troubleshooting) section

---

## Accessing Your Deployment

After successful deployment, you can access your Data Commons Accelerator application.

### View Deployment Outputs

Deployment outputs contain important information needed to access and manage your application.

**To view outputs:**

1. Go to **Infrastructure Manager** > **Deployments** in GCP Console
2. Click your deployment name
3. Click the **Outputs** tab
4. Review the outputs (see descriptions below)

**Key Outputs:**

| Output | Purpose | Example |
|--------|---------|---------|
| `namespace` | Kubernetes namespace where app is deployed | `datacommons` |
| `helm_release_name` | Helm release identifier | `datacommons-123abc` |
| `cloudsql_instance_name` | Database instance name | `datacommons-db-xyz123` |
| `cloudsql_connection_name` | Database connection string | `my-project:us-central1:datacommons-db-xyz123` |
| `gcs_bucket_name` | Cloud Storage bucket name | `data-datacommons-prod` |
| `workload_service_account` | Service account for the application | `datacommons-sa@my-project.iam.gserviceaccount.com` |
| `gke_cluster_name` | Your GKE cluster name | `prod-cluster` |
| `gke_cluster_location` | Cluster location | `us-central1` |
| `next_steps` | Post-deployment instructions | (Detailed instructions) |

### Verify Deployment

Before accessing the application, verify that all components are running correctly:

**Check Pods**

```bash
# Check that application pods are running
kubectl get pods -n datacommons

# Expected output: All pods should show status "Running"
```

**Check Services**

```bash
# Verify the service exists
kubectl get services -n datacommons

# Note: Service should be ClusterIP type
```

**View Application Logs**

```bash
# Check for any errors in the application logs
kubectl logs -n datacommons -l app=datacommons --tail=50
```

**Check Database**

1. Go to **Cloud SQL** > **Instances** in GCP Console
2. Click your instance
3. **Status** should show **Available**
4. Verify initialization is complete (takes 2–3 minutes after deployment)

### Access Application for Testing

**Important Security Note**

For security reasons, the application is deployed as a ClusterIP service without external access by default. This ensures your deployment is secure until you explicitly choose an exposure method.

**Quick Verification with Port-Forward**

For quick testing and verification of your deployment, use `kubectl port-forward` to create a temporary tunnel from your local machine to the application:

```bash
# Forward local port 8080 to the application service
kubectl port-forward -n datacommons svc/datacommons 8080:80

# Keep this terminal open; the port-forward will run in the foreground
# In another terminal, test the application:
curl http://localhost:8080

# Or open http://localhost:8080 in your browser
```

The application is NOT exposed to the internet with this method—you can only access it from your local machine.

**For Production Access**

To expose your application for production use, you must explicitly choose an exposure method and implement appropriate security controls. Follow the official GCP documentation:

**[GCP Guide: Exposing Applications in GKE](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps)**

---

## Using Data Commons Accelerator

### Key Features

**Statistical Data Explorer**

- Browse curated datasets from public sources (census data, economic indicators, health statistics)
- Filter by country, region, or time period
- Compare data across different dimensions

**Place Explorer**

- Geographic data visualization (maps, charts)
- Population and demographic analysis by location
- Economic indicators by region

**Timeline Tool**

- Time-series analysis for tracking trends
- Historical data visualization
- Forecasting and trend analysis (if enabled)

**Natural Language Queries** (if enabled)

- Ask questions in plain English
- Example queries:
  - "What is the population of California?"
  - "Show me healthcare spending by country"
  - "List the top 10 countries by GDP"

**Custom Datasets**

- Import your own data
- Combine with public Data Commons data
- Share datasets with team members

**Data Export**

- Download query results in CSV, JSON, or other formats
- Export to Cloud Storage for analysis in other tools
- Build API integrations with external systems

### Learning Resources

To get started using Data Commons:

- **Official Tutorials**: https://datacommons.org/tutorials
- **API Documentation**: https://docs.datacommons.org/api
- **Knowledge Graph Explorer**: https://datacommons.org/ (official site for learning about available data)
- **Custom Data Commons Guide**: https://docs.datacommons.org/custom_dc/

### Common Use Cases

**Government Agencies**
- Publish open data to citizens
- Combine multiple datasets for policy analysis
- Track public health and economic indicators

**Non-Profit Organizations**
- Research community statistics
- Track progress toward social impact goals
- Share data insights with stakeholders

**Researchers and Academics**
- Access curated datasets for studies
- Combine public data with primary research
- Teach data analysis and visualization

**Businesses**
- Market research using demographic data
- Competitive analysis by geography
- Economic trend analysis for strategy

---

## Troubleshooting

### Deployment Fails During Provisioning

**Symptoms:**
- Deployment shows "Error" status
- Infrastructure Manager shows failure message

**Common causes and solutions:**

**Issue: "GKE cluster not found"**
- Verify cluster name is spelled correctly
- Verify cluster is in the same project
- Go to **Kubernetes Engine** > **Clusters** to confirm cluster exists

**Issue: "Insufficient permissions"**
- You lack required IAM roles
- Ask your GCP administrator to grant:
  - Kubernetes Engine Admin
  - Cloud SQL Admin
  - Storage Admin
  - Service Account Admin

**Issue: "Private Service Access not configured"**
- VPC doesn't have Private Service Access enabled
- Ask your GCP administrator to configure it
- Or follow the [GCP Private Service Access guide](https://docs.cloud.google.com/sql/docs/mysql/configure-private-services-access)

**Issue: "CloudSQL quota exceeded"**
- Your project has created too many CloudSQL instances
- Delete unused instances, or request quota increase
- Go to **Quotas & System Limits** to view and request increases

### Application Pods Don't Start

**Symptoms:**
- Pods show **Pending** or **CrashLoopBackOff** status
- Application URL doesn't load

**To diagnose:**

1. Go to **Kubernetes Engine** > **Workloads**
2. Filter by namespace: `datacommons`
3. Click the deployment
4. Click a pod name
5. **Logs** tab shows error messages

**Common issues:**

**"ImagePullBackOff"** or **"ErrImagePull"**
- Container images can't be pulled from registry
- Likely: Wrong API key or image tag
- Solution: Verify the `dc_api_key` in the deployment form is correct
- If wrong, contact support for instructions to update the secret

**"CrashLoopBackOff"**
- Application is crashing immediately after starting
- Check logs (click pod > **Logs** tab)
- Common causes:
  - Database not reachable (wait 2–3 minutes for CloudSQL to initialize)
  - Invalid configuration in Kubernetes secrets
  - Insufficient memory (try upgrading `resource_tier`)

**"Pending" (pods won't start)**
- Cluster doesn't have capacity
- Check **Kubernetes Engine** > **Nodes** for available capacity
- If nodes are full, add more nodes to the cluster or delete other workloads

### Can't Access Application from Browser

**Symptoms:**
- Browser shows "Connection refused" or "ERR_CONNECTION_REFUSED"
- Or shows "Host unreachable"
- Port-forward fails to connect

**Troubleshooting Steps:**

**1. Verify Application Pods Are Running**

```bash
# Check pod status
kubectl get pods -n datacommons

# All pods should show "Running" status
# If any show "Pending" or "CrashLoopBackOff", see "Application Pods Don't Start" section
```

**2. Check the Service Exists**

```bash
# Verify service is created
kubectl get services -n datacommons

# Service should exist and be type "ClusterIP" by default
# By default, the application is NOT exposed externally for security
```

**3. Test Access with Port-Forward**

```bash
# Create a temporary tunnel to test the application
kubectl port-forward -n datacommons svc/datacommons 8080:80

# In another terminal, verify connectivity
curl http://localhost:8080/healthz

# If this works, the application is running correctly
# Issue is with your exposure method (see below)
```

**4. Review Application Logs**

```bash
# Check for errors in application logs
kubectl logs -n datacommons -l app=datacommons --tail=100

# Look for error messages that might indicate misconfiguration
# Check for database connection errors (usually appear in first log entries)
```

**Important Note**

By default, the application is deployed as a **ClusterIP service**. This is secure by default. You must explicitly expose it using:
- `kubectl port-forward` (for testing)
- Ingress (for production with TLS)
- LoadBalancer service (for direct external IP)
- Other methods described in the [GCP Exposing Applications guide](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps)

If port-forward works but browser access doesn't, the issue is with your exposure method, not the application itself.

### Private Service Access Issues

**Error: "Failed to create subnetwork. Couldn't find free blocks"**

**Cause:** The allocated IP range is exhausted or conflicts with existing ranges.

**Solution:**
1. Use a larger prefix length: Set `psa_range_prefix_length: 16` (65k IPs)
2. Or use existing PSA: Set `create_psa_connection: false` and provide existing range

**Error: "Cannot modify allocated ranges" or "Peering already exists"**

**Cause:** PSA connection already exists with different configuration.

**Solution:**
1. Set `create_psa_connection: false`
2. Find your existing range: `gcloud compute addresses list --global --filter="purpose=VPC_PEERING"`
3. Provide the range name in `existing_psa_range_name`

**Error: "Address 'xxx' is not in VPC 'yyy'"**

**Cause:** The provided PSA range is in a different VPC than your GKE cluster.

**Solution:**
1. Verify your GKE cluster's VPC: `gcloud container clusters describe CLUSTER --format="value(network)"`
2. Find PSA ranges in that VPC: `gcloud compute addresses list --global --filter="purpose=VPC_PEERING AND network~YOUR_VPC"`
3. Provide the correct range name

**Verifying PSA Configuration**

Check if PSA is properly configured:
```bash
# List PSA IP ranges
gcloud compute addresses list --global --filter="purpose=VPC_PEERING"

# List PSA connections
gcloud services vpc-peerings list --network=YOUR_VPC_NAME

# Verify CloudSQL can use the range
gcloud sql instances describe INSTANCE_NAME --format="value(ipConfiguration.privateNetwork)"
```

---

## Deleting Your Deployment

If you no longer need the Data Commons Accelerator, you can delete it to stop incurring costs.

### Delete via Infrastructure Manager

1. Go to **Infrastructure Manager** > **Deployments**
2. Click your deployment name
3. Click the **Delete** button
4. Confirm the deletion

**Duration:** 2–5 minutes

### What Gets Deleted

These resources are automatically deleted:

- Kubernetes namespace and all pods
- Helm release
- IAM service account bindings
- Kubernetes secrets

### What Persists (Manual Cleanup Required)

**Important:** These resources are NOT automatically deleted to prevent accidental data loss. You must delete them manually.

**CloudSQL Instance**

- Contains your database and datasets
- **To delete**:
  1. Go to **Cloud SQL** > **Instances**
  2. Click your instance
  3. Click **Delete** button
  4. Type the instance name to confirm
  5. Click **Delete**
- **Cost if left running**: ~$50–200/month depending on tier

**Cloud Storage Bucket**

- Contains your data exports and custom datasets
- **To delete**:
  1. Go to **Cloud Storage** > **Buckets**
  2. Click your bucket
  3. Click **Delete bucket** button
  4. Type the bucket name to confirm
  5. Click **Delete**
- **Warning**: Deletion is permanent if versioning not enabled
- **Cost if left running**: ~$0.015–0.020 per GB per month

**Service Account**

- Created for Workload Identity authentication
- **To delete** (optional):
  1. Go to **IAM & Admin** > **Service Accounts**
  2. Find the service account (name contains `datacommons`)
  3. Click the account
  4. Click **Delete** button
  5. Confirm deletion
- **Cost if left running**: Free (service accounts have no direct cost)

### Before Deleting

**Backup Important Data**

1. **Export from CloudSQL**:
   - Go to **Cloud SQL** > **Instances** > [your instance]
   - Click **Export** button
   - Choose export format (CSV, JSON, SQL dump)
   - Export to GCS bucket (if you're keeping the bucket)

2. **Download from GCS**:
   - Go to **Cloud Storage** > **Buckets** > [your bucket]
   - Select files you want to keep
   - Click **Download** to save locally

### After Deletion

**Cost Implications:**

- If you deleted the Infrastructure Manager deployment but kept CloudSQL and GCS:
  - You continue paying ~$50–200/month for CloudSQL
  - You continue paying for GCS storage
- **Delete CloudSQL and GCS** to stop all charges

**Redeployment:**

- CloudSQL and GCS data are deleted when you delete those resources
- To deploy again, you'll start from scratch
- The Infrastructure Manager deployment can be deleted and redeployed (you have the configuration saved)
