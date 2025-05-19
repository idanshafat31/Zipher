# Zipher Databricks Integration & Permission Validation

## Project Overview

This project outlines the solution for Zipher's home assignment, focusing on secure and efficient integration with customer Databricks workspaces. The primary objectives achieved are:

1.  **Permissions Analysis (Task 1):** A detailed analysis of Databricks permissions, culminating in a recommendation for the principle of least privilege – specifically, requesting `CAN MANAGE` permission on designated Databricks Job objects for Zipher's Service Principal.
2.  **Customer Integration Guide (Task 2):** A comprehensive, step-by-step guide for customers. This document details how to create a Service Principal, generate OAuth M2M credentials, and assign the necessary `CAN MANAGE` permissions on target jobs to Zipher's Service Principal.
3.  **Permissions Validation Script (Task 3):** A Python script (`databricks_permission_validator.py` or an equivalent Jupyter Notebook) that uses provided Databricks Service Principal credentials (Client ID & Secret) to:
    *   Authenticate to the Databricks workspace via OAuth M2M.
    *   Optionally discover Job IDs accessible to the Service Principal.
    *   Validate whether the Service Principal possesses the required `CAN_MANAGE` permission on a specified list of target Job IDs.
    *   Report the validation outcome.

This submission demonstrates an understanding of Databricks security, API interaction, and best practices for third-party integration.

---

## Task 1: Recommended Databricks Permissions for Zipher Integration

**Objective:** To identify the minimum necessary permissions Zipher requires to optimize Databricks job cluster configurations, adhering to the principle of least privilege.

**Background:**
Zipher's core functionality involves programmatically collecting data about Databricks jobs and applying optimization decisions, such as modifying job cluster configurations (e.g., cluster resizing, node type changes) to improve cost-efficiency and stability. This requires API access to specific job settings within the customer's Databricks workspace.

**Analysis of Databricks Permission Model:**
A review of the Databricks access control model, specifically focusing on [Access control lists](https://docs.databricks.com/aws/en/security/auth/access-control/) and the [Databricks Jobs API](https://docs.databricks.com/api/workspace/jobs), was conducted. The key findings are:

1.  **Job-Level Permissions:** Databricks provides granular permissions at the individual job level.
2.  **Editing Job Settings:** The ability to "Edit job settings," which includes modifying the cluster configuration (for both new job clusters and selecting existing all-purpose clusters), is granted by specific permission levels on the Job object.
3.  **API Endpoints:** Operations such as updating job settings are performed via API endpoints like `PATCH /api/2.1/jobs/{job_id}` (or the older `POST /api/2.0/jobs/update` and `POST /api/2.0/jobs/reset`). Access to these endpoints is governed by the permissions on the job itself.

**Actionable Recommendation:**

Zipher should request the following permission from its customers:

*   **Permission:** `CAN MANAGE`
*   **Scope:** To be granted to Zipher’s dedicated Service Principal **only on the specific Databricks Job objects** that Zipher is intended to optimize.
*   **Context:** This refers to the `CAN MANAGE` permission as defined within the **Job Access Control Lists (ACLs)** in Databricks.

**Justification for `CAN MANAGE` on Jobs:**

1.  **Sufficiency for Core Task:**
    *   **Edit Job Settings:** The `CAN MANAGE` permission on a Job object explicitly allows modification of the job's settings. This is essential for Zipher to change cluster types, sizes, auto-scaling configurations, and Spark parameters defined within the job.
    *   **Manage Job Runs:** This permission level also includes the ability to `CAN MANAGE RUN`, which allows Zipher to start and, critically, cancel job runs. Cancelling inefficient or overrunning jobs is a key aspect of cost optimization.

2.  **Adherence to Least Privilege:**
    *   **More Restricted than `IS OWNER`:** While `IS OWNER` also allows editing job settings, it additionally grants the permission to delete the job. `CAN MANAGE` (on Jobs) provides the necessary editing and run management capabilities without the ability to delete the job, making it the less privileged of the two options that can modify job settings.
    *   **No Broader Permissions Needed:** This permission does **not** grant workspace administrator rights. It is strictly scoped to the designated jobs. For its primary function of editing job cluster configurations *within job definitions*, Zipher does not require `CAN MANAGE` permissions on other Databricks objects like global Compute resources (clusters), Notebooks, Dashboards, or DLT Pipelines.
    *   *(Future Scope Consideration: If Zipher's services were to expand to include direct management of all-purpose clusters outside job definitions (e.g., live resizing), then `CAN MANAGE` on those specific Compute objects would be an additional, separate requirement.)*

**Conclusion:**
Requesting `CAN MANAGE` permission specifically for the target Databricks Job objects provides Zipher with the precise capabilities needed to deliver its optimization services while ensuring customers grant the most limited access necessary. The detailed steps for how customers can implement this are provided in the "Zipher Integration Guide" (Task 2).

---

## Task 2: Integrating Zipher with Your Databricks Workspace: A Step-by-Step Guide

*(Paste the full content of your Task 2 Markdown, including the updated Step 6, here)*

---

## Task 3: Python Script - Databricks Permission Validator

The Python script (`databricks_permission_validator.py` or the corresponding cells in the Jupyter Notebook `databricks_permission_validator.ipynb`) serves as a tool to verify that a given Databricks Service Principal has the necessary `CAN_MANAGE` permissions on specified jobs.

### Features

*   Authenticates using OAuth M2M with a Client ID and Client Secret.
*   Retrieves an access token using the workspace-specific token endpoint and the `all-apis` scope.
*   Optionally lists jobs accessible to the Service Principal (useful for identifying Job IDs).
*   For each target Job ID, attempts to fetch its permissions.
*   Validates if the Service Principal's permissions include `CAN_MANAGE`.
*   Outputs a clear success/failure status for each job and an overall summary.

### Prerequisites

*   Python 3.7+
*   The `requests` Python library. If not already installed, you can install it using pip:
    ```bash
    pip install requests
    ```

### Configuration (Within the Script/Notebook)

Before running, you need to configure the following variables at the top of the script or in the initial configuration cell of the Jupyter Notebook:

*   `DATABRICKS_WORKSPACE_URL`: The full URL of the target Databricks workspace.
    *   Example: `"https://dbc-28eb2f60-6b17.cloud.databricks.com/"`
*   `CLIENT_ID`: The Application (Client) ID of the Service Principal whose permissions are being validated.
    *   Example: `"0294a4a7-054a-4210-8482-15c5d1aa1fe5"`
*   `CLIENT_SECRET`: The OAuth secret associated with the Service Principal.
    *   Example: `"dosecbbb6cb467cdb795962e67738cf6a278"`
*   `TARGET_JOB_IDS`: A Python list of strings, where each string is a Job ID to be validated.
    *   Example using the job discovered during the assignment: `["10813307686450"]`

*(The script defaults to using the credentials and Job ID explored during the assignment, which demonstrate a `403 Forbidden` error when checking permissions, correctly indicating the SP lacks the necessary access to that specific job's permissions.)*

### How to Run

1.  **Ensure Prerequisites:** Confirm Python and the `requests` library are installed.
2.  **Configure Script:** Update the configuration variables (`DATABRICKS_WORKSPACE_URL`, `CLIENT_ID`, `CLIENT_SECRET`, `TARGET_JOB_IDS`) in the script/notebook as needed.
3.  **Execute:**
    *   If using the `.py` file:
        ```bash
        python databricks_permission_validator.py
        ```
    *   If using the Jupyter Notebook (`databricks_permission_validator.ipynb`): Run the cells in sequential order.

The script will print its progress, including token acquisition status and the validation result for each target job.

### Optional: Discovering Job IDs

If you need to find Job IDs accessible to the Service Principal:
1.  Locate the `if __name__ == "__main__":` block at the end of the script/notebook.
2.  Uncomment the line that calls `discover_workspace_jobs(...)`:
    ```python
    # Before:
    # discover_workspace_jobs(DATABRICKS_WORKSPACE_URL, CLIENT_ID, CLIENT_SECRET, TOKEN_ENDPOINT, OAUTH_SCOPE)
    # After:
    discover_workspace_jobs(DATABRICKS_WORKSPACE_URL, CLIENT_ID, CLIENT_SECRET, TOKEN_ENDPOINT, OAUTH_SCOPE)
    ```
3.  Run the script/notebook again. The job discovery function will execute after the main validation logic (or independently if `run_permission_validation()` is commented out). The output will list accessible Job IDs, which can then be used to update the `TARGET_JOB_IDS` configuration.

---
