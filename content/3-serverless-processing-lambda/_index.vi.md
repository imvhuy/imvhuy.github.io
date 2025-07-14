---
title: "Xử lý và Chuyển đổi Dữ liệu Serverless"
date: 2025-01-07T09:00:00+00:00
weight: 30
chapter: false
pre: "<b>3. </b>"
---

Trong module này, chúng ta sẽ xây dựng Lambda functions để xử lý và chuyển đổi dữ liệu thời tiết thô từ module 2 thành định dạng phù hợp cho phân tích. Đây là bước "Transform" trong pipeline ETL, giúp làm sạch, chuẩn hóa và tạo ra các metrics có ý nghĩa từ dữ liệu thô.


**Dữ liệu thô từ OpenWeatherMap API có nhiều vấn đề**:
- **Cấu trúc phức tạp**: Nested JSON khó query
- **Đơn vị không thống nhất**: Kelvin, m/s, Pascal...
- **Dữ liệu dư thừa**: Nhiều fields không cần thiết
- **Thiếu insights**: Không có derived metrics

**Vì vậy cần phải processing để có:**
- **Cấu trúc phẳng**: Dễ query với SQL
- **Đơn vị thống nhất**: Celsius, km/h, %...
- **Dữ liệu sạch**: Chỉ giữ thông tin cần thiết
- **Rich insights**: Heat index, comfort level, weather severity...

**Luồng xử lý:**

1. **Raw data** từ Module 2 được lưu vào S3
2. **S3 Event** trigger Lambda processor
3. **Lambda** transform data và lưu vào processed bucket
4. **Processed data** sẵn sàng cho analytics (Module 4)

## Bước 1: Tạo S3 Bucket cho Processed Data

**Tại sao cần chia bucket?**

- **Tách biệt concerns**: Raw vs Processed data
- **Security**: Khác nhau về access permissions
- **Cost optimization**: Khác nhau về storage class
- **Analytics**: Processed data tối ưu cho query


### 1.1 Tạo Processed Data Bucket

1. **AWS Console** → **S3** → **Create bucket**

2. **Bucket configuration**:
   - **Bucket name**: `weather-processed-{your-account-id}`
   - **Region**: `ap-southeast-1` (same as raw bucket)
   - **Block all public access**: Enabled
   - **Bucket versioning**: Enabled
   - **Default encryption**: SSE-S3

![Create Processed Bucket](/images/data-processing/3b1.png)

3. **Create bucket**

### 1.2 Tạo Folder Structure

**Tạo folder structure tối ưu cho analytics:**

```
weather-processed-{account-id}/
├── current-weather/
│   ├── year=2025/
│   │   ├── month=01/
│   │   │   ├── day=03/
│   │   │   │   ├── hour=00/
│   │   │   │   │   ├── hcm_20250103_000000.json
│   │   │   │   │   ├── hanoi_20250103_000000.json
│   │   │   │   │   └── ...
│   │   │   │   └── hour=01/
│   │   │   └── day=04/
│   │   └── month=02/
│   └── year=2026/
└── forecast/
    ├── year=2025/
    │   ├── month=01/
    │   │   ├── day=03/
    │   │   │   ├── hour=00/
    │   │   │   │   ├── hcm_forecast_20250103_000000.json
    │   │   │   │   └── ...
    │   │   │   └── hour=06/
    │   │   └── day=04/
    │   └── month=02/
    └── year=2026/
```

{{% notice info %}}
**Tại sao dùng Hive-style partitioning?**

- 📊 **Athena optimization**: Query performance tốt hơn
- 💰 **Cost savings**: Chỉ scan data cần thiết
- 🔍 **Easy filtering**: Filter theo year/month/day/hour
- 📈 **Scalability**: Handle large datasets efficiently
  {{% /notice %}}

## Bước 2: Tạo Lambda Function cho Data Processing

### 2.1 Tạo IAM Role cho Lambda

1. **AWS Console** → **IAM** → **Roles** → **Create role**

2. **Trusted entity**: **AWS service** → **Lambda**

3. **Permissions**: **Create policy** với JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::weather-data-*/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:PutObjectAcl"],
      "Resource": "arn:aws:s3:::weather-processed-*/*"
    },
    {
      "Effect": "Allow",
      "Action": ["cloudwatch:PutMetricData"],
      "Resource": "*"
    }
  ]
}
```

4. **Role name**: `WeatherDataProcessorRole`

![IAM Role](/images/data-processing/31b02.png)

### 2.2 Tạo Lambda Function

1. **AWS Console** → **Lambda** → **Create function**

2. **Function configuration**:
   - **Function name**: `weather-data-processor`
   - **Runtime**: `Python 3.11`
   - **Architecture**: `x86_64`
   - **Execution role**: `WeatherDataProcessorRole`

![Create Lambda](/images/data-processing/31b03.png)

3. **Advanced settings**:
   - **Memory**: `512 MB`
   - **Timeout**: `5 minutes`
   - **Environment variables**:
     - `PROCESSED_BUCKET_NAME`: `weather-processed-{your-account-id}`
     - `LOG_LEVEL`: `INFO`

![Lambda Configuration](/images/data-processing/31b04.png)

### 2.3 Lambda Function Code

**Thay thế code mặc định bằng code sau:**

```python
import json
import boto3
import os
import logging
from datetime import datetime, timezone
from typing import Dict, List, Optional, Any
from decimal import Decimal
import urllib.parse

# Logging configuration
logger = logging.getLogger()
logger.setLevel(os.environ.get('LOG_LEVEL', 'INFO'))

# AWS clients
s3_client = boto3.client('s3')
cloudwatch = boto3.client('cloudwatch')

# Configuration
PROCESSED_BUCKET_NAME = os.environ.get('PROCESSED_BUCKET_NAME')

def lambda_handler(event, context):
    """
    Main Lambda handler for weather data processing
    """
    logger.info(f"Processing event: {json.dumps(event)}")

    processed_count = 0
    failed_count = 0

    try:
        # Handle S3 event trigger
        if 'Records' in event:
            for record in event['Records']:
                try:
                    # Extract S3 information
                    bucket = record['s3']['bucket']['name']
                    key = urllib.parse.unquote_plus(record['s3']['object']['key'])

                    logger.info(f"Processing file: s3://{bucket}/{key}")

                    # Process the file
                    success = process_weather_file(bucket, key)

                    if success:
                        processed_count += 1
                    else:
                        failed_count += 1

                except Exception as e:
                    logger.error(f"Error processing record: {e}")
                    failed_count += 1

        # Handle manual invocation
        else:
            logger.info("Manual invocation - processing recent files")
            # This could be extended to process recent files
            processed_count = 1

        # Send metrics to CloudWatch
        send_processing_metrics(processed_count, failed_count)

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Weather data processing completed',
                'processed_count': processed_count,
                'failed_count': failed_count
            })
        }

    except Exception as e:
        logger.error(f"Lambda handler error: {e}")

        # Send error metric
        send_processing_metrics(0, 1)

        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': 'Internal processing error',
                'message': str(e)
            })
        }

def process_weather_file(source_bucket: str, source_key: str) -> bool:
    """
    Process a single weather data file
    """
    try:
        # Download raw data from S3
        response = s3_client.get_object(Bucket=source_bucket, Key=source_key)
        raw_data = json.loads(response['Body'].read().decode('utf-8'))

        # Determine data type based on key path
        if 'current-weather' in source_key:
            processed_data = transform_current_weather(raw_data)
            data_type = 'current-weather'
        elif 'forecast' in source_key:
            processed_data = transform_forecast_weather(raw_data)
            data_type = 'forecast'
        else:
            logger.warning(f"Unknown data type for key: {source_key}")
            return False

        # Generate processed file key
        processed_key = generate_processed_key(source_key, data_type)

        # Save processed data to S3
        s3_client.put_object(
            Bucket=PROCESSED_BUCKET_NAME,
            Key=processed_key,
            Body=json.dumps(processed_data, indent=2, ensure_ascii=False, default=decimal_default),
            ContentType='application/json; charset=utf-8',
            Metadata={
                'source-key': source_key,
                'processing-timestamp': datetime.now(timezone.utc).isoformat(),
                'data-type': data_type,
                'city': processed_data.get('city_name', 'unknown')
            }
        )

        logger.info(f"Successfully processed and saved: s3://{PROCESSED_BUCKET_NAME}/{processed_key}")
        return True

    except Exception as e:
        logger.error(f"Error processing file {source_key}: {e}")
        return False

def transform_current_weather(raw_data: Dict) -> Dict:
    """
    Transform current weather data from OpenWeatherMap format to analytics format
    """
    try:
        # Extract basic information
        city_name = raw_data.get('name', 'Unknown')

        # Get collection timestamp from metadata (added by our collector)
        collection_timestamp = raw_data.get('collection_timestamp')
        if collection_timestamp:
            dt = datetime.fromisoformat(collection_timestamp.replace('Z', '+00:00'))
        else:
            # Fallback to API timestamp
            dt = datetime.fromtimestamp(raw_data.get('dt', 0), tz=timezone.utc)

        # Base processed data structure
        processed = {
            # Identifiers
            'city_name': city_name,
            'city_id': raw_data.get('id'),
            'country_code': raw_data.get('sys', {}).get('country', 'Unknown'),
            'latitude': raw_data.get('coord', {}).get('lat'),
            'longitude': raw_data.get('coord', {}).get('lon'),

            # Timestamps
            'collection_timestamp': dt.isoformat(),
            'collection_date': dt.strftime('%Y-%m-%d'),
            'collection_hour': dt.hour,
            'api_timestamp': datetime.fromtimestamp(raw_data.get('dt', 0), tz=timezone.utc).isoformat(),

            # Weather conditions
            'weather_id': None,
            'weather_main': None,
            'weather_description': None,
            'weather_icon': None,

            # Temperature (convert from Kelvin to Celsius)
            'temperature_celsius': None,
            'temperature_fahrenheit': None,
            'feels_like_celsius': None,
            'feels_like_fahrenheit': None,
            'temp_min_celsius': None,
            'temp_max_celsius': None,

            # Atmospheric conditions
            'pressure_hpa': raw_data.get('main', {}).get('pressure'),
            'humidity_percent': raw_data.get('main', {}).get('humidity'),
            'visibility_meters': raw_data.get('visibility'),

            # Wind
            'wind_speed_ms': raw_data.get('wind', {}).get('speed'),
            'wind_speed_kmh': None,
            'wind_direction_deg': raw_data.get('wind', {}).get('deg'),
            'wind_gust_ms': raw_data.get('wind', {}).get('gust'),

            # Clouds and precipitation
            'cloud_coverage_percent': raw_data.get('clouds', {}).get('all'),
            'rain_1h_mm': raw_data.get('rain', {}).get('1h'),
            'rain_3h_mm': raw_data.get('rain', {}).get('3h'),
            'snow_1h_mm': raw_data.get('snow', {}).get('1h'),
            'snow_3h_mm': raw_data.get('snow', {}).get('3h'),

            # Sun times
            'sunrise_timestamp': None,
            'sunset_timestamp': None,

            # Derived metrics (calculated later)
            'heat_index_celsius': None,
            'comfort_level': None,
            'wind_condition': None,
            'weather_severity': None,
            'uv_risk_level': None
        }

        # Process weather conditions
        weather_list = raw_data.get('weather', [])
        if weather_list:
            weather = weather_list[0]
            processed.update({
                'weather_id': weather.get('id'),
                'weather_main': weather.get('main'),
                'weather_description': weather.get('description'),
                'weather_icon': weather.get('icon')
            })

        # Process temperature data
        main_data = raw_data.get('main', {})
        if main_data.get('temp'):
            temp_k = main_data['temp']
            processed['temperature_celsius'] = round(temp_k - 273.15, 1)
            processed['temperature_fahrenheit'] = round((temp_k - 273.15) * 9/5 + 32, 1)

        if main_data.get('feels_like'):
            feels_k = main_data['feels_like']
            processed['feels_like_celsius'] = round(feels_k - 273.15, 1)
            processed['feels_like_fahrenheit'] = round((feels_k - 273.15) * 9/5 + 32, 1)

        if main_data.get('temp_min'):
            min_k = main_data['temp_min']
            processed['temp_min_celsius'] = round(min_k - 273.15, 1)

        if main_data.get('temp_max'):
            max_k = main_data['temp_max']
            processed['temp_max_celsius'] = round(max_k - 273.15, 1)

        # Process wind data
        if processed['wind_speed_ms']:
            processed['wind_speed_kmh'] = round(processed['wind_speed_ms'] * 3.6, 1)

        # Process sun times
        sys_data = raw_data.get('sys', {})
        if sys_data.get('sunrise'):
            processed['sunrise_timestamp'] = datetime.fromtimestamp(sys_data['sunrise'], tz=timezone.utc).isoformat()
        if sys_data.get('sunset'):
            processed['sunset_timestamp'] = datetime.fromtimestamp(sys_data['sunset'], tz=timezone.utc).isoformat()

        # Calculate derived metrics
        processed.update(calculate_derived_metrics(processed))

        return processed

    except Exception as e:
        logger.error(f"Error transforming current weather data: {e}")
        raise

def transform_forecast_weather(raw_data: Dict) -> List[Dict]:
    """
    Transform forecast weather data from OpenWeatherMap format to analytics format
    """
    try:
        # Extract city information
        city_info = raw_data.get('city', {})
        city_name = city_info.get('name', 'Unknown')

        processed_forecasts = []

        # Process each forecast item
        for item in raw_data.get('list', []):
            # Get forecast timestamp
            dt = datetime.fromtimestamp(item.get('dt', 0), tz=timezone.utc)

            # Base forecast data structure
            forecast = {
                # Identifiers
                'city_name': city_name,
                'city_id': city_info.get('id'),
                'country_code': city_info.get('country', 'Unknown'),
                'latitude': city_info.get('coord', {}).get('lat'),
                'longitude': city_info.get('coord', {}).get('lon'),

                # Timestamps
                'forecast_timestamp': dt.isoformat(),
                'forecast_date': dt.strftime('%Y-%m-%d'),
                'forecast_hour': dt.hour,
                'collection_timestamp': raw_data.get('collection_timestamp'),

                # Weather conditions
                'weather_id': None,
                'weather_main': None,
                'weather_description': None,
                'weather_icon': None,

                # Temperature
                'temperature_celsius': None,
                'temperature_fahrenheit': None,
                'feels_like_celsius': None,
                'feels_like_fahrenheit': None,
                'temp_min_celsius': None,
                'temp_max_celsius': None,

                # Atmospheric conditions
                'pressure_hpa': item.get('main', {}).get('pressure'),
                'humidity_percent': item.get('main', {}).get('humidity'),
                'visibility_meters': item.get('visibility'),

                # Wind
                'wind_speed_ms': item.get('wind', {}).get('speed'),
                'wind_speed_kmh': None,
                'wind_direction_deg': item.get('wind', {}).get('deg'),
                'wind_gust_ms': item.get('wind', {}).get('gust'),

                # Clouds and precipitation
                'cloud_coverage_percent': item.get('clouds', {}).get('all'),
                'rain_3h_mm': item.get('rain', {}).get('3h'),
                'snow_3h_mm': item.get('snow', {}).get('3h'),
                'precipitation_probability': item.get('pop', 0) * 100,  # Convert to percentage

                # Derived metrics
                'heat_index_celsius': None,
                'comfort_level': None,
                'wind_condition': None,
                'weather_severity': None
            }

            # Process weather conditions
            weather_list = item.get('weather', [])
            if weather_list:
                weather = weather_list[0]
                forecast.update({
                    'weather_id': weather.get('id'),
                    'weather_main': weather.get('main'),
                    'weather_description': weather.get('description'),
                    'weather_icon': weather.get('icon')
                })

            # Process temperature data
            main_data = item.get('main', {})
            if main_data.get('temp'):
                temp_k = main_data['temp']
                forecast['temperature_celsius'] = round(temp_k - 273.15, 1)
                forecast['temperature_fahrenheit'] = round((temp_k - 273.15) * 9/5 + 32, 1)

            if main_data.get('feels_like'):
                feels_k = main_data['feels_like']
                forecast['feels_like_celsius'] = round(feels_k - 273.15, 1)
                forecast['feels_like_fahrenheit'] = round((feels_k - 273.15) * 9/5 + 32, 1)

            if main_data.get('temp_min'):
                min_k = main_data['temp_min']
                forecast['temp_min_celsius'] = round(min_k - 273.15, 1)

            if main_data.get('temp_max'):
                max_k = main_data['temp_max']
                forecast['temp_max_celsius'] = round(max_k - 273.15, 1)

            # Process wind data
            if forecast['wind_speed_ms']:
                forecast['wind_speed_kmh'] = round(forecast['wind_speed_ms'] * 3.6, 1)

            # Calculate derived metrics
            forecast.update(calculate_derived_metrics(forecast))

            processed_forecasts.append(forecast)

        return processed_forecasts

    except Exception as e:
        logger.error(f"Error transforming forecast weather data: {e}")
        raise

def calculate_derived_metrics(data: Dict) -> Dict:
    """
    Calculate derived weather metrics
    """
    derived = {}

    try:
        temp_c = data.get('temperature_celsius')
        humidity = data.get('humidity_percent')
        wind_speed_kmh = data.get('wind_speed_kmh')
        weather_main = data.get('weather_main', '').lower()

        # Calculate heat index (only meaningful above 27°C)
        if temp_c and humidity and temp_c >= 27:
            temp_f = temp_c * 9/5 + 32

            # Simplified heat index calculation
            if temp_f >= 80:
                heat_index_f = (
                    -42.379 +
                    2.04901523 * temp_f +
                    10.14333127 * humidity -
                    0.22475541 * temp_f * humidity -
                    6.83783e-3 * temp_f**2 -
                    5.481717e-2 * humidity**2 +
                    1.22874e-3 * temp_f**2 * humidity +
                    8.5282e-4 * temp_f * humidity**2 -
                    1.99e-6 * temp_f**2 * humidity**2
                )
                derived['heat_index_celsius'] = round((heat_index_f - 32) * 5/9, 1)

        # Calculate comfort level
        if temp_c and humidity:
            if temp_c < 16:
                comfort = 'cold'
            elif temp_c < 20:
                comfort = 'cool'
            elif temp_c <= 26 and humidity <= 60:
                comfort = 'comfortable'
            elif temp_c <= 30 and humidity <= 70:
                comfort = 'warm'
            elif temp_c <= 35:
                comfort = 'hot'
            else:
                comfort = 'very_hot'
            derived['comfort_level'] = comfort

        # Calculate wind condition
        if wind_speed_kmh:
            if wind_speed_kmh < 5:
                wind_condition = 'calm'
            elif wind_speed_kmh < 20:
                wind_condition = 'light'
            elif wind_speed_kmh < 40:
                wind_condition = 'moderate'
            elif wind_speed_kmh < 60:
                wind_condition = 'strong'
            else:
                wind_condition = 'very_strong'
            derived['wind_condition'] = wind_condition

        # Calculate weather severity
        if weather_main:
            if weather_main in ['thunderstorm', 'tornado']:
                severity = 'severe'
            elif weather_main in ['rain', 'snow', 'drizzle']:
                severity = 'moderate'
            elif weather_main in ['mist', 'fog', 'haze', 'smoke']:
                severity = 'mild'
            else:
                severity = 'normal'
            derived['weather_severity'] = severity

        return derived

    except Exception as e:
        logger.error(f"Error calculating derived metrics: {e}")
        return {}

def generate_processed_key(source_key: str, data_type: str) -> str:
    """
    Generate processed file key with proper partitioning
    """
    try:
        # Extract timestamp from source key
        now = datetime.now(timezone.utc)

        # Extract city name from source key
        key_parts = source_key.split('/')
        filename = key_parts[-1]
        city_name = filename.split('_')[0]

        # Generate processed key with Hive partitioning
        processed_key = (
            f"{data_type}/"
            f"year={now.year}/"
            f"month={now.month:02d}/"
            f"day={now.day:02d}/"
            f"hour={now.hour:02d}/"
            f"{city_name}_processed_{now.strftime('%Y%m%d_%H%M%S')}.json"
        )

        return processed_key

    except Exception as e:
        logger.error(f"Error generating processed key: {e}")
        return f"processed/{source_key}"

def send_processing_metrics(processed_count: int, failed_count: int):
    """
    Send processing metrics to CloudWatch
    """
    try:
        cloudwatch.put_metric_data(
            Namespace='Weather/Processing',
            MetricData=[
                {
                    'MetricName': 'ProcessedFiles',
                    'Value': processed_count,
                    'Unit': 'Count',
                    'Timestamp': datetime.now(timezone.utc)
                },
                {
                    'MetricName': 'FailedFiles',
                    'Value': failed_count,
                    'Unit': 'Count',
                    'Timestamp': datetime.now(timezone.utc)
                },
                {
                    'MetricName': 'ProcessingSuccess',
                    'Value': 1 if failed_count == 0 else 0,
                    'Unit': 'Count',
                    'Timestamp': datetime.now(timezone.utc)
                }
            ]
        )
        logger.info(f"Sent metrics - Processed: {processed_count}, Failed: {failed_count}")

    except Exception as e:
        logger.error(f"Error sending metrics: {e}")

def decimal_default(obj):
    """
    JSON serializer for objects not serializable by default json code
    """
    if isinstance(obj, Decimal):
        return float(obj)
    raise TypeError(f"Object of type {type(obj)} is not JSON serializable")
```

4. **Deploy** code bằng cách click **"Deploy"**

![Deploy Lambda](/images/data-processing/31b05.png)

## Bước 3: Thiết lập S3 Event Trigger

{{% notice warning %}}
**Quan trọng**: S3 Event Trigger sẽ tự động chạy Lambda mỗi khi có file mới được upload vào raw bucket. Đảm bảo Lambda function hoạt động đúng trước khi enable trigger.
{{% /notice %}}

### 3.1 Thêm Permission cho S3 invoke Lambda

1. **Lambda Console** → **weather-data-processor** → **Configuration** → **Permissions**

2. **Resource-based policy** → **Add permissions**:
   - **Principal**: `s3.amazonaws.com`
   - **Source ARN**: `arn:aws:s3:::weather-data-{your-account-id}`
   - **Action**: `lambda:InvokeFunction`

![Lambda Permissions](/images/data-processing/31b06.png)

### 3.2 Configure S3 Event Notification

1. **S3 Console** → **weather-data-{your-account-id}** bucket → **Properties**

2. **Event notifications** → **Create event notification**:
   - **Event name**: `weather-data-processing-trigger`
   - **Event types**: ✅ **All object create events**
   - **Prefix**: `raw/`
   - **Suffix**: `.json`
   - **Destination**: **Lambda function** → **weather-data-processor**

![S3 Event Notification](/images/data-processing/31b07.png)

3. **Save changes**

## Bước 4: Testing Data Processing

### 4.1 Test với Manual Upload

**Upload một file test để trigger processing:**

1. **Tạo test file** `test_weather.json`:

```json
{
  "coord": { "lon": 106.6297, "lat": 10.8231 },
  "weather": [
    {
      "id": 800,
      "main": "Clear",
      "description": "clear sky",
      "icon": "01d"
    }
  ],
  "main": {
    "temp": 305.15,
    "feels_like": 309.65,
    "temp_min": 305.15,
    "temp_max": 305.15,
    "pressure": 1013,
    "humidity": 74
  },
  "wind": { "speed": 3.2, "deg": 220 },
  "clouds": { "all": 0 },
  "dt": 1704268800,
  "sys": {
    "country": "VN",
    "sunrise": 1704225600,
    "sunset": 1704268800
  },
  "timezone": 25200,
  "id": 1566083,
  "name": "HoChiMinh",
  "collection_timestamp": "2025-01-03T08:00:00Z"
}
```

2. **Upload file**:

```bash
aws s3 cp test_weather.json s3://weather-data-{your-account-id}/raw/current-weather/year=2025/month=01/day=03/hour=08/hcm_20250103_080000.json
```

3. **Check CloudWatch Logs**:
   - **CloudWatch Console** → **Log groups** → `/aws/lambda/weather-data-processor`
   - **Latest log stream** → Verify processing logs

![CloudWatch Logs](/images/data-processing/31b08.png)

### 4.2 Verify Processed Data

1. **Check processed bucket**:

```bash
aws s3 ls s3://weather-processed-{your-account-id}/current-weather/ --recursive
```

2. **Download và examine processed file**:

```bash
aws s3 cp s3://weather-processed-{your-account-id}/current-weather/year=2025/month=01/day=03/hour=08/hcm_processed_20250103_080000.json ./
cat hcm_processed_20250103_080000.json | jq .
```

**Expected processed output:**

```json
{
  "city_name": "HoChiMinh",
  "country_code": "VN",
  "latitude": 10.8231,
  "longitude": 106.6297,
  "collection_timestamp": "2025-01-03T08:00:00+00:00",
  "collection_date": "2025-01-03",
  "collection_hour": 8,
  "weather_main": "Clear",
  "weather_description": "clear sky",
  "temperature_celsius": 32.0,
  "temperature_fahrenheit": 89.6,
  "feels_like_celsius": 36.5,
  "feels_like_fahrenheit": 97.7,
  "humidity_percent": 74,
  "pressure_hpa": 1013,
  "wind_speed_ms": 3.2,
  "wind_speed_kmh": 11.5,
  "wind_direction_deg": 220,
  "cloud_coverage_percent": 0,
  "comfort_level": "hot",
  "wind_condition": "light",
  "weather_severity": "normal"
}
```

### 4.3 Test với Real Data từ Module 2

**Nếu EventBridge rules từ Module 2 đang chạy:**

1. **Wait for scheduled collection** (next hour hoặc 6-hour interval)

2. **Check logs** để verify automatic processing:

```bash
aws logs filter-log-events \
    --log-group-name /aws/lambda/weather-data-processor \
    --start-time $(date -d '1 hour ago' +%s)000 \
    --filter-pattern "Processing file"
```

3. **Verify processed data structure**:

```bash
aws s3 ls s3://weather-processed-{your-account-id}/ --recursive | head -20
```

## Bước 5: Monitoring và Optimization

### 5.1 CloudWatch Metrics Dashboard

1. **CloudWatch Console** → **Dashboards** → **Create dashboard**

2. **Dashboard name**: `Weather-Data-Processing`

3. **Add widgets**:

**Widget 1 - Processing Success Rate:**

- **Metric**: `Weather/Processing > ProcessedFiles`
- **Metric**: `Weather/Processing > FailedFiles`
- **Period**: 1 hour
- **Statistic**: Sum

**Widget 2 - Lambda Performance:**

- **Metric**: `AWS/Lambda > Duration` (function: weather-data-processor)
- **Metric**: `AWS/Lambda > Errors` (function: weather-data-processor)
- **Period**: 5 minutes

**Widget 3 - S3 Object Count:**

- **Metric**: `AWS/S3 > NumberOfObjects` (bucket: weather-processed-\*)
- **Period**: 1 day

![Processing Dashboard](/images/data-processing/31b09.png)

### 5.2 CloudWatch Alarms

**Tạo alarm cho processing failures:**

1. **Alarm name**: `WeatherProcessing-HighFailureRate`
2. **Metric**: `Weather/Processing > FailedFiles`
3. **Condition**: `Greater than 2` (trong 1 hour)
4. **Action**: Send to SNS topic từ Module 2

**Tạo alarm cho Lambda errors:**

1. **Alarm name**: `WeatherProcessor-LambdaErrors`
2. **Metric**: `AWS/Lambda > Errors`
3. **Condition**: `Greater than 1` (trong 5 minutes)
4. **Action**: Send to SNS topic

### 5.3 Cost Optimization

**Optimize Lambda configuration:**

1. **Memory**: Monitor actual usage và adjust accordingly
2. **Timeout**: Reduce nếu processing time stable
3. **Reserved concurrency**: Set limit để avoid cost spikes

**S3 Storage optimization:**

1. **Lifecycle policies** cho processed data:
   - After 30 days: Move to IA
   - After 90 days: Move to Glacier
   - After 365 days: Move to Deep Archive

![S3 Lifecycle](/images/data-processing/31b10.png)

## Data Comparison: Raw vs Processed

### Raw Data (từ Module 2)

```json
{
  "coord": { "lon": 106.6297, "lat": 10.8231 },
  "weather": [
    { "id": 800, "main": "Clear", "description": "clear sky", "icon": "01d" }
  ],
  "main": {
    "temp": 305.15,
    "feels_like": 309.65,
    "pressure": 1013,
    "humidity": 74
  },
  "wind": { "speed": 3.2, "deg": 220 },
  "clouds": { "all": 0 },
  "dt": 1704268800,
  "sys": { "country": "VN", "sunrise": 1704225600, "sunset": 1704268800 },
  "timezone": 25200,
  "id": 1566083,
  "name": "HoChiMinh",
  "collection_timestamp": "2025-01-03T08:00:00Z"
}
```

### Processed Data (sau Module 3)

```json
{
  "city_name": "HoChiMinh",
  "country_code": "VN",
  "latitude": 10.8231,
  "longitude": 106.6297,
  "collection_timestamp": "2025-01-03T08:00:00+00:00",
  "collection_date": "2025-01-03",
  "collection_hour": 8,
  "weather_main": "Clear",
  "weather_description": "clear sky",
  "temperature_celsius": 32.0,
  "temperature_fahrenheit": 89.6,
  "feels_like_celsius": 36.5,
  "feels_like_fahrenheit": 97.7,
  "humidity_percent": 74,
  "pressure_hpa": 1013,
  "wind_speed_ms": 3.2,
  "wind_speed_kmh": 11.5,
  "wind_direction_deg": 220,
  "cloud_coverage_percent": 0,
  "comfort_level": "hot",
  "wind_condition": "light",
  "weather_severity": "normal"
}
```

**Key improvements:**

- ✅ **Flat structure**: Dễ query với SQL
- ✅ **Standardized units**: Celsius, km/h, %
- ✅ **Rich metadata**: Date, hour partitioning
- ✅ **Derived insights**: Comfort level, wind condition, weather severity
- ✅ **Analytics-ready**: Optimized cho Athena queries

## Troubleshooting Common Issues

### Processing Failures

**Issue**: Lambda function timeout
**Solution**:

- Increase timeout (max 15 minutes)
- Optimize code performance
- Process files in batches

**Issue**: S3 permission denied
**Solution**:

- Verify IAM role permissions
- Check bucket policies
- Ensure correct bucket names

**Issue**: JSON parsing errors
**Solution**:

- Add error handling for malformed JSON
- Log problematic files for manual inspection
- Implement retry logic

### Performance Issues

**Issue**: High processing latency
**Solution**:

- Increase Lambda memory (more CPU)
- Optimize data transformation logic
- Use concurrent processing

**Issue**: High costs
**Solution**:

- Right-size Lambda memory
- Implement S3 lifecycle policies
- Use reserved concurrency limits

## Tóm tắt

Trong module này, chúng ta đã hoàn thành:

**Xây dựng Data Processing Pipeline**:

- Lambda function xử lý real-time data từ S3 events
- Transform raw weather data thành analytics-ready format
- Automatic partitioning cho optimal query performance

**Data Transformation Features**:

- Convert units (Kelvin → Celsius, m/s → km/h)
- Calculate derived metrics (heat index, comfort level)
- Clean và normalize data structure
- Add rich metadata cho analytics

**Monitoring và Optimization**:

- CloudWatch metrics và alarms
- Performance monitoring dashboard
- Cost optimization strategies
- Error handling và retry logic

**Kết quả**: Dữ liệu thời tiết clean, structured, và ready cho advanced analytics trong Module 4!

**Tiếp theo**: Trong Module 4, chúng ta sẽ sử dụng Amazon Athena để query và analyze processed data này một cách hiệu quả.
