---
title: "2.3 Automated Scheduling with EventBridge"
date: 2025-01-03T08:45:00+07:00
weight: 3
---

In this section, we will set up automated scheduling for weather data collection using Amazon EventBridge (formerly CloudWatch Events). This allows Lambda functions to run on a regular schedule without manual intervention.

EventBridge is like AWS's "smart alarm clock":

- **Runs on time**: Triggers Lambda functions on a specific schedule
- **Precise**: Runs at the exact specified time (e.g., every hour, every day)
- **Automated**: No manual intervention needed
- **Cost-effective**: Only runs when needed, no resource cost when inactive

**Scheduling for:**

- Regular 24/7 data collection
- Data is always fresh and updated
- Fully automated
- Not dependent on an operator

**Schedule:**

- **Current Weather**: Every hour (24 times/day) - To monitor real-time weather

## Step 1: Set up EventBridge Rule for Current Weather

{{% notice tip %}}
**This step will create a schedule to run the Lambda every hour to collect current weather.**
{{% /notice %}}

### 1.1 Access EventBridge Console

1. **Go to AWS Console** → Search for **"EventBridge"** (or **"CloudWatch"** → **"Events"**)
2. **Click "EventBridge"** → **"Rules"** in the left menu
3. **Select the appropriate Region** (e.g., `us-east-1`)

![EventBridge Console](/images/data-collection/23b11.png)
![EventBridge Console](/images/data-collection/23b12.png)

### 1.2 Create Rule for Current Weather

1. **Click "Create rule"**

![Create Rule](/images/data-collection/23b13.png)

2. **Step 1 - Define rule detail:**
   - **Name**: `weather-current-hourly`
   - **Description**: `Collects current weather data every hour for 6 Vietnamese cities`
   - **Event bus**: `default`
   - **Rule type**: `Schedule`

![Rule Details](/images/data-collection/23b14.png)

3. **Click "Continue to create rule"**

### 1.3 Configure Schedule Pattern

1. **Schedule pattern**: Select **"A schedule that runs at a regular rate, such as every 10 minutes"**

2. **Rate expression**: Select **"rate"** and enter:
   ```
   1 hour
   ```

![Schedule Pattern](/images/data-collection/23b15.png)

**Schedule Pattern Options:**

- **Rate expression**:

  - `rate(1 hour)` = every hour
  - `rate(30 minutes)` = every 30 minutes
  - `rate(1 day)` = every day

- **Cron expression** (advanced):
  - `cron(0 * * * ? *)` = every hour at minute 0
  - `cron(0 8,12,16,20 * * ? *)` = 4 times/day (8h, 12h, 16h, 20h)
  - `cron(0 0 * * ? *)` = every day at 00:00
    ![Schedule Pattern](/images/data-collection/23b16.png)

3. **Click "Next"**

### 1.4 Select Target (Lambda Function)

1. **Target types**: Select **"AWS service"**

2. **Select a service**: Select **"Lambda function"**

3. **Function**: Select **`weather-current-collector`** (the function created in the previous step)

![Select Target](/images/data-collection/23b17.png)

4. **Additional settings**:
   - **Configure target input**: Select **"Constant (JSON text)"**
   - **JSON text**:
   ```json
   {
     "source": "eventbridge-schedule",
     "detail-type": "Scheduled Event",
     "detail": {
       "collection_type": "current_weather",
       "scheduled_time": "hourly",
       "trigger_source": "eventbridge"
     }
   }
   ```

![Target Input](/images/data-collection/23b18.png)

**This JSON input will:**

- Tell Lambda this is a scheduled event (not a manual test)
- Help differentiate current weather collection
- Provide metadata for logging and monitoring

5. **Click "Next"**

### 1.5 Configure tags and Review

1. **Tags (Optional)**:

   - **Key**: `Project` → **Value**: `WeatherETL`
   - **Key**: `Environment` → **Value**: `Production`

2. **Review all settings**:

   - Name: `weather-current-hourly`
   - Schedule: `rate(1 hour)`
   - Target: `weather-current-collector`
   - State: **Enabled**

3. **Click "Create rule"**

![Review Rule](/images/data-collection/23b19.png)

The rule for current weather has been created successfully.

The rule will trigger the `weather-current-collector` Lambda function every hour to collect current weather data.

![Review Rule](/images/data-collection/23b20.png)

{{% notice success %}}
**EventBridge Setup Complete!**

The EventBridge rule has been created and will automatically trigger the Lambda function every hour.

**Automated Workflow:**
`EventBridge` → `weather-current-collector` → `S3` → `CloudWatch Metrics`
{{% /notice %}}

## Step 3: Set up Monitoring with CloudWatch Alarms

**Why is Monitoring Necessary?**

When Lambda functions run automatically 24/7, you need to know immediately if there are problems:

- Lambda function fails
- Function runs too long
- Low success rate
- Unusual cost increase

### 3.1 Create SNS Topic for Email Alerts

**First, create an SNS Topic to receive email notifications for errors:**

1. **AWS Console** → **"SNS"** → **"Topics"** → **"Create topic"**

![SNS Topic](/images/data-collection/23b31.png)

2. **Topic configuration:**

   - **Type**: `Standard`
   - **Name**: `weather-etl-alerts`
   - **Display name**: `Weather ETL Alerts`

3. **Create topic**

4. **Create Subscription**:
   - **Protocol**: `Email`
   - **Endpoint**: `your-email@example.com`
   - **Confirm subscription** via email

![SNS Subscription](/images/data-collection/23b32.png)

### 3.2 Create CloudWatch Alarm for Lambda Errors

1. **AWS Console** → **"CloudWatch"** → **"Alarms"** → **"Create alarm"**

2. **Select metric**:

   - **Namespace**: `AWS/Lambda`
   - **Metric name**: `Errors`
   - **Dimensions**:
     - **FunctionName**: `weather-current-collector`

3. **Specify metric and conditions**:
   - **Statistic**: `Sum`
   - **Period**: `5 minutes`
   - **Threshold type**: `Static`
   - **Condition**: `Greater/Equal`
   - **Threshold value**: `1` (alert when there is ≥1 error in 5 minutes)

![Alarm Conditions](/images/data-collection/23b33.png)

4. **Configure actions**:

   - **Alarm state trigger**: `In alarm`
   - **SNS topic**: `weather-etl-alerts`

5. **Add name and description**:

   - **Alarm name**: `WeatherCurrentCollector-Errors`
   - **Description**: `Alert when Lambda weather-current-collector has errors`

6. **Create alarm**
   ![Alarm Conditions](/images/data-collection/23b34.png)

**Important**: Ensure everything is working correctly before letting it run automatically.

## Cost and Performance Optimization

### 1. Smart Scheduling Strategy

**Instead of running at the same frequency 24/7, you can optimize:**

- **Peak hours** (6:00-23:00): Every hour
- **Off-peak hours** (23:00-6:00): Every 2 hours
- **Weekends**: Every 2 hours (fewer people care about work-related weather)

**Create multiple rules with different schedules:**

**Rule 1 - Peak Hours:**

```
cron(0 6-23 * * ? *)
```

**Rule 2 - Off-peak Hours:**

```
cron(0 0,2,4 * * ? *)
```

### 2. Regional Optimization

**If you need to collect data for multiple regions:**

```json
{
```
