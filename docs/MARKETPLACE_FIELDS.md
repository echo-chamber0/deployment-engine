# GCP Marketplace Deployment Form Fields Reference

---

## Form Overview

The Marketplace deployment form is organized into five sections. Fields are listed below in the order they appear in the form.

---

## 1. Kubernetes Cluster

Configure your existing GKE cluster for Data Commons Accelerator deployment.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| **GKE Cluster Location** | Location selector | Yes | — | Region or zone where your GKE cluster runs |
| **GKE Cluster Name** | Cluster selector | Yes | — | Your existing GKE cluster |

### GKE Cluster Location

**Description:** Select the region or zone where your existing GKE cluster is located. The Marketplace UI defaults to showing the "Regional" tab.

**Details:**
- This must match the actual location of your GKE cluster
- CloudSQL will be automatically deployed in the same region
- Choose a region close to your users to minimize latency

**Validation:** Must be a valid GCP region or zone.

---

### GKE Cluster Name

**Description:** Select your existing GKE cluster from the dropdown. The list shows only clusters in the location you selected above.

**Details:**
- You must have an existing GKE cluster before deploying
- The cluster must have sufficient resources for Data Commons workloads
- Workload Identity must be enabled on the cluster for GCP service integration

**Validation:** Must be an existing GKE cluster in the selected location.

---

## 2. CloudSQL Database

MySQL database configuration for Data Commons Accelerator metadata and state. CloudSQL is automatically deployed in the same region as your GKE cluster with private IP connectivity.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| **CloudSQL Instance Tier** | Dropdown | Yes | `db-n1-standard-1` | Machine type for CloudSQL instance |
| **CloudSQL Disk Size (GB)** | Number | Yes | `20` | Initial disk size in GB |
| **Enable High Availability** | Boolean | Yes | `false` | Enable CloudSQL HA for production |
| **Private Service Access Configuration** | Dropdown | Yes | `create_20` | PSA range configuration |
| **Existing PSA Range Name** | String | Conditional | — | Name of existing PSA IP range |

### CloudSQL Instance Tier

**Description:** Machine tier for the CloudSQL MySQL instance with private IP connectivity. Choose based on workload requirements.

**Default:** `db-n1-standard-1` (3.75GB RAM, 1 vCPU)

**Valid Options:**

| Option | Value | RAM | vCPU | Use Case |
|--------|-------|-----|------|----------|
| Micro - 0.6GB RAM (dev/test only) | `db-f1-micro` | 0.6GB | Shared | Development/testing only |
| Small - 1.7GB RAM, 1 vCPU | `db-g1-small` | 1.7GB | 1 | Small dev/test environments |
| Standard-1 - 3.75GB RAM, 1 vCPU | `db-n1-standard-1` | 3.75GB | 1 | Small production workloads |
| Standard-2 - 7.5GB RAM, 2 vCPU (recommended) | `db-n1-standard-2` | 7.5GB | 2 | **Recommended for production** |
| Standard-4 - 15GB RAM, 4 vCPU | `db-n1-standard-4` | 15GB | 4 | High-traffic production |
| Standard-8 - 30GB RAM, 8 vCPU | `db-n1-standard-8` | 30GB | 8 | Very high-traffic (100+ users) |

**Recommendations:**
- **Dev/Test:** `db-f1-micro` or `db-g1-small`
- **Production:** `db-n1-standard-2` or higher
- **High-traffic:** `db-n1-standard-4` or `db-n1-standard-8` (100+ concurrent users)

**Note:** Micro and Small tiers are NOT recommended for production use.

---

### CloudSQL Disk Size (GB)

**Description:** Initial disk size in GB for the CloudSQL instance. CloudSQL will automatically grow the disk if needed.

**Type:** Number
**Default:** `20`
**Minimum:** `10` (MySQL requirement)

**Details:**
- Storage auto-grows when needed, so start conservatively
- You can adjust this later if needed
- Storage costs scale with size

**Validation:** Must be a positive integer >= 10.
**Regex Pattern:** `^[1-9][0-9]*$`

**Recommendations:**
- **Dev/Test:** 10-20 GB
- **Production:** 20-50 GB (depending on dataset size)
- **Large datasets:** 100+ GB

---

### Enable High Availability

**Description:** Enable CloudSQL high availability for production workloads. Creates a standby instance in a different zone for automatic failover.

**Type:** Boolean
**Default:** `false`
**Recommended for production:** `true`

**Details:**
- **Enabled:** Creates a standby replica in a different zone within the same region
- **Disabled:** Single instance (no automatic failover)
- HA provides automatic failover with minimal downtime
- HA increases cost (standby instance + synchronous replication)

**Use Cases:**
- **Enable (true):** Production workloads requiring high availability and automatic failover
- **Disable (false):** Development/testing environments, cost-sensitive non-critical workloads

---

### Private Service Access Configuration

**Description:** Choose how to configure Private Service Access (PSA) for CloudSQL private IP connectivity.

**Type:** Dropdown
**Default:** `create_20` (Create new /20 range)
**Required:** Yes

**Valid Options:**

| Option | Value | IP Count | Use Case |
|--------|-------|----------|----------|
| Create new /20 range (4,096 IPs - recommended) | `create_20` | 4,096 | **Recommended** - Standard production |
| Create new /24 range (256 IPs - dev/test) | `create_24` | 256 | Development/testing |
| Create new /16 range (65,536 IPs - large deployments) | `create_16` | 65,536 | Very large multi-deployment environments |
| Use my existing PSA range (enter name below) | `existing` | — | VPCs with existing PSA configuration |

**⚠️ CRITICAL WARNING:**

"Create new" options should **ONLY** be used on VPCs with **NO existing PSA configuration**.

Creating a new range on a VPC with existing PSA will **REPLACE all existing reserved peering ranges**, which may disrupt connectivity for other services using Private Service Access (CloudSQL, Cloud Composer, etc.).

**When to use each option:**

1. **`create_20` (default)** - Use for first PSA deployment on a VPC, standard production workloads
2. **`create_24`** - Use for dev/test on a dedicated VPC with no other PSA services
3. **`create_16`** - Use for large environments with many CloudSQL instances planned
4. **`existing`** - **Use this if your VPC already has PSA configured** (requires "Existing PSA Range Name" below)

**How to check if your VPC has existing PSA:**
```bash
gcloud compute addresses list --global --filter="purpose=VPC_PEERING" --format="value(name)"
```

If this returns results, select **"Use my existing PSA range"** and enter the name below.

---

### Existing PSA Range Name

**Description:** Name of your existing Private Service Access IP range. Required when you selected "Use my existing PSA range" above.

**Type:** String
**Default:** — (empty)
**Required:** Only if `psa_range_configuration = existing`
**Placeholder:** `google-managed-services-default`

**Details:**
- This field is ignored unless you selected "Use my existing PSA range" above
- Enter the exact name of your existing PSA IP allocation
- Common names: `google-managed-services-default`, `cloudsql-private-ip-range`

**How to find your PSA range name:**
```bash
gcloud compute addresses list --global \
  --filter="purpose=VPC_PEERING" \
  --format="value(name)"
```

**Example values:**
- `google-managed-services-default`
- `cloudsql-private-ip-range`
- `private-service-access`

---

## 3. Cloud Storage

GCS bucket configuration for Data Commons Accelerator data and custom datasets.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| **GCS Bucket Location** | Dropdown | Yes | `US` | Storage location for your bucket |
| **GCS Storage Class** | Dropdown | Yes | `STANDARD` | Storage class (cost vs access speed) |

### GCS Bucket Location

**Description:** Storage location for your Data Commons bucket. Choose between multi-region (higher availability) or single-region (lower latency and cost).

**Type:** Dropdown
**Default:** `US` (Multi-region)
**Required:** Yes

**Valid Options:**

**Multi-Region:**
| Option | Value | Description |
|--------|-------|-------------|
| US (Multi-region) | `US` | United States multi-region (highest availability) |
| EU (Multi-region) | `EU` | European Union multi-region |
| ASIA (Multi-region) | `ASIA` | Asia-Pacific multi-region |

**Single-Region (Americas):**
| Option | Value | Description |
|--------|-------|-------------|
| us-central1 | `us-central1` | Iowa, USA |
| us-east1 | `us-east1` | South Carolina, USA |
| us-west1 | `us-west1` | Oregon, USA |

**Single-Region (Europe):**
| Option | Value | Description |
|--------|-------|-------------|
| europe-west1 | `europe-west1` | Belgium |
| europe-west3 | `europe-west3` | Frankfurt, Germany |

**Single-Region (Asia):**
| Option | Value | Description |
|--------|-------|-------------|
| asia-southeast1 | `asia-southeast1` | Singapore |
| asia-northeast1 | `asia-northeast1` | Tokyo, Japan |

**💡 TIP:** Choose a location that matches your GKE cluster region to minimize latency and data transfer costs.

**Examples:**
- GKE cluster in `us-central1` → choose `US` or `us-central1`
- GKE cluster in `europe-west3` → choose `EU` or `europe-west1`
- GKE cluster in `asia-southeast1` → choose `ASIA` or `asia-southeast1`

**Cost & Performance Trade-offs:**
- **Multi-region:** Higher availability, higher cost, geo-redundant
- **Single-region:** Lower cost, lower latency within region, single-region redundancy

---

### GCS Storage Class

**Description:** Storage class determines cost and access speed. Choose based on how frequently you'll access your data.

**Type:** Dropdown
**Default:** `STANDARD`
**Required:** Yes

**Valid Options:**

| Option | Value | Use Case | Access Pattern |
|--------|-------|----------|----------------|
| Standard - Frequent access | `STANDARD` | Production workloads | Frequent/real-time access |
| Nearline - Monthly access | `NEARLINE` | Infrequent access | ~1x/month |
| Coldline - Quarterly access | `COLDLINE` | Archival data | ~1x/quarter |
| Archive - Long-term storage | `ARCHIVE` | Long-term archival | Rarely accessed |

**Recommendations:**
- **`STANDARD`** - Recommended for most Data Commons deployments (active datasets)
- **`NEARLINE`** - For backup datasets accessed occasionally
- **`COLDLINE`** - For compliance/archival datasets
- **`ARCHIVE`** - For long-term retention with rare access

**Note:** Lower-tier classes have retrieval costs and minimum storage durations (e.g., Nearline = 30 days minimum).

---

## 4. API Keys

API keys required for Data Commons integration.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| **Data Commons API Key** | String | Yes | — | API key for Data Commons API access |

### Data Commons API Key

**Description:** API key for accessing Data Commons APIs. Required for the application to function.

**Type:** String (sensitive)
**Required:** Yes
**Placeholder:** `...`

**Where to get it:**
Follow the instructions at: [https://docs.datacommons.org/custom_dc/quickstart.html#get-a-data-commons-api-key](https://docs.datacommons.org/custom_dc/quickstart.html#get-a-data-commons-api-key)

**Format:**
Alphanumeric string with underscores and hyphens.

**Security:**
- Stored as Kubernetes Secret in your cluster
- Rotatable after deployment by updating the secret

---

## 5. Application Settings

Data Commons Accelerator application configuration and resource allocation.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| **Resource Tier** | Dropdown | Yes | `medium` | Resource allocation for pods |
| **Domain Template** | Dropdown | Yes | — | Pre-built domain configuration |
| **Application Replicas** | Number | Yes | `1` | Number of application replicas |

### Resource Tier

**Description:** Resource allocation tier for Data Commons Accelerator pods. Determines CPU and memory limits.

**Type:** Dropdown
**Default:** `medium` (recommended)
**Required:** Yes

**Valid Options:**

| Option | Value | RAM | CPU | Use Case |
|--------|-------|-----|-----|----------|
| Small - 2Gi RAM, 1 CPU | `small` | 2Gi | 1 | Development, small datasets |
| Medium - 4Gi RAM, 2 CPU (recommended) | `medium` | 4Gi | 2 | **Recommended for production** |
| Large - 8Gi RAM, 4 CPU | `large` | 8Gi | 4 | Large datasets, high concurrency |

**Recommendations:**
- **`small`** - Dev/test only, small datasets
- **`medium`** - Standard production deployments (recommended starting point)
- **`large`** - High-traffic production, large datasets (>100GB), many concurrent users

**Note:** Ensure your GKE cluster has sufficient capacity for the selected tier × number of replicas.

---

### Domain Template

**Description:** Select a pre-built Data Commons configuration optimized for a specific domain. Each domain includes curated datasets, statistical variables, and visualizations tailored to that subject area. Choose the domain that best matches your use case.

**Type:** Dropdown
**Required:** Yes

**Valid Options:**

| Option | Value | Description |
|--------|-------|-------------|
| Education (education related data) | `education` | Pre-configured for education-related datasets (schools, enrollment, outcomes) |
| Health (health related data) | `health` | Pre-configured for health-related datasets (epidemiology, healthcare) |
| Energy (energy related data) | `energy` | Pre-configured for energy-related datasets (consumption, generation, emissions) |

**Note:** You can customize any template after deployment. The template just provides a starting point.

---

### Application Replicas (Advanced)

**Description:** Number of Data Commons Accelerator application replicas for high availability and load distribution.

**Type:** Number
**Default:** `1`
**Required:** Yes
**Level:** 1 (Advanced)

**Valid Range:** 1-10

**Details:**
- **1 replica** - Single instance (no HA, suitable for dev/test)
- **2-3 replicas** - High availability with load balancing (recommended for production)
- **4+ replicas** - High-traffic production with automatic scaling

**Recommendations:**
- **Dev/Test:** 1 replica
- **Production:** 2-3 replicas (for HA and rolling updates)
- **High-traffic:** 4+ replicas

**Capacity Planning:**
- Total resource usage = resource_tier × app_replicas
- Example: `medium` tier (4Gi RAM, 2 CPU) × 3 replicas = 12Gi RAM, 6 CPU total
- Ensure your GKE cluster has sufficient capacity

**Note:** Multiple replicas provide:
- High availability (if one pod fails, others continue serving)
- Load distribution across pods
- Zero-downtime rolling updates
