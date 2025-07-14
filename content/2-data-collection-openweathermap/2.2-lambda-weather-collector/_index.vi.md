---
title: "Xây dựng Lambda Weather Collector"
date: 2025-01-03T08:30:00+07:00
weight: 2
---

Trong phần này, chúng ta sẽ tạo các hàm AWS Lambda để tự động thu thập dữ liệu thời tiết từ OpenWeatherMap API và lưu trữ vào S3. Những hàm này sẽ là lõi của hệ thống thu thập dữ liệu thời tiết.

## Bước 1: Tạo IAM Role cho Lambda

### 1.1 Tạo Lambda Execution Role

1. **Điều hướng đến IAM Console**

   - AWS Console → IAM → Roles
   - Click "Create role"

2. **Chọn Trusted Entity**

   - **Service**: Lambda
   - Click "Next"
     ![Trusted Entity](/images/data-collection/22b1.png)

3. **Cấu hình Role**
   - **Role Name**: `WeatherCollectorLambdaRole`
   - **Description**: `Execution role for weather data collection Lambda functions`

### 1.2 Gắn AWS Managed Policies

**AWS Managed Policy là gì?**

AWS đã tạo sẵn nhiều policies phổ biến mà chúng ta có thể "gắn" (attach) vào Role thay vì tự viết từ đầu. Điều này tiết kiệm thời gian và đảm bảo bảo mật.

**Cách gắn các policies trong AWS Console:**

1. **Khi tạo Role** `WeatherCollectorLambdaRole`, ở bước "Add permissions"
2. **Tab "Permissions"** → Click "Attach policies directly"
3. **Tìm kiếm và chọn từng policy sau:**
   - ✅ Gõ `AWSLambdaBasicExecutionRole` → tick chọn
   - ✅ Gõ `AmazonS3FullAccess` → tick chọn
   - ✅ Gõ `AmazonSSMReadOnlyAccess` → tick chọn
   - ✅ Gõ `CloudWatchAgentServerPolicy` → tick chọn
4. **Click "Add Permissions"** và hoàn thành tạo role

![Managed Policies](/images/data-collection/22b11.png)

**Kết quả:**
Role `WeatherCollectorLambdaRole` sẽ có **4 managed policies**

**Policy này cho phép Lambda:**

- **Tạo CloudWatch Log Group** để ghi logs
- **Ghi logs vào CloudWatch** khi function chạy
- **Basic networking** để Lambda hoạt động

## Bước 2: Tạo S3 Bucket cho Dữ liệu Thời tiết

### 2.1 Tạo Weather Data Bucket

1. **Điều hướng đến S3 Console**

   - AWS Console → S3 → Create bucket

![S3](/images/data-collection/22b21.png)

2. **Cấu hình Bucket**

   - **Bucket Name**: `weather-data-{your-account-id}` (thay thế bằng AWS account ID của bạn)
   - **Region**: us-east-1 (hoặc region ưa thích của bạn)
   - **Block Public Access**: Giữ tất cả cài đặt được bật (khuyến nghị)

![res-S3](/images/data-collection/22b22.png)

3. **Cấu trúc Bucket**
   ```
   weather-data-123456789012/
   ├── raw/
       ├── current-weather/
       │   └── year=2025/month=01/day=03/hour=10/
                 └── year=2025/month=01/day=03/
   ```

## Bước 3: Lambda Function cho Current Weather

### 3.1 Tạo Current Weather Function

1. **Điều hướng đến Lambda Console**

   - AWS Console → Lambda → Create function
     ![Lambda Console](/images/data-collection/22b31.png)

2. **Cấu hình Function**
   - **Function Name**: `weather-current-collector`
   - **Runtime**: Python 3.11
   - **Architecture**: x86_64
   - **Execution Role**: Use existing role → `WeatherCollectorLambdaRole`

![Create Lambda](/images/data-collection/22b32.png)

### 3.2 Thêm Function Code

**Quan trọng**: Trước khi copy code, bạn cần **thay đổi API key**.
Ở dòng `OPENWEATHER_API_KEY = 'API Key`  
→ Thay bằng API key của bạn từ OpenWeatherMap

1. **Trong Lambda Console** → Scroll xuống **Code source**
2. **Xóa toàn bộ code mặc định** trong file `lambda_function.py`
3. **Copy và paste code sau:**

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

def lambda_handler(event, context):
    """Main handler for Lambda function."""
    logger.info(f"Starting weather data collection: {json.dumps(event)}")

    # Basic validation
    if not BUCKET_NAME:
        logger.error("WEATHER_BUCKET_NAME environment variable is required")
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': 'Configuration error',
                'message': 'WEATHER_BUCKET_NAME environment variable not set'
            })
        }

    successful_collections = 0
    failed_collections = 0
    results = []

    try:
        # API key validation
        if not OPENWEATHER_API_KEY or OPENWEATHER_API_KEY == 'your_api_key_here':
            logger.error("OpenWeatherMap API key is not configured")
            return {
                'statusCode': 500,
                'body': json.dumps({
                    'error': 'Configuration error',
                    'message': 'OpenWeatherMap API key not set'
                })
            }

        # Collect data for each city
        for city in CITIES:
            logger.info(f"Collecting data for {city['name']}")

            # Fetch weather data
            weather_data = fetch_current_weather(city, OPENWEATHER_API_KEY)

            if weather_data:
                # Save to S3
                if save_to_s3(weather_data, city['name']):
                    successful_collections += 1
                    results.append({
                        'city': city['name'],
                        'status': 'success',
                        'temperature': weather_data.get('main', {}).get('temp'),
                        'description': weather_data.get('weather', [{}])[0].get('description')
                    })
                else:
                    failed_collections += 1
                    results.append({
                        'city': city['name'],
                        'status': 'failed',
                        'error': 'S3 save failed'
                    })
            else:
                failed_collections += 1
                results.append({
                    'city': city['name'],
                    'status': 'failed',
                    'error': 'API fetch failed'
                })

        # Send metrics (batched to save cost)
        if successful_collections > 0 or failed_collections > 0:
            try:
                cloudwatch.put_metric_data(
                    Namespace='Weather/ETL',
                    MetricData=[
                        {
                            'MetricName': 'SuccessfulCollections',
                            'Value': successful_collections,
                            'Unit': 'Count',
                            'Timestamp': datetime.now(timezone.utc)
                        },
                        {
                            'MetricName': 'FailedCollections',
                            'Value': failed_collections,
                            'Unit': 'Count',
                            'Timestamp': datetime.now(timezone.utc)
                        },
                        {
                            'MetricName': 'TotalCollections',
                            'Value': successful_collections + failed_collections,
                            'Unit': 'Count',
                            'Timestamp': datetime.now(timezone.utc)
                        }
                    ]
                )
            except Exception as e:
                logger.error(f"Metrics send error: {e}")

        # Log results
        logger.info(f"Collection completed - Success: {successful_collections}, Failed: {failed_collections}")

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Weather data collection completed',
                'successful_collections': successful_collections,
                'failed_collections': failed_collections,
                'results': results
            })
        }

    except Exception as e:
        logger.error(f"Lambda handler error: {e}")
        send_metrics('LambdaErrors', 1)

        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': 'Internal error',
                'message': str(e)
            })
        }
```

Giải thích code

1. **Import libraries**: Các thư viện cần thiết cho HTTP requests, AWS services, logging
2. **Cấu hình API key**: Lưu OpenWeatherMap API key (nhớ thay bằng key của bạn!)
3. **Danh sách thành phố**: 6 thành phố Việt Nam với tọa độ GPS chính xác
4. **`fetch_current_weather()`**: Gọi API để lấy thời tiết hiện tại
5. **`save_to_s3()`**: Lưu dữ liệu JSON vào S3 với cấu trúc partition thông minh
6. **`send_metrics()`**: Gửi metrics tới CloudWatch để monitoring
7. **`lambda_handler()`**: Hàm chính - xử lý từng thành phố và trả về kết quả

**Luồng hoạt động:**
`Lambda được trigger` → `Kiểm tra cấu hình` → `Lặp qua 6 thành phố` → `Gọi API` → `Lưu vào S3` → `Gửi metrics` → `Trả về kết quả`

### 3.3 Cấu hình Environment Variables

**Bước quan trọng**: Lambda cần biết tên S3 bucket để lưu dữ liệu.

1. **Trong Lambda Console** → **Configuration** tab → **Environment variables**
   ![Environment Variables](/images/data-collection/22b33.png)
2. **Click "Edit"** → **Add environment variable**:
   - **Key**: `WEATHER_BUCKET_NAME`
   - **Value**: `weather-data-{your-account-id}` (thay {your-account-id} bằng account ID thật)

![Environment Variables](/images/data-collection/22b34.png)

{{% notice tip %}}
Account ID có thể tìm ở góc phải trên AWS Console (dãy số 12 chữ số)
{{% /notice %}}

### 3.4 Test Lambda Function

**Bây giờ hãy test xem function hoạt động không!**

1. **Click "Deploy"** để lưu code và cấu hình
2. **Tạo test event**:
   - **Test** tab → **Create new event**
   - **Event name**: `manual-test`
   - **Event JSON**: `{}` (để trống vì không cần input)

![Test Event](/images/data-collection/22b35.png)

3. **Click "Test"** và đợi kết quả

**Kết quả mong đợi:**

```json
{
  "statusCode": 200,
  "body": {
    "message": "Weather data collection completed",
    "successful_collections": 6,
    "failed_collections": 0,
    "results": [
      {
        "city": "HoChiMinh",
        "status": "success",
        "temperature": 28.5,
        "description": "broken clouds"
      }
      // ... 5 thành phố khác
    ]
  }
}
```

![S3 Results](/images/data-collection/22b36.png) 4. **Kiểm tra S3**: Vào S3 bucket và xem có file JSON mới không

![S3 Results](/images/data-collection/22b37.png)

Nếu test thành công, bạn đã có:

- Lambda function hoạt động
- Dữ liệu thời tiết được lưu vào S3
- CloudWatch metrics được gửi
- Error handling hoạt động

**Lambda đã sẵn sàng để tự động chạy theo schedule**

**Nếu test thất bại**, kiểm tra:

1. **CloudWatch Logs**: Lambda → Monitoring → View logs → Xem lỗi cụ thể
2. **API Key**: Đảm bảo key OpenWeatherMap đúng
3. **Environment variable**: WEATHER_BUCKET_NAME đúng format
4. **IAM Role**: Đảm bảo role có đủ permissions
5. **S3 Bucket**: Bucket đã được tạo chưa

### 4.1 Test Lambda Function

**Bây giờ hãy test xem function hoạt động không!**

1. **Click "Deploy"** để lưu code và cấu hình
2. **Tạo test event**:
   - **Test** tab → **Create new event**
   - **Event name**: `manual-test`
   - **Event JSON**: `{}` (để trống vì không cần input)

![Test Event](/images/data-collection/22b35.png)

3. **Click "Test"** và đợi kết quả

**Kết quả mong đợi:**

```json
{
  "statusCode": 200,
  "body": {
    "message": "Weather data collection completed",
    "successful_collections": 6,
    "failed_collections": 0,
    "results": [
      {
        "city": "HoChiMinh",
        "status": "success",
        "temperature": 28.5,
        "description": "broken clouds"
      }
      // ... 5 thành phố khác
    ]
  }
}
```

![S3 Results](/images/data-collection/22b36.png) 4. **Kiểm tra S3**: Vào S3 bucket và xem có file JSON mới không

![S3 Results](/images/data-collection/22b37.png)

Nếu test thành công, bạn đã có:

- Lambda function hoạt động
- Dữ liệu thời tiết được lưu vào S3
- CloudWatch metrics được gửi
- Error handling hoạt động

**Lambda đã sẵn sàng để tự động chạy theo schedule**

**Nếu test thất bại**, kiểm tra:

1. **CloudWatch Logs**: Lambda → Monitoring → View logs → Xem lỗi cụ thể
2. **API Key**: Đảm bảo key OpenWeatherMap đúng
3. **Environment variable**: WEATHER_BUCKET_NAME đúng format
4. **IAM Role**: Đảm bảo role có đủ permissions
5. **S3 Bucket**: Bucket đã được tạo chưa

## Tóm tắt

Trong phần này, chúng ta đã:

- ✅ Tạo IAM roles và policies cho Lambda function
- ✅ Thiết lập S3 bucket với cấu trúc partition thông minh
- ✅ Xây dựng Lambda function thu thập thời tiết hiện tại
- ✅ Cấu hình error handling và CloudWatch metrics
- ✅ Test function thành công với 6 thành phố Việt Nam

**Kết quả đạt được:**

- Lambda function `weather-current-collector` hoạt động ổn định
- Dữ liệu thời tiết được lưu vào S3 với format analytics-ready
- Monitoring và metrics đầy đủ qua CloudWatch
- Error handling robust với retry mechanism

**Tiếp theo**: Trong module 2.3, chúng ta sẽ thiết lập automated scheduling với EventBridge để chạy function này theo lịch trình tự động.
