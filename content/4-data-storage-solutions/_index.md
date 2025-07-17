---
title: "Data Analysis with Athena"
date: 2025-01-07T09:00:00+00:00
weight: 40
chapter: false
pre: "<b>4. </b>"
---

In this section, we will use Amazon Athena to run SQL queries directly on the processed weather data stored in S3. Athena is a serverless query service that makes it easy to analyze data using standard SQL without needing to set up a complex data warehouse infrastructure.

## Objectives:

- Set up Amazon Athena to query data from S3
- Create a database and an external table for the weather data
- Write SQL queries to analyze weather data
- Use enriched fields like `comfort_level`, `weather_severity`, and `wind_condition`
- Create reports and insights from weather data
- Optimize performance and cost for Athena queries

## Prerequisites

- **Completed Module 3**: Data Processing and Transformation
- **Have processed weather data** in S3 in the provided JSON format
- **AWS Console access** with Administrator or permissions for Athena, S3
- **Basic understanding of SQL** (SELECT, WHERE, GROUP BY, etc.)

## Step 1: Set up S3 Bucket for Athena Query Results

Athena needs an S3 bucket to store query results. We will create this bucket manually through the AWS Console.

### 1.1 Create S3 Bucket for Query Results

**Step 1: Access S3 Console**

1. Log in to the **AWS Console**
2. Search for and select the **S3** service
3. Click **Create bucket**

![S3 Console](/images/data-storage-solutions/4b1.png)

**Step 2: Configure Bucket**

1. **Bucket name**: Enter a unique name, e.g., `weather-athena-query-results-[your-account-id]`
2. **Region**: Select the same region as your weather data bucket
3. **Object Ownership**: Leave the default **ACLs disabled**
4. **Block Public Access**: Leave the default (block all public access)

![S3 Bucket Configuration](/images/data-storage-solutions/4b2.png)

**Step 3: Advanced Settings**

1. **Bucket Versioning**: Disable (not necessary for query results)
2. **Default encryption**: Enable with **Amazon S3 managed keys (SSE-S3)**
3. Click **Create bucket**

![S3 Bucket Advanced](/images/data-storage-solutions/4b3.png)

### 1.2 Set up Lifecycle Policy (Optional)

To automatically delete old query results and save costs:

**Step 1: Go to Bucket Management**

1. Click on the newly created bucket
2. Select the **Management** tab
3. Click **Create lifecycle rule**

**Step 2: Configure Lifecycle Rule**

1. **Rule name**: `delete-old-query-results`
2. **Status**: Enable
3. **Rule scope**: Apply to all objects in the bucket
4. **Lifecycle rule actions**: Expire current versions of objects
5. **Expire current versions of objects**: 30 days
6. Click **Create rule**

{{% notice info %}}
This lifecycle policy will automatically delete query results after 30 days to save on storage costs.
{{% /notice %}}

![S3 Lifecycle Policy](/images/data-storage-solutions/4b4.png)

### 1.3 Configure Athena Query Result Location

**Step 1: Access Athena Console**

1. Search for and select the **Amazon Athena** service

![Athena console](/images/data-storage-solutions/4b5.png)

**Step 2: Set up Query Result Location**

1. Select **Query editor** from the right-hand menu
2. Click **Settings** in the top right corner
3. Click **Manage**

![Athena Settings](/images/data-storage-solutions/4b6.png)

**Step 3: Configure Location**

1. **Query result location**: Select Browse S3 and choose the results bucket you just created.
   {{% notice warning %}}
   Remember to add a `/` at the end of the S3 path!
   {{% /notice %}}
2. **Expected bucket owner**: Leave blank
3. **Encrypt query results**: Check the box and select **SSE-S3**
4. Click **Save**

![Athena Settings](/images/data-storage-solutions/4b7.png)

## Step 2: Create Database and External Table

### 2.1 Create Weather Analytics Database

**Step 1: Open Query Editor**

1. In the Athena Console, select **Query editor** from the left menu
2. Ensure you are in **Data source**: AwsDataCatalog
3. **Database**: default

**Step 2: Create Database**

Copy and run the following query in the Athena Query Editor:

```sql
CREATE DATABASE IF NOT EXISTS weather_analytics
LOCATION 's3://weather-athena-query-results-[your-account-id]/databases/weather_analytics/';
```

![Athena Settings](/images/data-storage-solutions/4b8.png)

**Step 3: Select Database**

1. After the query runs successfully, refresh the page
2. In the **Database** dropdown, select `weather_analytics`

![Athena Settings](/images/data-storage-solutions/4b9.png)

### 2.2 Create External Table for Weather Data

**Step 1: Prepare Schema**

Based on the processed JSON structure from before, we will create an external table:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS weather_analytics.current_weather (
  timestamp STRING,
  city_name STRING,
  country STRING,
  latitude DOUBLE,
  longitude DOUBLE,
  data_collection_date STRING,
  temperature_celsius DOUBLE,
  temperature_fahrenheit DOUBLE,
  temperature_kelvin DOUBLE,
  feels_like_celsius DOUBLE,
  feels_like_fahrenheit DOUBLE,
  humidity_percent INT,
  pressure_hpa INT,
  visibility_meters BIGINT,
  uv_index DOUBLE,
  weather_id INT,
  weather_main STRING,
  weather_description STRING,
  weather_icon STRING,
  wind_speed_ms DOUBLE,
  wind_direction_deg INT,
  wind_gust_ms DOUBLE,
  wind_speed_kmh DOUBLE,
  wind_speed_mph DOUBLE,
  cloud_coverage_percent INT,
  heat_index_fahrenheit DOUBLE,
  heat_index_celsius DOUBLE,
  comfort_level STRING,
  wind_condition STRING,
  weather_severity STRING
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = '1'
)
LOCATION 's3://your-weather-processed-bucket/processed/'
TBLPROPERTIES (
  'has_encrypted_data'='false',
  'projection.enabled'='true',
  'projection.data_collection_date.type'='date',
  'projection.data_collection_date.range'='2025-01-01,NOW',
  'projection.data_collection_date.format'='yyyy-MM-dd',
  'storage.location.template'='s3://your-weather-processed-bucket/processed/year=${data_collection_date}'
);
```

![Athena Settings](/images/data-storage-solutions/4b10.png)

{{% notice info %}}
**Schema Explanation**:

- This schema exactly matches the JSON structure from module 3
- Includes new fields: `comfort_level`, `wind_condition`, `weather_severity`
- Uses JsonSerDe to parse JSON files
- Has partition projection for better performance
  {{% /notice %}}

### 2.3 Verify Table Creation

**Test Basic Query**

```sql
SELECT COUNT(*) as total_records
FROM weather_analytics.current_weather;
```

![Athena Query](/images/data-storage-solutions/4b11.png)

**Sample Data**

```sql
SELECT *
FROM weather_analytics.current_weather
ORDER BY timestamp DESC
LIMIT 5;
```

![Athena Query](/images/data-storage-solutions/4b12.png)

## Step 3: Basic Data Analysis

### 3.1 Data Quality Check

**Check data quality:**

```sql
SELECT
  COUNT(*) as total_records,
  COUNT(DISTINCT city_name) as unique_cities,
  COUNT(DISTINCT data_collection_date) as unique_dates,
  COUNT(temperature_celsius) as temp_records,
  COUNT(comfort_level) as comfort_records,
  COUNT(weather_severity) as severity_records,
  MIN(data_collection_date) as earliest_date,
  MAX(data_collection_date) as latest_date
FROM weather_analytics.current_weather;
```

![Athena Query](/images/data-storage-solutions/4b13.png)
