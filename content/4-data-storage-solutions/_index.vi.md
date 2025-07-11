---
title: "Phân tích Dữ liệu với Athena"
date: 2025-01-07T09:00:00+00:00
weight: 40
chapter: false
pre: "<b>4. </b>"
---

# Phân tích Dữ liệu với Athena

Trong module này, bạn sẽ học cách sử dụng Amazon Athena để chạy các truy vấn SQL trực tiếp trên dữ liệu thời tiết đã xử lý được lưu trữ trong S3. Athena là dịch vụ truy vấn serverless giúp dễ dàng phân tích dữ liệu bằng SQL tiêu chuẩn mà không cần thiết lập cơ sở hạ tầng kho dữ liệu phức tạp.

## Tổng quan Module

Amazon Athena cho phép bạn phân tích dữ liệu trong S3 bằng các truy vấn SQL tiêu chuẩn mà không cần phải di chuyển dữ liệu hoặc thiết lập máy chủ. Điều này làm cho nó trở thành công cụ lý tưởng để phân tích dữ liệu thời tiết của bạn. Trong module này, chúng ta sẽ thiết lập Athena để truy vấn dữ liệu thời tiết đã xử lý và trích xuất những hiểu biết có ý nghĩa.

**Thời gian:** 40-50 phút  
**Chi phí:** ~$0.50

## Những gì bạn sẽ xây dựng

```mermaid
graph LR
    A[S3 Processed Data] --> B[Athena Query Service]
    B --> C[SQL Analysis]
    C --> D[Weather Insights]

    style B fill:#4fc3f7,stroke:#232f3e,stroke-width:3px
    style A fill:#f3e5f5
    style D fill:#66bb6a
```

## Điều kiện tiên quyết

- Đã hoàn thành Module 3: Xử lý và Chuyển đổi Dữ liệu
- Dữ liệu thời tiết đã xử lý có sẵn trong S3
- Truy cập AWS Console

## Bước 1: Thiết lập Vị trí Kết quả Truy vấn Athena

Trước khi sử dụng Athena, bạn cần cấu hình vị trí để lưu trữ kết quả truy vấn.

### 1.1 Tạo S3 Bucket cho Kết quả Truy vấn

```bash
# Tạo bucket cho kết quả truy vấn Athena
aws s3 mb s3://your-athena-query-results-bucket

# Tùy chọn: Thiết lập lifecycle policy để xóa kết quả cũ sau 30 ngày
cat > lifecycle-policy.json << EOF
{
  "Rules": [
    {
      "ID": "DeleteOldQueryResults",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Expiration": {"Days": 30}
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket your-athena-query-results-bucket \
  --lifecycle-configuration file://lifecycle-policy.json
```

### 1.2 Cấu hình Vị trí Kết quả Truy vấn Athena

1. Mở **Amazon Athena console**
2. Nhấp **Settings** ở góc phải trên
3. Nhấp **Manage**
4. Nhập đường dẫn S3 bucket: `s3://your-athena-query-results-bucket/`
5. Nhấp **Save**

## Bước 2: Tạo Database và Table

### 2.1 Tạo Weather Analytics Database

Trong Athena Query Editor, chạy:

```sql
CREATE DATABASE IF NOT EXISTS weather_analytics
COMMENT 'Database for weather data analysis'
LOCATION 's3://your-athena-query-results-bucket/databases/weather_analytics/';
```

### 2.2 Tạo External Table cho Dữ liệu Thời tiết Đã Xử lý

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS weather_analytics.current_weather (
  timestamp STRING,
  city_name STRING,
  country STRING,
  latitude DOUBLE,
  longitude DOUBLE,
  data_collection_date STRING,
  temperature_kelvin DOUBLE,
  temperature_celsius DOUBLE,
  temperature_fahrenheit DOUBLE,
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
  wind_speed_kmh DOUBLE,
  wind_speed_mph DOUBLE,
  wind_direction_deg INT,
  wind_gust_ms DOUBLE,
  cloud_coverage_percent INT,
  rain_1h_mm DOUBLE,
  rain_3h_mm DOUBLE,
  snow_1h_mm DOUBLE,
  snow_3h_mm DOUBLE,
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

### 2.3 Xác minh Tạo Table

```sql
-- Kiểm tra xem table đã được tạo thành công chưa
SHOW TABLES IN weather_analytics;

-- Lấy schema của table
DESCRIBE weather_analytics.current_weather;

-- Đếm tổng số bản ghi
SELECT COUNT(*) as total_records
FROM weather_analytics.current_weather;
```

## Bước 3: Phân tích Dữ liệu Thời tiết Cơ bản

### 3.1 Truy vấn Khám phá Dữ liệu

#### Xem Dữ liệu Mẫu

```sql
-- Lấy mẫu dữ liệu của bạn
SELECT *
FROM weather_analytics.current_weather
LIMIT 10;
```

#### Kiểm tra Chất lượng Dữ liệu

```sql
-- Kiểm tra giá trị thiếu trong các trường chính
SELECT
  COUNT(*) as total_records,
  COUNT(temperature_celsius) as temp_records,
  COUNT(humidity_percent) as humidity_records,
  COUNT(pressure_hpa) as pressure_records,
  COUNT(wind_speed_ms) as wind_records
FROM weather_analytics.current_weather;
```

#### Phân tích Phạm vi Ngày

```sql
-- Kiểm tra phạm vi ngày của dữ liệu
SELECT
  MIN(data_collection_date) as earliest_date,
  MAX(data_collection_date) as latest_date,
  COUNT(DISTINCT data_collection_date) as unique_dates
FROM weather_analytics.current_weather;
```

### 3.2 Phân tích Nhiệt độ

#### Nhiệt độ Trung bình theo Thành phố

```sql
SELECT
  city_name,
  country,
  COUNT(*) as measurement_count,
  ROUND(AVG(temperature_celsius), 2) as avg_temp_celsius,
  ROUND(MIN(temperature_celsius), 2) as min_temp_celsius,
  ROUND(MAX(temperature_celsius), 2) as max_temp_celsius
FROM weather_analytics.current_weather
GROUP BY city_name, country
ORDER BY avg_temp_celsius DESC;
```

#### Xu hướng Nhiệt độ Theo Thời gian

```sql
SELECT
  data_collection_date,
  city_name,
  ROUND(AVG(temperature_celsius), 2) as avg_temp,
  ROUND(AVG(feels_like_celsius), 2) as avg_feels_like
FROM weather_analytics.current_weather
WHERE data_collection_date >= DATE('2025-01-01')
GROUP BY data_collection_date, city_name
ORDER BY data_collection_date, city_name;
```

#### Phân tích Chỉ số Nhiệt

```sql
SELECT
  city_name,
  ROUND(AVG(heat_index_celsius), 2) as avg_heat_index,
  comfort_level,
  COUNT(*) as occurrence_count
FROM weather_analytics.current_weather
WHERE heat_index_celsius IS NOT NULL
GROUP BY city_name, comfort_level
ORDER BY avg_heat_index DESC;
```

### 3.3 Phân tích Mẫu Thời tiết

#### Phân bố Điều kiện Thời tiết

```sql
SELECT
  weather_main,
  weather_description,
  COUNT(*) as occurrence_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage
FROM weather_analytics.current_weather
GROUP BY weather_main, weather_description
ORDER BY occurrence_count DESC;
```

#### Phân tích Gió

```sql
SELECT
  city_name,
  wind_condition,
  COUNT(*) as occurrence_count,
  ROUND(AVG(wind_speed_kmh), 2) as avg_wind_speed_kmh,
  ROUND(MAX(wind_speed_kmh), 2) as max_wind_speed_kmh
FROM weather_analytics.current_weather
GROUP BY city_name, wind_condition
ORDER BY city_name, avg_wind_speed_kmh DESC;
```

#### Tương quan Độ ẩm và Nhiệt độ

```sql
SELECT
  city_name,
  CASE
    WHEN temperature_celsius < 15 THEN 'Lạnh'
    WHEN temperature_celsius < 25 THEN 'Vừa'
    ELSE 'Nóng'
  END as temp_category,
  ROUND(AVG(humidity_percent), 2) as avg_humidity,
  ROUND(AVG(temperature_celsius), 2) as avg_temperature,
  COUNT(*) as sample_count
FROM weather_analytics.current_weather
GROUP BY city_name,
  CASE
    WHEN temperature_celsius < 15 THEN 'Lạnh'
    WHEN temperature_celsius < 25 THEN 'Vừa'
    ELSE 'Nóng'
  END
ORDER BY city_name, temp_category;
```

## Bước 4: Truy vấn Phân tích Nâng cao

### 4.1 Phân tích Chuỗi Thời gian

#### Tóm tắt Thời tiết Hàng ngày

```sql
SELECT
  data_collection_date,
  COUNT(DISTINCT city_name) as cities_measured,
  ROUND(AVG(temperature_celsius), 2) as global_avg_temp,
  ROUND(AVG(humidity_percent), 2) as global_avg_humidity,
  ROUND(AVG(pressure_hpa), 2) as global_avg_pressure
FROM weather_analytics.current_weather
GROUP BY data_collection_date
ORDER BY data_collection_date;
```

#### Cực trị Thời tiết

```sql
-- Tìm cực trị nhiệt độ
WITH temp_extremes AS (
  SELECT
    data_collection_date,
    city_name,
    temperature_celsius,
    ROW_NUMBER() OVER (PARTITION BY data_collection_date ORDER BY temperature_celsius DESC) as hot_rank,
    ROW_NUMBER() OVER (PARTITION BY data_collection_date ORDER BY temperature_celsius ASC) as cold_rank
  FROM weather_analytics.current_weather
)
SELECT
  data_collection_date,
  MAX(CASE WHEN hot_rank = 1 THEN city_name END) as hottest_city,
  MAX(CASE WHEN hot_rank = 1 THEN temperature_celsius END) as highest_temp,
  MAX(CASE WHEN cold_rank = 1 THEN city_name END) as coldest_city,
  MAX(CASE WHEN cold_rank = 1 THEN temperature_celsius END) as lowest_temp
FROM temp_extremes
WHERE hot_rank = 1 OR cold_rank = 1
GROUP BY data_collection_date
ORDER BY data_collection_date;
```

### 4.2 Phân tích Địa lý

#### Thời tiết theo Vùng Địa lý

```sql
SELECT
  CASE
    WHEN latitude > 23.5 THEN 'Ôn đới Bắc'
    WHEN latitude > 0 THEN 'Nhiệt đới Bắc'
    WHEN latitude > -23.5 THEN 'Nhiệt đới Nam'
    ELSE 'Ôn đới Nam'
  END as climate_zone,
  COUNT(*) as measurement_count,
  ROUND(AVG(temperature_celsius), 2) as avg_temperature,
  ROUND(AVG(humidity_percent), 2) as avg_humidity,
  ROUND(AVG(wind_speed_kmh), 2) as avg_wind_speed
FROM weather_analytics.current_weather
GROUP BY
  CASE
    WHEN latitude > 23.5 THEN 'Ôn đới Bắc'
    WHEN latitude > 0 THEN 'Nhiệt đới Bắc'
    WHEN latitude > -23.5 THEN 'Nhiệt đới Nam'
    ELSE 'Ôn đới Nam'
  END
ORDER BY avg_temperature DESC;
```

## Bước 5: Xuất và Trực quan hóa Kết quả

### 5.1 Lưu Kết quả Truy vấn

1. Chạy bất kỳ truy vấn nào trong Athena
2. Nhấp **Download** để lưu kết quả dưới dạng CSV
3. Kết quả cũng được tự động lưu vào S3 query results bucket của bạn

### 5.2 Tạo Trực quan hóa Đơn giản

Bạn có thể sử dụng các file CSV đã xuất để tạo biểu đồ trong:

- **Excel hoặc Google Sheets** cho biểu đồ cơ bản
- **Python/Jupyter** cho phân tích nâng cao
- **QuickSight** (được đề cập trong module tiếp theo)

### 5.3 Ví dụ Quy trình Phân tích

```sql
-- Tạo báo cáo thời tiết toàn diện
SELECT
  city_name,
  country,
  COUNT(*) as total_measurements,
  ROUND(AVG(temperature_celsius), 1) as avg_temp_c,
  ROUND(MIN(temperature_celsius), 1) as min_temp_c,
  ROUND(MAX(temperature_celsius), 1) as max_temp_c,
  ROUND(AVG(humidity_percent), 1) as avg_humidity,
  ROUND(AVG(pressure_hpa), 1) as avg_pressure,
  ROUND(AVG(wind_speed_kmh), 1) as avg_wind_kmh,
  MODE() WITHIN GROUP (ORDER BY weather_main) as most_common_weather,
  MODE() WITHIN GROUP (ORDER BY comfort_level) as most_common_comfort
FROM weather_analytics.current_weather
GROUP BY city_name, country
HAVING COUNT(*) >= 5  -- Chỉ các thành phố có ít nhất 5 phép đo
ORDER BY avg_temp_c DESC;
```

## Bước 6: Tối ưu hóa Hiệu suất

### 6.1 Mẹo Hiệu suất Truy vấn

1. **Sử dụng LIMIT** cho truy vấn khám phá:

```sql
SELECT * FROM weather_analytics.current_weather
WHERE data_collection_date = '2025-01-15'
LIMIT 100;
```

2. **Lọc trên cột partition**:

```sql
-- Tốt - lọc trên cột đã partition
SELECT * FROM weather_analytics.current_weather
WHERE data_collection_date BETWEEN '2025-01-01' AND '2025-01-07';
```

3. **Chỉ select cột cần thiết**:

```sql
-- Hiệu suất tốt hơn
SELECT city_name, temperature_celsius, humidity_percent
FROM weather_analytics.current_weather;
```

### 6.2 Tối ưu hóa Chi phí

Giám sát việc sử dụng Athena của bạn:

1. Kiểm tra **CloudWatch metrics** cho dữ liệu được quét
2. Sử dụng **Athena Query History** để phân tích chi phí
3. Xem xét chuyển đổi sang **định dạng Parquet** cho dataset lớn

## Khắc phục Các Vấn đề Thường gặp

### Vấn đề 1: Lỗi "Table not found"

- Xác minh quyền S3 bucket của bạn
- Kiểm tra dữ liệu tồn tại trong vị trí S3 đã chỉ định
- Đảm bảo schema table phù hợp với dữ liệu của bạn

### Vấn đề 2: "Zero records returned"

- Xác minh định dạng dữ liệu phù hợp với schema table
- Kiểm tra đường dẫn S3 trong định nghĩa table
- Đảm bảo files ở định dạng JSON như mong đợi

### Vấn đề 3: Lỗi "Access denied"

- Kiểm tra quyền IAM cho Athena và S3
- Xác minh quyền query results bucket

## Phân tích Chi phí

Chi phí điển hình cho module này:

- **Truy vấn Athena**: ~$5 per TB dữ liệu được quét
- **Lưu trữ S3**: ~$0.023 per GB/tháng
- **Yêu cầu S3**: ~$0.0004 per 1,000 yêu cầu

Đối với dataset thời tiết điển hình (1-10 MB), mong đợi tổng chi phí dưới $0.50.

## Các bước tiếp theo

Sau khi hoàn thành module này, bạn sẽ có thể phân tích dữ liệu thời tiết của mình bằng các truy vấn SQL trong Athena. Trong module tiếp theo, chúng ta sẽ xây dựng trên nền tảng này để tạo bảng điều khiển tương tác bằng Amazon QuickSight.

{{% notice tip %}}
Khi viết truy vấn Athena, luôn cố gắng giới hạn lượng dữ liệu được quét bằng cách sử dụng mệnh đề WHERE thích hợp trên các trường ngày tháng và các cột được lọc thường xuyên khác.
{{% /notice %}}

{{% notice info %}}
Lưu các truy vấn hữu ích của bạn dưới dạng **Saved Queries** trong Athena để tái sử dụng. Bạn cũng có thể tạo **Views** cho các truy vấn phức tạp thường được sử dụng.
{{% /notice %}}

{{% notice warning %}}
Nhớ thay thế tên bucket placeholder bằng tên S3 bucket thực tế của bạn trong tất cả truy vấn SQL.
{{% /notice %}}
