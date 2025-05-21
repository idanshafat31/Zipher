# Zipher Databricks Integration & Permission Validation

## Project Overview

This repository contains the complete solution for Zipher's assignment, focusing on secure and efficient integration between Zipher's optimization services and the companies customers Databricks workspaces

### TOC + TL:DR -

1.  **Permissions Analysis (Task 1):** A detailed analysis of Databricks permissions, culminating in a recommendation for the principle of least privilege. This involves requesting `CAN MANAGE` permission on designated Databricks Job objects for Zipher's Service Principal.
2.  **Customer Integration Guide (Task 2):** A comprehensive, step-by-step guide for customers. This document details how to create a Service Principal, generate OAuth M2M credentials, and assign the necessary `CAN MANAGE` permissions on target jobs to Zipher's Service Principal.
3.  **Permissions Validation Script (Task 3):** A Python notebook script, (`zipher_validator.ipynb`), that uses Databricks Service Principal credentials (Client ID & Secret) to:
    *   Authenticate to the Databricks workspace via OAuth M2M.
    *   Optionally discover Job IDs accessible to the Service Principal.
    *   Validate whether the Service Principal possesses the required `CAN_MANAGE` permission on a specified list of target Job IDs, and report the validation outcome (For our example the answer is that the job is missing the required permissions).
---

## Task 1: Recommended Databricks Permissions for Zipher Integration

Zipher's product requires API access to specific job settings within the customer's Databricks workspace.
We want to identify the minimum necessary permissions Zipher requires to optimize Databricks job cluster configurations, adhering to the principle of least privilege.

**Analysis of Databricks Permission Model:**
Following a review I did of the Databricks access control model (in [Access control lists](https://docs.databricks.com/aws/en/security/auth/access-control/) and [Databricks Jobs API](https://docs.databricks.com/api/workspace/jobs)):

1.  **Job-Level Permissions:** Databricks provides granular permissions with individual job level.
2.  **Editing Job Settings:** The ability to "Edit job settings," which includes modifying the cluster configuration (for both new job clusters and existing all-purpose clusters), is granted by specific permission levels on the Job object.
3.  **API Endpoints:** Updating job settings is performed via API endpoints like `PATCH /api/2.1/jobs/{job_id}` (or the older `POST /api/2.0/jobs/update` and `POST /api/2.0/jobs/reset`). Access to these endpoints is managed by the permissions on the job itself.

**Actionable Recommendation:**

Zipher should request the following permission from its customers:

*   **Permission:** `CAN MANAGE`
*   **Scope:** To be granted to Zipher’s dedicated Service Principal **only on the specific Databricks Job objects** that Zipher is intended to optimize.
*   **Context:** This refers to the `CAN MANAGE` permission as defined within the **Job Access Control Lists (ACLs)** in Databricks, and not for all access settings. Could be that Zipher would also want `CAN MANAGE` on global Compute resources (clusters), in case the product also is intended to edit those.

**Why `CAN MANAGE`:**

1.  **Sufficiency for Core Task:**
    *   **Edit Job Settings:** The `CAN MANAGE` permission on a Job object explicitly allows modification of the job's settings. This is essential for Zipher to change cluster types, sizes, auto-scaling configurations, and Spark parameters defined within the job (Lower permission levels will not have these functionalities)
    *   **Manage Job Runs:** This permission level also includes the ability to `CAN MANAGE RUN`, which allows Zipher to start, or critically cancel job runs if needed. Cancelling inefficient or overrunning jobs is a key aspect of cost optimization.

2.  **Adherence to Least Privilege:**
    *   **More Restricted than `IS OWNER` (and for sure from Admin access):** While `IS OWNER` also allows editing job settings, it additionally grants the permission to delete the job, which Zipher should not be able to do in their customers environments. `CAN MANAGE` (on Jobs) provides the necessary editing and run management capabilities without the ability to delete the job, making it the less privileged of the two options that can modify job settings.
    *   **No Broader Permissions Needed:** This permission does **not** grant workspace administrator rights. It is strictly scoped to the designated jobs. For its primary function of editing job cluster configurations *within job definitions*, Zipher does **not** require `CAN MANAGE` permissions on other Databricks objects like Notebooks, Dashboards, or DLT Pipelines.
    *   *(Future Scope Consideration: If Zipher's services were to expand to include direct management of all-purpose clusters outside job definitions (e.g., live resizing), then `CAN MANAGE` on those specific Compute objects would be an additional, separate requirement.)*

**Conclusion:**
Requesting `CAN MANAGE` permission specifically for the target Databricks Job objects (and perhaps also for clusters) provides Zipher with the precise capabilities needed to deliver its optimization services while ensuring customers grant the most limited access necessary. The detailed steps for how customers can implement this are provided in the "Zipher Integration Guide" (Task 2).

*Visualizing Permission Scope*
The following diagram illustrates the scoped nature of the recommended permissions (not including higher roles of permissions):

![Databricks Permission Flow Chart for Zipher](assets/image.png "Zipher's Recommended Permission Scope")

---

## Task 2: Integrating Zipher with Your Databricks Workspace

A comprehensive, step-by-step guide for customers on setting up the Zipher integration is provided in the following document:

**[Zipher_Databricks_Integration_Guide.pdf](./Zipher_Databricks_Integration_Guide.pdf)**

This guide covers:
*   Prerequisites for integration.
*   A summary of the required `CAN MANAGE` permission on specific Databricks Jobs.
*   Detailed instructions for:
    *   Creating a Databricks Service Principal in the Account Console.
    *   Generating an OAuth M2M secret for the Service Principal.
    *   Granting `CAN MANAGE` permission on target jobs to the Service Principal (via UI and REST API).
    *   Configuring network IP Allow Lists.
    *   Securely sharing the necessary credentials with Zipher.
    *   Optionally verifying API connectivity.
*   Security best practices and emergency access revocation procedures.
*   A final checklist for customers.

---

## Task 3: Python Script - Databricks Permission Validator

A Python script to validate if a given set of Databricks Service Principal credentials has the required `CAN MANAGE` permissions on specified jobs.

**The Script:**
*   **Jupyter Notebook: [zipher_validator.ipynb](./zipher_validator.ipynb)**

### Features

*   Authenticates to Databricks using OAuth M2M with a Client ID and Client Secret.
*   Retrieves an access token using the workspace-specific token endpoint and the `all-apis` scope.
*   Optionally lists jobs accessible to the Service Principal (useful for identifying Job IDs, as we did not get a job ID in our instructions).
*   For each target Job ID, attempts to fetch its permissions.
*   Validates if the Service Principal's permissions include `CAN_MANAGE`.
*   Outputs a clear success/failure status for each job and an overall summary.

### Prerequisites

*   Python 3.7+
*   The `requests` Python library. If needed, install it using pip:
    ```bash
    pip install requests
    ```

### Configuration (Within the Notebook)

Before running the notebook, you need to configure the following variables in the initial configurations (Would be moved to a Constants notebook if the project grew. Same as our functions would be in a utils script and not with main):

*   `DATABRICKS_WORKSPACE_URL`: The full URL of the target Databricks workspace.
    *   Example recieved: `"https://dbc-28eb2f60-6b17.cloud.databricks.com/"`
*   `CLIENT_ID`: The Application (Client) ID of the Service Principal whose permissions are being validated.
    *   Example recieved: `"0294a4a7-054a-4210-8482-15c5d1aa1fe5"`
*   `CLIENT_SECRET`: The OAuth secret associated with the Service Principal.
    *   Example recieved: `"dosecbbb6cb467cdb795962e67738cf6a278"`
*   `TARGET_JOB_IDS`: A Python list of strings, where each string is a Job ID to be validated.
    *   Example we found while using the job discovered during the assignment: `["10813307686450"]`

### How to Run

1.  **Ensure Prerequisites:** Confirm Python and the `requests` library are installed in your Jupyter Notebook environment.
2.  **Open the Notebook:** Launch `zipher_validator.ipynb` in any IDE.
3.  **Configure:** Update the configuration variables as needed.
4.  **Execute Script** If needed, discover job IDs on the first run, and run a second time to get the permissions clarrified. 

The notebook will output the token acquisition status and the validation result for each target job.

### Discovering Job IDs

If you need to find Job IDs accessible to the Service Principal using the script:
1.  Look within the main at the end of the script, last 2 rows.
2.  Ensure the last 2 rows that call the `discover_workspace_jobs(...)` function is uncommented for this run (Would also place this in a different script for a full repo / workflow).
3.  Run the script - The output will list accessible Job IDs, which can then be used to update the `TARGET_JOB_IDS` configuration for the main validation.
4.  Comment (#) the last 2 rows of the script again for a run that just validates permissions and not discovering job IDs. 

### Validation Outcome for Provided Credentials (Task 3 Answer)

When the `zipher_validator.ipynb` notebook is run with the Databricks credentials provided in the assignment:

*   **Workspace URL:** `https://dbc-28eb2f60-6b17.cloud.databricks.com/`
*   **Client ID:** `0294a4a7-054a-4210-8482-15c5d1aa1fe5`
*   **Secret:** (as provided in the assignment)
*   **Target Job ID:** `["10813307686450"]` (Discovered by our function)

The script successfully authenticates and obtains an OAuth token. However, when attempting to retrieve permissions for Job ID `10813307686450`, it encounters a `403 Forbidden` error. The relevant output from the script, demonstrating this, is as follows:

```text
Validating permissions for Job ID: 10813307686450...
Http Error for Job ID 10813307686450: 403 Client Error: Forbidden for url: https://dbc-28eb2f60-6b17.cloud.databricks.com/api/2.0/permissions/jobs/10813307686450
  Details: Access to Job ID 10813307686450 permissions is Forbidden. The Service Principal may lack even CAN_VIEW permission on this job.
  Could not retrieve or parse permissions for Job ID 10813307686450. Validation failed for this job.
------------------------------

==================================================
OVERALL RESULT: FAILURE! Not all jobs meet the required permission level or errors occurred.
==================================================

Attempting to get details for job ID: 10813307686450
Successfully retrieved job details. Job name: candidate-assignment-job
This confirms you have at least READ access to the job.
You have access to view job task details.
```

This output confirms that, for the provided credentials and target job, the Service Principal **does not have the required `CAN_MANAGE` permission** (it lacks even the `CAN_VIEW` permission necessary to check its full permission set, though we do see we have at least READ permissions). This successfully demonstrates the validator script's ability to determine the permission status, thereby fulfilling the requirements of Task 3.
