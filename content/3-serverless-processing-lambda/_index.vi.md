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
   - **Bucket versioning**: Disable
   - **Default encryption**: SSE-S3

![Create Processed Bucket](/images/data-processing/3b1.png)

3. **Create bucket**

### 1.2 Tạo Folder Structure

**Tạo folder structure tối ưu cho analytics:**

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

**Ở đây sẽ dùng Hive-style partitioning vì:**

- **Athena optimization**: Query performance tốt hơn
- **Cost savings**: Chỉ scan data cần thiết
- **Easy filtering**: Filter theo year/month/day/hour
- **Scalability**: Xử lý các bộ dữ liệu lớn một cách hiệu quả

## Bước 2: Tạo Lambda Function cho Data Processing

### 2.1 Tạo Lambda Function

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

**Thay thế code mặc định bằng code sau:**

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

                # Save processed data to S3 dưới dạng NDJSON (jsonl)
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

        # Other weather parameters
        processed_data.update({
            'humidity_percent': raw_data.get('main', {}).get('humidity'),
            'pressure_hpa': raw_data.get('main', {}).get('pressure'),
            'visibility_meters': raw_data.get('visibility'),
            'uv_index': raw_data.get('uvi')  # If available
        })

        # Weather description
        weather_list = raw_data.get('weather', [])
        if weather_list:
            weather = weather_list[0]
            processed_data.update({
                'weather_id': weather.get('id'),
                'weather_main': weather.get('main'),
                'weather_description': weather.get('description'),
                'weather_icon': weather.get('icon')
            })

        # Wind information
        wind_data = raw_data.get('wind', {})
        processed_data.update({
            'wind_speed_ms': wind_data.get('speed'),
            'wind_direction_deg': wind_data.get('deg'),
            'wind_gust_ms': wind_data.get('gust')
        })

        # Convert wind speed to km/h and mph
        if wind_data.get('speed'):
            processed_data['wind_speed_kmh'] = round(wind_data['speed'] * 3.6, 2)
            processed_data['wind_speed_mph'] = round(wind_data['speed'] * 2.237, 2)

        # Cloud coverage
        processed_data['cloud_coverage_percent'] = raw_data.get('clouds', {}).get('all')

        # Precipitation (if available)
        rain_data = raw_data.get('rain', {})
        if rain_data:
            processed_data['rain_1h_mm'] = rain_data.get('1h')
            processed_data['rain_3h_mm'] = rain_data.get('3h')

        snow_data = raw_data.get('snow', {})
        if snow_data:
            processed_data['snow_1h_mm'] = snow_data.get('1h')
            processed_data['snow_3h_mm'] = snow_data.get('3h')

        # Add derived fields
        processed_data.update(calculate_derived_fields(processed_data))

        return processed_data

    except Exception as e:
        logger.error(f"Weather data transformation error: {str(e)}")
        raise

def calculate_derived_fields(data):
    """
    Calculate derived weather indicators
    """
    derived = {}

    try:
        # Calculate heat index (simplified)
        temp_f = data.get('temperature_fahrenheit')
        humidity = data.get('humidity_percent')

        if temp_f and humidity:
            if temp_f >= 80:  # Heat index only meaningful above 80°F
                # Simplified heat index formula
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
                derived['heat_index_fahrenheit'] = round(heat_index_f, 2)
                derived['heat_index_celsius'] = round((heat_index_f - 32) * 5/9, 2)

        # Comfort level based on temperature and humidity
        temp_c = data.get('temperature_celsius')
        if temp_c and humidity:
            if temp_c < 10:
                comfort = 'cold'
            elif temp_c < 18:
                comfort = 'cool'
            elif temp_c <= 24 and humidity <= 60:
                comfort = 'comfortable'
            elif temp_c <= 30 and humidity <= 70:
                comfort = 'warm'
            else:
                comfort = 'hot'
            derived['comfort_level'] = comfort

        # Wind condition
        wind_speed_kmh = data.get('wind_speed_kmh')
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

        # Weather severity level
        weather_main = data.get('weather_main', '').lower()
        if weather_main:
            if weather_main in ['thunderstorm', 'tornado']:
                severity = 'severe'
            elif weather_main in ['rain', 'snow', 'drizzle']:
                severity = 'moderate'
            elif weather_main in ['mist', 'fog', 'haze']:
                severity = 'light'
            else:
                severity = 'normal'
            derived['weather_severity'] = severity

        return derived

    except Exception as e:
        logger.error(f"Error calculating derived fields: {str(e)}")
        return {}

def decimal_default(obj):
    """
    JSON serializer for objects that are not serializable by default
    """
    if isinstance(obj, Decimal):
        return float(obj)
    raise TypeError
```

4. **Deploy** code bằng cách click **"Deploy"**

![Deploy Lambda](/images/data-processing/3b4.png)

## Bước 3: Thiết lập S3 Event Trigger

**Quan trọng**: S3 Event Trigger sẽ tự động chạy Lambda mỗi khi có file mới được upload vào raw bucket. Đảm bảo Lambda function hoạt động đúng trước khi enable trigger.

### 3.1 Thêm Permission cho S3 invoke Lambda

1. **Lambda Console** → **weather-data-processor** → **Configuration** → **Permissions**

2. **Resource-based policy** → **Add permissions**:
   - **Principal**: `s3.amazonaws.com`
   - **Source ARN**: `arn:aws:s3:::weather-data-{your-account-id}`
   - **Action**: `lambda:InvokeFunction`

![Lambda Permissions](/images/data-processing/3b5.png)

### 3.2 Configure S3 Event Notification

1. **S3 Console** → **weather-data-{your-account-id}** bucket → **Properties**

2. **Event notifications** → **Create event notification**:
   - **Event name**: `weather-data-processing-trigger`
   - **Event types**: **All object create events**
   - **Prefix**: `raw/`
   - **Suffix**: `.json`
   - **Destination**: **Lambda function** → **weather-data-processor**

![S3 Event Notification](/images/data-processing/3b6.png)

3. **Save changes**

## Bước 4: Testing Data Processing

### Test với Real Data từ Module 2

**Nếu EventBridge rules từ Module 2 đang chạy:**

1. **Chờ Lambda tự động chạy**: EventBridge sẽ trigger functions theo lịch:

   - **Current weather**: Mỗi giờ

2. **Monitoring real-time processing**:

   ```bash
   # Check raw data được tạo
   aws s3 ls s3://weather-data-{your-account-id}/raw/current-weather/ --recursive

   # Check processed data được tạo
   aws s3 ls s3://weather-processed-{your-account-id}/current-weather/ --recursive
   ```

3. **CloudWatch Logs real-time**:
   - **Lambda Console** → **Functions** → **weather-data-processor** → **Monitor** → **View logs in CloudWatch**

**Expected workflow**:

```
Module 2 EventBridge → weather-current-collector → S3 raw/current-weather/
                        ↓ (S3 Event)
                      weather-data-processor → S3 processed/current-weather/
```

## Tổng kết

**Đã hoàn thành:**

- **S3 Processed Bucket**: Storage tối ưu cho analytics
- **Lambda Data Processor**: Transform current weather data
- **S3 Event Triggers**: Tự động processing khi có raw data
- **Data Transformation**: Clean, enrich, standardize weather data
- **Hive Partitioning**: Analytics-ready folder structure
- **Error Handling**: Robust error handling và retry logic

**Kết quả:**

- Weather data được transform từ complex JSON → flat structure
- Temperature converted từ Kelvin → Celsius/Fahrenheit
- Derived metrics: comfort level, wind condition, weather severity
- Partitioned data cho efficient querying
- Real-time processing pipeline hoạt động 24/7

**Sẵn sàng cho Module 4**: Data Analytics và Visualization!
