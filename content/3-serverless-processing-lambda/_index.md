---
title: "Serverless Data Processing and Transformation"
date: 2025-01-07T09:00:00+00:00
weight: 30
chapter: false
pre: "<b>3. </b>"
---

In this module, we will build Lambda functions to process and transform the raw weather data from module 2 into a format suitable for analysis. This is the "Transform" step in our ETL pipeline, which helps clean, standardize, and create meaningful metrics from raw data.

**Raw data from the OpenWeatherMap API has several issues**:

- **Complex structure**: JSON is hard to query
- **Inconsistent units**: Kelvin, m/s, Pascal...
- **Redundant data**: Many unnecessary fields
- **Lacks insights**: No derived metrics

**Therefore, processing is needed to get:**

- **Flat structure**: Easy to query with SQL
- **Consistent units**: Celsius, km/h, %...

**Processing Flow:**

1. **Raw data** from Module 2 is saved to S3
2. **S3 Event** triggers the Lambda processor
3. **Lambda** transforms the data and saves it to the processed bucket
4. **Processed data** is ready for analytics (Module 4)

## Step 1: Create S3 Bucket for Processed Data

**Why separate buckets?**

- **Separation of concerns**: Raw vs. Processed data
- **Security**: Different access permissions
- **Cost optimization**: Different storage classes
- **Analytics**: Processed data is optimized for queries

### 1.1 Create Processed Data Bucket

1. **AWS Console** → **S3** → **Create bucket**

2. **Bucket configuration**:
   - **Bucket name**: `weather-processed-{your-account-id}`
   - **Region**: `ap-southeast-1` (same as raw bucket)
   - **Block all public access**: Enabled
   - **Bucket versioning**: Disable
   - **Default encryption**: SSE-S3

![Create Processed Bucket](/images/data-processing/3b1.png)

3. **Create bucket**

### 1.2 Create Folder Structure

**Create a folder structure optimized for analytics:**

```
weather-processed-{account-id}/
├── current-weather/
    ├── year=2025/
    │   ├── month=01/
    │   │   ├── day=03/
    │   │   │   ├── hour=00/
    │   │   │   │   ├── hcm_20250103_000000.json
    │   │   │   │   ├── hanoi_20250103_000000.json
    │   │   │   │   └── ...
    │   │   │   └── hour=01/
    │   │   └── day=04/
    │   └── month=02/
    └── year=2026/

```

**We will use Hive-style partitioning because:**

- **Athena optimization**: Better query performance
- **Cost savings**: Only scans necessary data
- **Easy filtering**: Filter by year/month/day/hour
- **Scalability**: Efficiently handles large datasets

## Step 2: Create Lambda Function for Data Processing

### 2.1 Create Lambda Function

1. **AWS Console** → **Lambda** → **Create function**

2. **Function configuration**:
   - **Function name**: `weather-data-processor`
   - **Runtime**: `Python 3.11`
   - **Architecture**: `x86_64`
   - **Execution role**: `WeatherCollectorLambdaRole`

![Create Lambda](/images/data-processing/3b2.png)

3. **Advanced settings**:
   - **Memory**: `512 MB`
   - **Timeout**: `5 minutes`
   - **Environment variables**:
     - `PROCESSED_BUCKET_NAME`: `weather-processed-{your-account-id}`
     - `LOG_LEVEL`: `INFO`

![Lambda Configuration](/images/data-processing/3b3.png)

### 2.2 Lambda Function Code

**Replace the default code with the following:**

```python
import json
import boto3
import datetime
import logging
from decimal import Decimal
import os
from urllib.parse import unquote_plus

# Logging configuration
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# AWS clients with retry configuration
s3_client = boto3.client('s3', config=boto3.session.Config(
    retries={'max_attempts': 3, 'mode': 'adaptive'}
))

def lambda_handler(event, context):
    """
    Main Lambda handler for weather data processing
    """
    try:
        # Environment variable validation
        processed_bucket = os.environ.get('PROCESSED_BUCKET_NAME')
        if not processed_bucket:
            logger.error("PROCESSED_BUCKET_NAME environment variable is required")
            return {
                'statusCode': 500,
                'body': json.dumps({
                    'error': 'Configuration error',
                    'message': 'PROCESSED_BUCKET_NAME environment variable not set'
                })
            }

        processed_count = 0

        # Process each S3 event record
        for record in event['Records']:
            try:
                # Extract bucket and key from event
                source_bucket = record['s3']['bucket']['name']
                source_key_encoded = record['s3']['object']['key']
                source_key = unquote_plus(source_key_encoded)

                logger.info(f"Processing file: {source_key} from bucket: {source_bucket}")
                if source_key_encoded != source_key:
                    logger.debug(f"URL decoded key from {source_key_encoded} to {source_key}")

                # Get raw weather data from S3
                response = s3_client.get_object(Bucket=source_bucket, Key=source_key)
                raw_data = json.loads(response['Body'].read().decode('utf-8'))

                # Transform weather data
                processed_data = transform_weather_data(raw_data)

                # Create key for processed file
                processed_key = source_key.replace('raw/', 'processed/').replace('.json', '.jsonl')

                # Save processed data to S3 as NDJSON (jsonl)
                if isinstance(processed_data, list):
                    jsonl_content = '\n'.join(json.dumps(obj, default=decimal_default) for obj in processed_data)
                else:
                    jsonl_content = json.dumps(processed_data, default=decimal_default)

                s3_client.put_object(
                    Bucket=processed_bucket,
                    Key=processed_key,
                    Body=jsonl_content,
                    ContentType='application/jsonl; charset=utf-8',
                    ServerSideEncryption='AES256'
                )

                processed_count += 1
                logger.info(f"Successfully processed and saved: {processed_key}")

            except Exception as e:
                logger.error(f"Error processing record {source_key}: {str(e)}")
                continue

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': f'Successfully processed {processed_count} files',
                'processedCount': processed_count
            })
        }

    except Exception as e:
        logger.error(f"Lambda execution error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': str(e)
            })
        }

def transform_weather_data(raw_data):
    """
    Transform raw OpenWeatherMap data into analytics-friendly format
    """
    try:
        # Extract timestamp (use collection_timestamp if available, otherwise dt)
        if 'collection_timestamp' in raw_data:
            timestamp = raw_data['collection_timestamp']
            collection_date = datetime.datetime.fromisoformat(timestamp.replace('Z', '+00:00')).strftime('%Y-%m-%d')
        else:
            timestamp = datetime.datetime.fromtimestamp(raw_data['dt']).isoformat() + 'Z'
            collection_date = datetime.datetime.fromtimestamp(raw_data['dt']).strftime('%Y-%m-%d')

        # Extract basic location and weather information
        processed_data = {
            'timestamp': timestamp,
            'city_name': raw_data.get('name', 'Unknown'),
            'country': raw_data.get('sys', {}).get('country', 'Unknown'),
            'latitude': raw_data.get('coord', {}).get('lat'),
            'longitude': raw_data.get('coord', {}).get('lon'),
            'data_collection_date': collection_date
        }

        # Add custom city metadata if available
        if 'city_metadata' in raw_data:
            city_meta = raw_data['city_metadata']
            processed_data.update({
                'city_name': city_meta.get('name', processed_data['city_name']),
                'latitude': city_meta.get('lat', processed_data['latitude']),
                'longitude': city_meta.get('lon', processed_data['longitude'])
            })

        # Temperature conversion (OpenWeatherMap returns Celsius when units=metric)
        temp_celsius = raw_data.get('main', {}).get('temp')
        if temp_celsius:
            processed_data['temperature_celsius'] = round(temp_celsius, 2)
            processed_data['temperature_fahrenheit'] = round(temp_celsius * 9/5 + 32, 2)
            processed_data['temperature_kelvin'] = round(temp_celsius + 273.15, 2)

        # Feels like temperature
        feels_like_celsius = raw_data.get('main', {}).get('feels_like')
        if feels_like_celsius:
            processed_data['feels_like_celsius'] = round(feels_like_celsius, 2)
            processed_data['feels_like_fahrenheit'] = round(feels_like_celsius * 9/5 + 32, 2)

```
