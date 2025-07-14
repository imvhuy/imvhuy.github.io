---
title: "Lập lịch Tự động với EventBridge"
date: 2025-01-03T08:45:00+07:00
weight: 3
---

Trong phần này, chúng ta sẽ thiết lập lập lịch tự động cho việc thu thập dữ liệu thời tiết bằng Amazon EventBridge (trước đây gọi là CloudWatch Events). Điều này giúp các Lambda functions chạy theo lịch trình đều đặn mà không cần can thiệp thủ công.

EventBridge giống như "đồng hồ báo thức thông minh" của AWS:

- **Chạy đúng giờ**: Trigger Lambda functions theo lịch trình cụ thể
- **Chính xác**: Chạy đúng thời gian đã định (ví dụ: mỗi giờ, mỗi ngày)
- **Tự động**: Không cần can thiệp thủ công
- **Tiết kiệm**: Chỉ chạy khi cần, không tốn tài nguyên khi không hoạt động

**Tại sao cần Scheduling?**

- Thu thập dữ liệu đều đặn 24/7
- Dữ liệu luôn fresh và cập nhật
- Tự động hóa hoàn toàn
- Không phụ thuộc vào người vận hành

**Lịch trình đề xuất:**

- **Current Weather**: Mỗi giờ (24 lần/ngày) - Để theo dõi thời tiết real-time
- **Forecast**: Mỗi 6 giờ (4 lần/ngày) - Dự báo không thay đổi nhiều

## Bước 1: Thiết lập EventBridge Rule cho Current Weather

{{% notice tip %}}
**Bước này sẽ tạo lịch trình chạy Lambda mỗi giờ để thu thập thời tiết hiện tại.**

Ví dụ: Function sẽ chạy lúc 00:00, 01:00, 02:00... 23:00 hàng ngày
{{% /notice %}}

### 1.1 Truy cập EventBridge Console

1. **Vào AWS Console** → Tìm **"EventBridge"** (hoặc **"CloudWatch"** → **"Events"**)
2. **Click "EventBridge"** → **"Rules"** ở menu bên trái
3. **Chọn Region** phù hợp (ví dụ: `us-east-1`)

![EventBridge Console](/images/data-collection/23b11.png)
![EventBridge Console](/images/data-collection/23b12.png)

### 1.2 Tạo Rule cho Current Weather

1. **Click "Create rule"**

![Create Rule](/images/data-collection/23b13.png)

2. **Bước 1 - Define rule detail:**
   - **Name**: `weather-current-hourly`
   - **Description**: `Thu thập dữ liệu thời tiết hiện tại mỗi giờ cho 6 thành phố Việt Nam`
   - **Event bus**: `default`
   - **Rule type**: `Schedule`

![Rule Details](/images/data-collection/23b14.png)

3. **Click "Continue to create rule"**

### 1.3 Cấu hình Schedule Pattern

1. **Schedule pattern**: Chọn **"A schedule that runs at a regular rate, such as every 10 minutes"**

2. **Rate expression**: Chọn **"rate"** và nhập:
   ```
   1 hour
   ```

![Schedule Pattern](/images/data-collection/23b15.png)

**Các lựa chọn Schedule Pattern:**

- **Rate expression**:

  - `rate(1 hour)` = mỗi giờ
  - `rate(30 minutes)` = mỗi 30 phút
  - `rate(1 day)` = mỗi ngày

- **Cron expression** (nâng cao):
  - `cron(0 * * * ? *)` = mỗi giờ đúng phút 0
  - `cron(0 8,12,16,20 * * ? *)` = 4 lần/ngày (8h, 12h, 16h, 20h)
  - `cron(0 0 * * ? *)` = mỗi ngày lúc 00:00
![Schedule Pattern](/images/data-collection/23b16.png)
3. **Click "Next"**

### 1.4 Select Target (Lambda Function)

1. **Target types**: Chọn **"AWS service"**

2. **Select a service**: Chọn **"Lambda function"**

3. **Function**: Chọn **`weather-current-collector`** (function đã tạo ở bước trước)

![Select Target](/images/data-collection/23b17.png)

4. **Additional settings**:
   - **Configure target input**: Chọn **"Constant (JSON text)"**
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

**JSON input này sẽ:**
- Cho Lambda biết đây là scheduled event (không phải manual test)
- Giúp phân biệt current weather vs forecast collection
- Cung cấp metadata để logging và monitoring

5. **Click "Next"**

### 1.5 Configure tags và Review

1. **Tags (Optional)**:

   - **Key**: `Project` → **Value**: `WeatherETL`
   - **Key**: `Environment` → **Value**: `Production`

2. **Review tất cả cài đặt**:

   - ✅ Name: `weather-current-hourly`
   - ✅ Schedule: `rate(1 hour)`
   - ✅ Target: `weather-current-collector`
   - ✅ State: **Enabled**

3. **Click "Create rule"**

![Review Rule](/images/data-collection/23b19.png)

Rule cho current weather đã được tạo thành công.

Rule sẽ trigger Lambda function `weather-current-collector` mỗi giờ để thu thập dữ liệu thời tiết hiện tại.

![Review Rule](/images/data-collection/23b20.png)

## Bước 2: Thiết lập EventBridge Rule cho Weather Forecast

**Bước này sẽ tạo lịch trình chạy Lambda mỗi 6 giờ để thu thập dự báo thời tiết.**

Ví dụ: Function sẽ chạy lúc 00:00, 06:00, 12:00, 18:00 hàng ngày


### 2.1 Tạo Rule thứ hai

1. **Trong EventBridge Console** → **"Create rule"**

2. **Bước 1 - Define rule detail:**
   - **Name**: `weather-forecast-6hourly`
   - **Description**: `Thu thập dữ liệu dự báo thời tiết mỗi 6 giờ cho 6 thành phố Việt Nam`
   - **Event bus**: `default`
   - **Rule type**: `Schedule`

### 2.2 Cấu hình Schedule cho Forecast

1. **Schedule pattern**: Chọn **"A fine-grained schedule that runs at a specific time"**

2. **Cron expression**:
   ```
   cron(0 0,6,12,18 * * ? *)
   ```

![Forecast Schedule](/images/data-collection/23b21.png)

### 2.3 Select Target cho Forecast

1. **Target types**: **"AWS service"**
2. **Select a service**: **"Lambda function"**
3. **Function**: **`weather-forecast-collector`**

4. **Configure target input**: **"Constant (JSON text)"**

   ```json
   {
     "source": "eventbridge-schedule",
     "detail-type": "Scheduled Event",
     "detail": {
       "collection_type": "forecast",
       "scheduled_time": "every_6_hours",
       "trigger_source": "eventbridge"
     }
   }
   ```

5. **Create rule**
![Forecast Schedule](/images/data-collection/23b22.png)

## Bước 3: Thiết lập Monitoring với CloudWatch Alarms

**Tại sao cần Monitoring?**

Khi Lambda functions chạy tự động 24/7, bạn cần biết ngay khi có vấn đề:

- Lambda function bị lỗi
- Function chạy quá lâu
- Tỷ lệ thành công thấp
- Chi phí tăng bất thường

### 3.1 Tạo SNS Topic cho Email Alerts

**Trước tiên, tạo SNS Topic để nhận email thông báo khi có lỗi:**

1. **AWS Console** → **"SNS"** → **"Topics"** → **"Create topic"**

![SNS Topic](/images/data-collection/23b31.png)

2. **Topic configuration:**

   - **Type**: `Standard`
   - **Name**: `weather-etl-alerts`
   - **Display name**: `Weather ETL Alerts`

3. **Create topic**

4. **Tạo Subscription**:
   - **Protocol**: `Email`
   - **Endpoint**: `your-email@example.com` (thay bằng email của bạn)
   - **Confirm subscription** qua email

![SNS Subscription](/images/data-collection/23b32.png)

### 3.2 Tạo CloudWatch Alarm cho Lambda Errors

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
   - **Threshold value**: `1` (alert khi có ≥1 error trong 5 phút)

![Alarm Conditions](/images/data-collection/23b33.png)

4. **Configure actions**:

   - **Alarm state trigger**: `In alarm`
   - **SNS topic**: `weather-etl-alerts`

5. **Add name and description**:

   - **Alarm name**: `WeatherCurrentCollector-Errors`
   - **Description**: `Alert khi Lambda weather-current-collector có lỗi`

6. **Create alarm**
![Alarm Conditions](/images/data-collection/23b34.png)


**Quan trọng**: Hãy đảm bảo tất cả hoạt động đúng trước khi để nó tự động chạy

## Tối ưu hóa Chi phí và Hiệu suất

### 1. Smart Scheduling Strategy

**Thay vì chạy cùng frequency 24/7, có thể tối ưu:**

- **Giờ cao điểm** (6:00-23:00): Mỗi giờ
- **Giờ thấp điểm** (23:00-6:00): Mỗi 2 giờ
- **Cuối tuần**: Mỗi 2 giờ (ít người quan tâm thời tiết công việc)


**Tạo multiple rules với different schedules:**

**Rule 1 - Peak Hours:**

```
cron(0 6-23 * * ? *)
```

**Rule 2 - Off-peak Hours:**

```
cron(0 0,2,4 * * ? *)
```

### 2. Regional Optimization

**Nếu cần collect data cho multiple regions:**

```json
{
  "source": "eventbridge-schedule",
  "detail": {
    "region": "southeast-asia",
    "cities": ["HoChiMinh", "Hanoi", "Bangkok"],
    "priority": "high"
  }
}
```

### 3. Error Recovery Strategy

**Thêm Dead Letter Queue cho Lambda:**

1. **Lambda function configuration** → **"Asynchronous invocation"**
2. **Dead letter queue**: Enable với SQS queue
3. **Maximum age of event**: 6 hours
4. **Retry attempts**: 2

## Tóm tắt

Trong phần này, chúng ta đã hoàn thành:

**Thiết lập EventBridge Rules**:

- Current weather: Mỗi giờ (24 lần/ngày)
- Forecast: Mỗi 6 giờ (4 lần/ngày)

**Testing và Verification**:

- Manual testing các rules
- Verify S3 data collection
- Check CloudWatch logs và metrics

**Kết quả**: Hệ thống thu thập dữ liệu thời tiết tự động 24/7 với monitoring đầy đủ!

**Tiếp theo**: Trong module 2.4, chúng ta sẽ thiết lập testing và validation toàn diện để đảm bảo data quality và system reliability.
