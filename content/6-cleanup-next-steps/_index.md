---
title: "Resource Cleanup"
date: 2025-01-07T09:00:00+00:00
weight: 60
chapter: false
pre: "<b>6. </b>"
---

After completing the hands-on workshop, it is crucial to clean up resources to prevent unexpected charges.

### 6.1 QuickSight Resources

1. **Delete Dashboard**

We will clean up in the reverse order of creation to avoid dependency errors (e.g., you can't delete an S3 bucket if it's still being used by another service).

### **Step 1: Unsubscribe from Amazon QuickSight**

QuickSight is a service with a monthly subscription fee, so prioritize unsubscribing from it first.

1.  **Access QuickSight**:
    - Open the AWS Console, search for and select **QuickSight**.
    - Click **Go to QuickSight** to enter the management interface.
2.  **Delete Assets (Analyses, Dashboards, Datasets)**:
    - In the left menu, go to each section: **Dashboards**, **Analyses**, and **Datasets**.
    - For each section, select all assets related to the workshop (e.g., "Weather Analysis Dashboard") and click **Delete**. Confirm the deletion.
3.  **Unsubscribe**:
    - In the top right corner, click your **profile icon** and select **Manage QuickSight**.
    - In the left menu, select **Your Account**.
    - Click **Manage account**. A confirmation dialog will appear. Confirm and proceed to delete the account.

### **Step 2: Delete Amazon Athena Resources**

Athena itself doesn't cost anything, but the query results stored in S3 do.

1.  **Access Athena**:
    - Open the AWS Console, search for and select **Athena**.
2.  **Drop Table**:
    - In the **Query editor**, make sure you have selected the `weather_analytics` database.
    - Run the following command to delete the table:
      ```sql
      DROP TABLE IF EXISTS weather_analytics.current_weather;
      ```
3.  **Drop Database**:
    - After deleting the table, run the following command:
      ```sql
      DROP DATABASE IF EXISTS weather_analytics;
      ```

### **Step 3: Delete Amazon S3 Buckets**

S3 charges for storage, so deleting buckets is very important.

{{% notice danger %}}
You must empty the bucket (delete all objects) before you can delete the bucket.
{{% /notice %}}

1.  **Access S3**:
    - Open the AWS Console, search for and select **S3**.
2.  **Empty and Delete Each Bucket**:
    - Perform the following steps for all 3 buckets:
      - `your-weather-raw-bucket-[ID]`
      - `your-weather-processed-bucket-[ID]`
      - `your-athena-query-results-bucket-[ID]`
    - **Steps for each bucket**:
      1.  Click the bucket name to open it.
      2.  Select all objects and folders inside.
      3.  Click the **Delete** button.
      4.  In the confirmation screen, type `permanently delete` and click **Delete objects**.
      5.  After the bucket is empty, go back to the list of buckets.
      6.  Select the (empty) bucket and click **Delete**.
      7.  In the confirmation screen, type the bucket name and click **Delete bucket**.

### **Step 4: Delete Scheduling and Lambda Resources**

1.  **Delete EventBridge (CloudWatch Events) Rule**:
    - Open the AWS Console, search for and select **Amazon EventBridge**.
    - In the left menu, select **Rules**.
    - Select the rule you created (e.g., `weather-collection-schedule`).
    - Click **Delete** and confirm.
2.  **Delete Lambda Functions**:
    - Open the AWS Console, search for and select **Lambda**.
    - Delete each function one by one:
      - Select the `weather-data-collector` function. Click **Actions** > **Delete**. Confirm deletion.
      - Select the `weather-data-processor` function. Click **Actions** > **Delete**. Confirm deletion.

### **Step 5: Delete CloudWatch Log Groups**

Lambda automatically creates log groups. They take up space and can incur small charges.

1.  **Access CloudWatch**:
    - Open the AWS Console, search for and select **CloudWatch**.
2.  **Delete Log Groups**:
    - In the left menu, select **Log groups**.
    - Find and select the following log groups:
      - `/aws/lambda/weather-data-collector`
      - `/aws/lambda/weather-data-processor`
    - Click **Actions** > **Delete log group(s)** and confirm.

### **Step 6: Delete IAM Roles and Policies**

This is an important step to clean up unnecessary permissions.

1.  **Access IAM**:
    - Open the AWS Console, search for and select **IAM**.
2.  **Delete IAM Role for Lambda**:
    - In the left menu, select **Roles**.
    - Find the role you created for Lambda (e.g., `weather-lambda-role`).
    - **Note**: If this role has attached policies, you need to **detach** them before you can delete the role. Usually, when you delete a role, AWS automatically detaches the policies you created.
    - Select the role, click **Delete** and confirm.
3.  **Delete IAM Role for QuickSight** (if you created one separately):
    - Repeat the process for the QuickSight role (e.g., `quicksight-athena-role`).

## Final Verification

After completing the above steps, double-check everything:

1.  **Check Billing Dashboard**: Go to **AWS Billing** and see if there are any unusual charges in the next few hours.
2.  **Check Services**: Glance through the S3, Lambda, and IAM consoles to ensure no resources related to the workshop remain.
