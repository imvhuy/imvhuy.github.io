---
title: "2.2 Building the Lambda Weather Collector"
date: 2025-01-03T08:30:00+07:00
weight: 2
---

In this section, we will create AWS Lambda functions to automatically collect weather data from the OpenWeatherMap API and store it in S3. These functions will be the core of our weather data collection system.

## Step 1: Create IAM Role for Lambda

### 1.1 Create Lambda Execution Role

1. **Navigate to IAM Console**

   - AWS Console → IAM → Roles
   - Click "Create role"

2. **Select Trusted Entity**

   - **Service**: Lambda
   - Click "Next"
     ![Trusted Entity](/images/data-collection/22b1.png)

3. **Configure Role**
   - **Role Name**: `WeatherCollectorLambdaRole`
   - **Description**: `Execution role for weather data collection Lambda functions`

### 1.2 Attach AWS Managed Policies

**What is an AWS Managed Policy?**

AWS has pre-created many common policies that we can attach to a Role instead of writing them from scratch. This saves time and ensures security.

**How to attach policies in the AWS Console:**

1. **When creating the `WeatherCollectorLambdaRole`**, at the "Add permissions" step
2. **"Permissions" tab** → Click "Attach policies directly"
3. **Search for and select each of the following policies:**
   - Type `AWSLambdaBasicExecutionRole` → check the box
   - Type `AmazonS3FullAccess` → check the box
   - Type `AmazonSSMReadOnlyAccess` → check the box
   - Type `CloudWatchAgentServerPolicy` → check the box
4. **Click "Add Permissions"** and finish creating the role

![Managed Policies](/images/data-collection/22b11.png)

**Result:**
The `WeatherCollectorLambdaRole` will have **4 managed policies**.

**This policy allows Lambda to:**

- **Create a CloudWatch Log Group** to write logs
- **Write logs to CloudWatch** when the function runs
- **Basic networking** for Lambda to operate

## Step 2: Create S3 Bucket for Weather Data

### 2.1 Create Weather Data Bucket

1. **Navigate to S3 Console**

   - AWS Console → S3 → Create bucket

![S3](/images/data-collection/22b21.png)

2. **Configure Bucket**

   - **Bucket Name**: `weather-data-{your-account-id}` (replace with your AWS account ID)
   - **Region**: us-east-1 (or your preferred region)
   - **Block Public Access**: Keep all settings enabled (recommended)

![res-S3](/images/data-collection/22b22.png)

3. **Bucket Structure**
   ```
   weather-data-123456789012/
   ├── raw/
       ├── current-weather/
       │   └── year=2025/month=01/day=03/hour=10/
                 └── year=2025/month=01/day=03/
   ```

## Step 3: Lambda Function for Current Weather

### 3.1 Create Current Weather Function

1. **Navigate to Lambda Console**

   - AWS Console → Lambda → Create function
     ![Lambda Console](/images/data-collection/22b31.png)

2. **Configure Function**
   - **Function Name**: `weather-current-collector`
   - **Runtime**: Python 3.11
   - **Architecture**: x86_64
   - **Execution Role**: Use existing role → `WeatherCollectorLambdaRole`

![Create Lambda](/images/data-collection/22b32.png)

### 3.2 Add Function Code

**Important**: Before copying the code, you need to **change the API key**.
In the line `OPENWEATHER_API_KEY = 'API Key`  
→ Replace it with your API key from OpenWeatherMap

1. **In the Lambda Console** → Scroll down to **Code source**
2. **Delete all default code** in the `lambda_function.py` file
3. **Copy and paste the following code:**

**File**: `lambda_function.py`

```python
import json
import boto3
import urllib.request
import urllib.parse
import urllib.error
import os
from datetime import datetime, timezone
from typing import Dict, List, Optional
import logging

# Logging configuration
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# AWS clients with retry configuration
s3_client = boto3.client('s3', config=boto3.session.Config(
    retries={'max_attempts': 3, 'mode': 'adaptive'}
))
cloudwatch = boto3.client('cloudwatch', config=boto3.session.Config(
    retries={'max_attempts': 3, 'mode': 'adaptive'}
))

# Configuration
BUCKET_NAME = os.environ.get('WEATHER_BUCKET_NAME')
OPENWEATHER_API_KEY = 'your_api_key'  # Replace with your actual API key

# Target cities for weather data collection
CITIES = [
    {"name": "HoChiMinh", "lat": 10.7769, "lon": 106.7009},
    {"name": "Hanoi", "lat": 21.0285, "lon": 105.8542},
    {"name": "Danang", "lat": 16.0471, "lon": 108.2068},
    {"name": "GiaLai", "lat": 13.9833, "lon": 108.0000},
    {"name": "CanTho", "lat": 10.0452, "lon": 105.7469},
    {"name": "Hue", "lat": 16.4637, "lon": 107.5909}
]

def fetch_current_weather(city: Dict, api_key: str) -> Optional[Dict]:
    """Fetch current weather data for a city."""
    base_url = "https://api.openweathermap.org/data/2.5/weather"

    params = {
        'lat': str(city['lat']),
        'lon': str(city['lon']),
        'appid': api_key,
        'units': 'metric',
        'lang': 'en'
    }

    try:
        # Build URL with parameters
        query_string = urllib.parse.urlencode(params)
        full_url = f"{base_url}?{query_string}"

        # Make HTTP request
        with urllib.request.urlopen(full_url, timeout=30) as response:
            if response.status != 200:
                logger.error(f"HTTP error {response.status} for {city['name']}")
                return None

            response_data = response.read().decode('utf-8')
            data = json.loads(response_data)

        # Add metadata
        data['collection_timestamp'] = datetime.now(timezone.utc).isoformat()
        data['city_metadata'] = city

        return data

    except urllib.error.URLError as e:
        logger.error(f"URL error for {city['name']}: {e}")
        return None
    except urllib.error.HTTPError as e:
        logger.error(f"HTTP error for {city['name']}: {e}")
        return None
    except json.JSONDecodeError as e:
        logger.error(f"JSON decode error for {city['name']}: {e}")
        return None
    except Exception as e:
        logger.error(f"Unexpected error for {city['name']}: {e}")
        return None

def save_to_s3(data: Dict, city_name: str) -> bool:
    """Save weather data to S3."""
    try:
        if not BUCKET_NAME:
            logger.error("WEATHER_BUCKET_NAME environment variable not set")
            return False

        now = datetime.now(timezone.utc)

        # Create S3 key with time partitioning (Hive format for analytics)
        city_safe = city_name.lower().replace(' ', '_').replace('.', '').replace('-', '_')
        key = f"raw/current-weather/year={now.year}/month={now.month:02d}/day={now.day:02d}/hour={now.hour:02d}/{city_safe}_{now.strftime('%Y%m%d_%H%M%S')}.json"

        # Prepare body data with UTF-8 encoding
        json_data = json.dumps(data, ensure_ascii=False, indent=2)
        body_bytes = json_data.encode('utf-8')

        # Upload file with optimized S3 parameters
        s3_client.put_object(
            Bucket=BUCKET_NAME,
            Key=key,
            Body=body_bytes,
            ContentType='application/json; charset=utf-8',
            ContentEncoding='utf-8',
            Metadata={
                'city': city_name,
                'collection-time': now.isoformat(),
                'data-type': 'current-weather',
                'source': 'openweathermap'
            },
            ServerSideEncryption='AES256',  # Server-side encryption
            StorageClass='STANDARD_IA'  # Save 40% storage cost compared to STANDARD
        )

        logger.info(f"Saved data for {city_name} at s3://{BUCKET_NAME}/{key}")
        return True

    except Exception as e:
        logger.error(f"S3 save error for {city_name}: {e}")
        return False

def send_metrics(metric_name: str, value: float, unit: str = 'Count'):
    """Send custom metrics to CloudWatch."""
    try:
        cloudwatch.put_metric_data(
            Namespace='Weather/ETL',
            MetricData=[
                {
                    'MetricName': metric_name,
                    'Value': value,
                    'Unit': unit,
                    'Timestamp': datetime.now(timezone.utc)
                }
            ]
        )
    except Exception as e:
        logger.error(f"Metric send error {metric_name}: {e}")
```
