---
title: "Phân tích Dữ liệu với Athena"
date: 2025-01-07T09:00:00+00:00
weight: 40
chapter: false
pre: "<b>4. </b>"
---

Trong phần này, ta sẽ sử dụng Amazon Athena để chạy các truy vấn SQL trực tiếp trên dữ liệu thời tiết đã xử lý được lưu trữ trong S3. Athena là dịch vụ truy vấn serverless giúp dễ dàng phân tích dữ liệu bằng SQL tiêu chuẩn mà không cần thiết lập cơ sở hạ tầng kho dữ liệu phức tạp.

## Mục tiêu:

- Thiết lập Amazon Athena để query dữ liệu từ S3
- Tạo database và external table cho dữ liệu thời tiết
- Viết SQL queries để phân tích dữ liệu thời tiết
- Sử dụng các trường đã enriched như `comfort_level`, `weather_severity`, `wind_condition`
- Tạo reports và insights từ dữ liệu thời tiết
- Tối ưu hóa performance và cost cho Athena queries

## Điều kiện tiên quyết

- **Hoàn thành Module 3**: Xử lý và Chuyển đổi Dữ liệu
- **Có dữ liệu thời tiết đã xử lý** trong S3 với format JSON đã cung cấp
- **Truy cập AWS Console** với quyền Administrator hoặc quyền cho Athena, S3
- **Hiểu biết cơ bản về SQL** (SELECT, WHERE, GROUP BY, etc.)


## Bước 1: Thiết lập S3 Bucket cho Athena Query Results

Athena cần một S3 bucket để lưu trữ kết quả truy vấn. Chúng ta sẽ tạo bucket này thủ công qua AWS Console.

### 1.1 Tạo S3 Bucket cho Query Results

**Bước 1: Truy cập S3 Console**

1. Đăng nhập vào **AWS Console**
2. Tìm kiếm và chọn **S3** service
3. Nhấp **Create bucket**

![S3 Console](/images/data-storage-solutions/4b1.png)

**Bước 2: Cấu hình Bucket**

1. **Bucket name**: Nhập tên unique, ví dụ: `weather-athena-query-results-[your-account-id]`

2. **Region**: Chọn cùng region với bucket dữ liệu weather của bạn 

3. **Object Ownership**: Để mặc định **ACLs disabled**

4. **Block Public Access**: Để mặc định (block all public access) 

![S3 Bucket Configuration](/images/data-storage-solutions/4b2.png)

**Bước 3: Advanced Settings**

1. **Bucket Versioning**: Disable (không cần thiết cho query results)
2. **Default encryption**: Enable với **Amazon S3 managed keys (SSE-S3)**
3. Nhấp **Create bucket**

![S3 Bucket Advanced](/images/data-storage-solutions/4b3.png)

### 1.2 Thiết lập Lifecycle Policy (Tùy chọn)

Để tự động xóa query results cũ và tiết kiệm chi phí:

**Bước 1: Vào Bucket Management**

1. Nhấp vào bucket vừa tạo
2. Chọn tab **Management**
3. Nhấp **Create lifecycle rule**

**Bước 2: Cấu hình Lifecycle Rule**

1. **Rule name**: `delete-old-query-results`
2. **Status**: Enable
3. **Rule scope**: Apply to all objects in the bucket
4. **Lifecycle rule actions**: Expire current versions of objects
5. **Expire current versions of objects**: 30 days
6. Nhấp **Create rule**

{{% notice info %}}
Lifecycle policy này sẽ tự động xóa query results sau 30 ngày để tiết kiệm chi phí storage.
{{% /notice %}}

![S3 Lifecycle Policy](/images/data-storage-solutions/4b4.png)

### 1.3 Cấu hình Athena Query Result Location

**Bước 1: Truy cập Athena Console**

1. Tìm kiếm và chọn **Amazon Athena** service

![Athena console](/images/data-storage-solutions/4b5.png)

**Bước 2: Set up Query Result Location**
1. Chọn **Query editor** ở menu bên phải
1. Nhấp **Settings** ở góc phải trên cùng
2. Nhấp **Manage**

![Athena Settings](/images/data-storage-solutions/4b6.png)

**Bước 3: Configure Location**

1. **Query result location**: Chọn Browse S3 đang có sẵn là bucket result vừa tạo ở trên

   {{% notice warning %}}
   Nhớ thêm `/` ở cuối đường dẫn S3!
   {{% /notice %}}

2. **Expected bucket owner**: Để trống
3. **Encrypt query results**: Check và chọn **SSE-S3**
4. Nhấp **Save**

![Athena Settings](/images/data-storage-solutions/4b7.png)

## Bước 2: Tạo Database và External Table

### 2.1 Tạo Weather Analytics Database

**Bước 1: Mở Query Editor**

1. Trong Athena Console, chọn **Query editor** từ menu bên trái
2. Đảm bảo bạn đang ở **Data source**: AwsDataCatalog
3. **Database**: default

**Bước 2: Tạo Database**

Copy và chạy query sau trong Athena Query Editor:

```sql
CREATE DATABASE IF NOT EXISTS weather_analytics
LOCATION 's3://weather-athena-query-results-[your-account-id]/databases/weather_analytics/';
```

![Athena Settings](/images/data-storage-solutions/4b8.png)

**Bước 3: Chọn Database**

1. Sau khi query chạy thành công, refresh page
2. Trong dropdown **Database**, chọn `weather_analytics`

![Athena Settings](/images/data-storage-solutions/4b9.png)

### 2.2 Tạo External Table cho Weather Data

**Bước 1: Chuẩn bị Schema**

Dựa trên cấu trúc JSON đã xử lý trước đó, chúng ta sẽ tạo external table:

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

- Schema này match chính xác với JSON structure từ module 3
- Bao gồm các trường mới: `comfort_level`, `wind_condition`, `weather_severity`
- Sử dụng JsonSerDe để parse JSON files
- Có partition projection cho performance tốt hơn
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

## Bước 3: Phân tích Dữ liệu Cơ bản

### 3.1 Data Quality Check

**Kiểm tra chất lượng dữ liệu:**

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

### 3.2 City-wise Temperature Analysis

**Phân tích nhiệt độ theo thành phố:**

```sql
WITH base_data AS (
  SELECT
    CASE
      WHEN city_name IN ('HoChiMinh', 'Ho Chi Minh City', 'TPHCM') THEN 'Ho Chi Minh'
      WHEN city_name IN ('CanTho', 'Can Tho') THEN 'Can Tho'
      WHEN city_name IN ('GiaLai', 'Pleiku') THEN 'Gia Lai'
      WHEN city_name IN ('Da Nang', 'Danang', 'Ap Ba') THEN 'Da Nang'
      WHEN city_name IN ('Hanoi', 'Ha Noi', 'Xom Pho') THEN 'Ha Noi'
      WHEN city_name IN ('Hue') THEN 'Hue'
      ELSE city_name
    END AS standardized_city,
    country,
    comfort_level,
    temperature_celsius,
    heat_index_celsius
  FROM weather_analytics.current_weather
),

comfort_counts AS (
  SELECT
    standardized_city,
    country,
    comfort_level,
    COUNT(*) AS level_count,
    ROW_NUMBER() OVER (
      PARTITION BY standardized_city, country
      ORDER BY COUNT(*) DESC
    ) AS rn
  FROM base_data
  GROUP BY standardized_city, country, comfort_level
)

SELECT
  t.standardized_city AS city_name,
  t.country,
  COUNT(*) AS measurement_count,
  ROUND(AVG(t.temperature_celsius), 2) AS avg_temp_celsius,
  ROUND(MIN(t.temperature_celsius), 2) AS min_temp_celsius,
  ROUND(MAX(t.temperature_celsius), 2) AS max_temp_celsius,
  ROUND(AVG(t.heat_index_celsius), 2) AS avg_heat_index,
  c.comfort_level AS most_common_comfort_level
FROM base_data t
LEFT JOIN comfort_counts c
  ON t.standardized_city = c.standardized_city
 AND t.country = c.country
 AND c.rn = 1
GROUP BY t.standardized_city, t.country, c.comfort_level
ORDER BY avg_temp_celsius DESC;
```

![Athena Query](/images/data-storage-solutions/4b14.png)

### 3.3 Comfort Level Distribution

**Phân tích comfort level:**

```sql
SELECT
  comfort_level,
  COUNT(*) as occurrence_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as percentage,
  ROUND(AVG(temperature_celsius), 2) as avg_temp,
  ROUND(AVG(humidity_percent), 2) as avg_humidity,
  ROUND(AVG(heat_index_celsius), 2) as avg_heat_index
FROM weather_analytics.current_weather
WHERE comfort_level IS NOT NULL
GROUP BY comfort_level
ORDER BY occurrence_count DESC;
```

![Athena Query](/images/data-storage-solutions/4b15.png)

### 3.4 Weather Severity Analysis

**Phân tích mức độ nghiêm trọng thời tiết:**

```sql
SELECT
  weather_severity,
  COUNT(*) as total_occurrences,
  COUNT(DISTINCT city_name) as cities_affected,
  ROUND(AVG(temperature_celsius), 2) as avg_temperature,
  ROUND(AVG(wind_speed_kmh), 2) as avg_wind_speed,
  ROUND(AVG(humidity_percent), 2) as avg_humidity,
  ROUND(AVG(pressure_hpa), 2) as avg_pressure
FROM weather_analytics.current_weather
WHERE weather_severity IS NOT NULL
GROUP BY weather_severity
ORDER BY
  CASE weather_severity
    WHEN 'severe' THEN 1
    WHEN 'moderate' THEN 2
    WHEN 'light' THEN 3
    ELSE 4
  END;
```

![Athena Query](/images/data-storage-solutions/4b16.png)

## Bước 4: Advanced Analytics

### 4.1 Daily Weather Summary

**Tạo báo cáo thời tiết hàng ngày:**

```sql
WITH mode_counts AS (
  SELECT
    data_collection_date,
    city_name,
    weather_main,
    comfort_level,
    wind_condition,
    weather_severity,
    COUNT(*) AS freq,
    ROW_NUMBER() OVER (PARTITION BY data_collection_date, city_name ORDER BY COUNT(*) DESC) AS rn_weather,
    ROW_NUMBER() OVER (PARTITION BY data_collection_date, city_name, comfort_level ORDER BY COUNT(*) DESC) AS rn_comfort,
    ROW_NUMBER() OVER (PARTITION BY data_collection_date, city_name, wind_condition ORDER BY COUNT(*) DESC) AS rn_wind,
    ROW_NUMBER() OVER (PARTITION BY data_collection_date, city_name, weather_severity ORDER BY COUNT(*) DESC) AS rn_severity
  FROM weather_analytics.current_weather
  WHERE DATE(data_collection_date) >= DATE('2025-07-01')

  GROUP BY data_collection_date, city_name, weather_main, comfort_level, wind_condition, weather_severity
)

SELECT
  stats.data_collection_date,
  stats.city_name,
  stats.daily_measurements,
  stats.avg_temp_c,
  stats.avg_humidity,
  stats.avg_pressure,
  stats.avg_wind_kmh,
  m.weather_main AS dominant_weather,
  m.comfort_level AS dominant_comfort,
  m.wind_condition AS dominant_wind_condition,
  m.weather_severity AS dominant_severity
FROM (
  SELECT
    data_collection_date,
    city_name,
    COUNT(*) AS daily_measurements,
    ROUND(AVG(temperature_celsius), 1) AS avg_temp_c,
    ROUND(AVG(humidity_percent), 1) AS avg_humidity,
    ROUND(AVG(pressure_hpa), 1) AS avg_pressure,
    ROUND(AVG(wind_speed_kmh), 1) AS avg_wind_kmh
  FROM weather_analytics.current_weather
  WHERE DATE(data_collection_date) >= DATE('2025-07-01')
  GROUP BY data_collection_date, city_name
) stats
LEFT JOIN mode_counts m
  ON stats.data_collection_date = m.data_collection_date
 AND stats.city_name = m.city_name
 AND m.rn_weather = 1
 AND m.rn_comfort = 1
 AND m.rn_wind = 1
 AND m.rn_severity = 1
ORDER BY stats.data_collection_date DESC, stats.city_name;

```

![Athena Query](/images/data-storage-solutions/4b17.png)

### 4.2 Weather Pattern Correlation

**Phân tích mối tương quan giữa các yếu tố thời tiết:**

```sql
SELECT
  comfort_level,
  weather_main,
  weather_severity,
  COUNT(*) as occurrence_count,
  ROUND(AVG(temperature_celsius), 2) as avg_temperature,
  ROUND(AVG(humidity_percent), 2) as avg_humidity,
  ROUND(AVG(heat_index_celsius), 2) as avg_heat_index,
  ROUND(AVG(wind_speed_kmh), 2) as avg_wind_speed
FROM weather_analytics.current_weather
WHERE comfort_level IS NOT NULL
  AND weather_severity IS NOT NULL
GROUP BY comfort_level, weather_main, weather_severity
HAVING COUNT(*) >= 3  -- Chỉ hiển thị patterns với ít nhất 3 occurrences
ORDER BY comfort_level, occurrence_count DESC;
```

![Athena Query](/images/data-storage-solutions/4b18.png)

### 4.3 Wind Analysis

**Phân tích điều kiện gió:**

```sql
WITH base_data AS (
  SELECT
    CASE
      WHEN city_name IN ('HoChiMinh', 'Ho Chi Minh City', 'TPHCM') THEN 'Ho Chi Minh'
      WHEN city_name IN ('CanTho', 'Can Tho') THEN 'Can Tho'
      WHEN city_name IN ('GiaLai', 'Pleiku') THEN 'Gia Lai'
      WHEN city_name IN ('Da Nang', 'Danang', 'Ap Ba') THEN 'Da Nang'
      WHEN city_name IN ('Hanoi', 'Ha Noi', 'Xom Pho') THEN 'Ha Noi'
      WHEN city_name IN ('Hue') THEN 'Hue'
      ELSE city_name
    END AS standardized_city,
    wind_condition,
    wind_speed_kmh,
    wind_gust_ms,
    wind_direction_deg
  FROM weather_analytics.current_weather
  WHERE wind_condition IS NOT NULL
)

SELECT
  standardized_city AS city_name,
  wind_condition,
  COUNT(*) AS occurrence_count,
  ROUND(AVG(wind_speed_kmh), 2) AS avg_wind_speed_kmh,
  ROUND(MAX(wind_speed_kmh), 2) AS max_wind_speed_kmh,
  ROUND(AVG(wind_gust_ms), 2) AS avg_wind_gust_ms,
  CASE
    WHEN AVG(wind_direction_deg) BETWEEN 0 AND 45 OR AVG(wind_direction_deg) > 315 THEN 'North'
    WHEN AVG(wind_direction_deg) BETWEEN 45 AND 135 THEN 'East'
    WHEN AVG(wind_direction_deg) BETWEEN 135 AND 225 THEN 'South'
    WHEN AVG(wind_direction_deg) BETWEEN 225 AND 315 THEN 'West'
    ELSE 'Variable'
  END AS dominant_wind_direction
FROM base_data
GROUP BY standardized_city, wind_condition
ORDER BY city_name, avg_wind_speed_kmh DESC;
```

![Athena Query](/images/data-storage-solutions/4b19.png)

## Bước 5: Tạo Views cho Reporting

### 5.1 Daily Weather Summary View

**Tạo view cho báo cáo hàng ngày:**

```sql
CREATE OR REPLACE VIEW weather_analytics.daily_weather_summary AS
SELECT
  data_collection_date,
  city_name,
  country,
  COUNT(*) as measurement_count,
  ROUND(AVG(temperature_celsius), 2) as avg_temperature,
  ROUND(AVG(feels_like_celsius), 2) as avg_feels_like,
  ROUND(AVG(heat_index_celsius), 2) as avg_heat_index,
  ROUND(AVG(humidity_percent), 2) as avg_humidity,
  ROUND(AVG(pressure_hpa), 2) as avg_pressure,
  ROUND(AVG(wind_speed_kmh), 2) as avg_wind_speed,
  MODE() WITHIN GROUP (ORDER BY weather_main) as dominant_weather,
  MODE() WITHIN GROUP (ORDER BY comfort_level) as dominant_comfort_level,
  MODE() WITHIN GROUP (ORDER BY wind_condition) as dominant_wind_condition,
  MODE() WITHIN GROUP (ORDER BY weather_severity) as dominant_weather_severity
FROM weather_analytics.current_weather
GROUP BY data_collection_date, city_name, country;
```

### 5.2 Sử dụng Views

**Query từ view:**

```sql
SELECT *
FROM weather_analytics.daily_weather_summary
WHERE data_collection_date >= DATE('2025-07-01')
ORDER BY data_collection_date DESC, city_name;
```

![Athena Query](/images/data-storage-solutions/4b20.png)

**Export results:**

1. Chạy query trong Athena
2. Nhấp **Download** để lưu kết quả CSV
3. Kết quả cũng tự động lưu trong S3 query results bucket


Kết quả:

- Phân tích dữ liệu thời tiết với SQL queries phức tạp  
- Tạo Insights từ comfort levels và weather severity  
- Tạo Report cho business applications  

- Trong phần tiếp theo, chúng ta sẽ tạo **QuickSight dashboards** từ Athena data