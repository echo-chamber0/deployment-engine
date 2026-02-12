# Data Commons Accelerator - User Guide

This guide explains how to access, configure, and use your Custom Data Commons instance deployed via the Google Cloud Marketplace.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Data Commons for Education](#data-commons-for-education)
3. [Data Commons for Health](#data-commons-for-health)
4. [Data Commons for Energy](#data-commons-for-energy)
5. [Known Limitations](#known-limitations)
6. [Request Support](#request-support)

---

## Getting Started

To configure the landing page, upload your company logo, and manage private data, you need to log in as the Data Commons Administrator. The steps below apply to all domain templates (Education, Health, Energy).

> [!TIP]
> For deployment and initial setup instructions, see the [Deployment Guide](DEPLOYMENT_GUIDE.md).

### Retrieve Administrator Credentials

The application administrator password is not provided in the deployment outputs for security reasons. To retrieve your initial credentials:

> [!TIP]
> These commands are available pre-populated with your deployment values in the [Infrastructure Manager Deployments](https://console.cloud.google.com/infra-manager/deployments) > your deployment > **Outputs** tab. You can copy and run them directly.

1. **Connect to your cluster** via Cloud Shell:

    ```bash
    gcloud container clusters get-credentials [CLUSTER_NAME] --region [REGION]
    ```

2. **Run the secret retrieval command**:

    ```bash
    echo 'Admin Username:' && kubectl get secret datacommons -n [NAMESPACE] -o jsonpath='{.data.ADMIN_PANEL_USERNAME}' | base64 -d && echo && echo 'Admin Password:' && kubectl get secret datacommons -n [NAMESPACE] -o jsonpath='{.data.ADMIN_PANEL_PASSWORD}' | base64 -d && echo
    ```

   Replace `[CLUSTER_NAME]`, `[REGION]`, and `[NAMESPACE]` with your deployment values. The namespace matches your deployment name.

### Administrator Log In

1. Navigate to your application URL (e.g., `https://education.example.com/`)
2. To access the **Admin Panel**, append `/admin` to the URL (e.g., `https://education.example.com/admin/`)
3. Enter the username and password retrieved in the previous step
4. You will be logged in as an administrator for the domain template selected during deployment (Education, Health, or Energy)

---

## Data Commons for Education

***Template: Student Recruitment Intelligence Center***

### Education Overview

The Education template combines your private applicant data with public demographic trends to help universities identify high-potential recruitment regions.

### Education: For Administrators

#### Prepare Custom Data

To populate the dashboard with your university's private data:

1. See [Prepare and load your own data](https://docs.datacommons.org/custom_dc/custom_data.html).
2. Ensure your data matches the required schema for Education template. You can download a sample CSV directly from the application **Data & Files** tab and fill in your data there.

#### Upload Custom Data

To populate the dashboard with your university's private data:

1. Log in and navigate to the **Admin Panel**.
2. Go to **Data & Files** tab.
3. Locate **Applicant Data Upload** section.
4. Click **Choose File**, select your CSV, and click **Upload**.
   - *Success*: You will see a "Rows successfully uploaded" message.
   - *Error*: The system will indicate specific line/column issues.

> [!NOTE]
> Large CSV files may take a few moments to process. The dashboard refreshes automatically after upload.

> [!TIP]
> **Trigger data sync immediately** — After uploading your CSV, the data is synced to the database by a CronJob (`datacommons-db-sync`) that runs every 3 hours. To avoid waiting, trigger it manually:
>
> **Via GKE Console (recommended):**
> 1. Go to **Kubernetes Engine** > **Workloads**
> 2. Filter by your namespace (matches your deployment name)
> 3. Click the **datacommons-db-sync** CronJob
> 4. Click **Run now** in the top toolbar
> 5. Monitor progress in the **Events** and **Logs** tabs
>
> **Via kubectl:**
> ```bash
> kubectl create job --from=cronjob/datacommons-db-sync manual-sync -n [NAMESPACE]
> kubectl logs -n [NAMESPACE] -l job-name=manual-sync -f
> ```

#### Customize User Interface

1. In the Admin Panel, navigate to **Theme Settings**.
2. **University Branding**:
   - **Name:** Update the university name displayed in the top bar.
   - **Logo:** Upload a PNG image.

3. **Dashboard Text**:
   - **Header Text:** Edit the main title (e.g., "Student Recruitment Intelligence Center").
   - **Hero Description:** Update the subtitle describing the purpose of your custom Data Commons.

4. Click **Save Changes**. Updates are applied immediately.

#### Data Security

Your data resides within a Google Cloud Storage bucket inside your dedicated GCP environment project. This setup allows you to control who can upload and manage the data that will be used for subsequent analytics.

> [!NOTE]
> Your data is kept private and is not shared with the public Data Commons. Data mixing and processing occur only on your deployed Custom Data Commons instance.

### Education: For Data Analysts & Researchers

#### Explore Recruitment Metrics

The dashboard provides a high-level view of your recruitment landscape:

- **Total Applicants:** Aggregated count for the target year.
- **Avg Opportunity Score:** A calculated metric indicating regional potential.
- **High Opportunity Markets:** Count of regions exceeding your target criteria.
- **Avg Household Income:** Public demographic data correlated with your target regions.

#### Interactive Maps

The **Recruitment Potential by State** map visualizes where your applicants are coming from versus high-opportunity areas.

- **Hover:** Hover over a state to see specific applicant counts and opportunity scores.

#### Filtering & Deep Dives

- **Filters:** Use the dropdowns at the top (e.g., Target Year) to filter all widgets on the page.
- **Standard Tools:** Click "Explore in Timeline Tool" on specific widgets to analyze the data using standard Data Commons graphing tools.

---

## Data Commons for Health

***Template: Population Health & City Comparison***

### Health Overview

The Health template allows organizations to compare specific health metrics (e.g., obesity, diabetes, smoking) across different cities, blending local private data with public CDC data.

### Health: For Administrators

#### Upload & Configuration

1. Log in and navigate to the **Admin Panel**
2. Go to the **Data & Files** tab
3. Upload your CSV containing city-level health metrics

See the [Getting Started](#getting-started) section for detailed upload steps and the sample CSV format.

- **Data Requirement:** Ensure your CSV contains city-level health metrics formatted according to the template schema.

#### Customize Branding

Follow the standard Theme Settings instructions to update the Organization Name and Logo.

### Health: For Data Analysts & Researchers

#### Compare Cities

The primary feature of this dashboard is the City Comparator.

1. Locate the "Compare Cities" section at the top.
2. The default city (e.g., Boston, MA) is selected.
3. Click **+ Add City** to select up to 4 additional cities.
4. The dashboard will update to show side-by-side metrics.

> [!TIP]
> The dashboard updates in real-time as you add or remove cities from the comparison.

#### Key Metrics Indicators

View cards displaying current percentages for:

- Obesity
- Smoking
- Physical Health
- Diabetes
- High Blood Pressure

#### Visual Comparison & Distribution

- **Bar Charts:** Compare the selected cities against each other across multiple health categories (e.g., "People Vaccinated," "People Who Are Sick").
- **Trend Lines:** View the "Health Issue Distribution" over time (2020–2024) to identify rising or falling trends.

---

## Data Commons for Energy

***Template: Methane Insights & Asset Risk***

### Energy Overview

The Energy template focuses on environmental monitoring, specifically correlating private asset locations with public methane plume data to identify high-risk leaks and community impact.

### Energy: For Administrators

#### Upload Asset Data

1. Log in and navigate to **Admin Panel** > **Data & Files**
2. Upload your **Asset Locations CSV** containing coordinates (latitude/longitude) of your infrastructure
3. See the sample CSV in the **Data & Files** tab for the required format

> [!IMPORTANT]
> Your CSV must contain latitude/longitude coordinates for each asset. See the sample file in the Admin Panel for the required format.

### Energy: For Data Analysts & Researchers

#### Risk Overview (KPI Cards)

- **Total Plume-Asset Intersections:** Percentage of assets currently intersecting with detected methane plumes.
- **Assets Near Communities:** Count of assets within a specific radius of populated areas.
- **High-Risk Issues:** Critical alerts detected in the last 30 days.

#### Methane Map Chart

This interactive map layers three datasets:

- **Methane Plumes (Public):** Satellite detection data.
- **Asset Density (Private):** Your uploaded infrastructure.
- **Community Risk Index (Public):** Census data indicating vulnerable populations.

#### Detailed Intersections Table

Review specific leak events in the table at the bottom of the dashboard:

- **Risk Score:** Low / Medium / High / Critical.
- **Leak Event ID:** Unique identifier for the detection.
- **Suspected Asset:** The specific asset ID linked to the leak.
- **Vulnerability Level:** Demographic risk score of the nearby community.
- **Action Status:** Current operational status (e.g., "Normal Operations").

---

## Known Limitations

- **Data Sync:** Dashboard data refreshes automatically after upload, but large CSVs may take a few moments to process.
- **Browser Support:** For best performance, use the latest version of Chrome.

> [!TIP]
> For troubleshooting deployment issues, port-forwarding errors, or pod status problems, see the [Deployment Guide — Troubleshooting](DEPLOYMENT_GUIDE.md#troubleshooting).

---

## Request Support

If you encounter issues not covered in this guide:

1. Check the deployment logs in your Google Cloud Console.
2. Contact your organization's system administrator.
3. To report bugs, request new features [Get Data Commons support](https://docs.datacommons.org/support.html).
