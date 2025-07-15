---
title: "Dọn dẹp Tài nguyên"
date: 2025-01-07T09:00:00+00:00
weight: 60
chapter: false
pre: "<b>6. </b>"
---


Chúc mừng bạn đã xây dựng xong pipeline ETL thời tiết serverless hoàn chỉnh! Module cuối cùng này tập trung vào việc dọn dẹp đúng cách tất cả tài nguyên AWS để tránh các khoản phí liên tục và khám phá các cơ hội cải tiến thêm.

## Tổng quan Module

Sau khi hoàn thành workshop thực hành, việc dọn dẹp tài nguyên là rất quan trọng để ngăn ngừa các khoản phí bất ngờ. Module này cung cấp các thủ tục dọn dẹp toàn diện và gợi ý các bước tiếp theo để mở rộng pipeline thời tiết của bạn.

## Mục tiêu Học tập

Sau khi hoàn thành module này, bạn sẽ:

- Hiểu tầm quan trọng của việc dọn dẹp tài nguyên trong AWS
- Thực hiện các thủ tục dọn dẹp toàn diện cho tất cả tài nguyên đã tạo
- Học các chiến lược tối ưu chi phí cho các dự án tương lai
- Khám phá các tính năng nâng cao và cải tiến cho pipeline của bạn
- Khám phá các tài nguyên học tập bổ sung

## Tầm quan trọng của Dọn dẹp

Dọn dẹp đúng cách là cần thiết vì:

- **Quản lý Chi phí**: Ngăn ngừa các khoản phí bất ngờ từ tài nguyên không sử dụng
- **Bảo mật**: Loại bỏ các quyền và điểm truy cập không sử dụng
- **Best Practices**: Thiết lập thói quen quản lý tài nguyên AWS tốt
- **Vệ sinh Tài khoản**: Giữ tài khoản AWS của bạn có tổ chức và dễ quản lý

## Danh sách Kiểm tra Dọn dẹp Hoàn chỉnh

### 6.1 Tài nguyên QuickSight

**Ưu tiên: Cao (Chi phí đăng ký liên tục)**

1. **Xóa Dashboard**

   - Loại bỏ dashboard thời tiết của bạn
   - Xóa mọi phiên bản đã chia sẻ

2. **Xóa Dataset**

   - Loại bỏ dataset dữ liệu thời tiết
   - Xóa mọi dữ liệu cached

3. **Hủy Đăng ký** (nếu không còn cần thiết)
   - Hạ cấp từ Author xuống Reader
   - Hủy đăng ký để tránh phí hàng tháng

### 6.2 Tài nguyên Athena

**Ưu tiên: Trung bình (Lưu trữ kết quả truy vấn)**

1. **Drop Tables**

   ```sql
   DROP TABLE weather_data_processed;
   DROP TABLE weather_data_raw;
   ```

2. **Drop Database**

   ```sql
   DROP DATABASE weather_analytics;
   ```

3. **Dọn dẹp Kết quả Truy vấn**
   - Xóa các file kết quả truy vấn Athena từ S3

### 6.3 S3 Bucket và Dữ liệu

**Ưu tiên: Cao (Chi phí lưu trữ)**

1. **Xóa Raw Data Bucket**

   ```bash
   aws s3 rm s3://weather-data-raw-[YOUR-ID] --recursive
   aws s3 rb s3://weather-data-raw-[YOUR-ID]
   ```

2. **Xóa Processed Data Bucket**

   ```bash
   aws s3 rm s3://weather-data-processed-[YOUR-ID] --recursive
   aws s3 rb s3://weather-data-processed-[YOUR-ID]
   ```

3. **Xóa Athena Results Bucket**
   ```bash
   aws s3 rm s3://aws-athena-query-results-[ACCOUNT-ID]-[REGION] --recursive
   ```

### 6.4 Lambda Function

**Ưu tiên: Trung bình (Chi phí liên tục tối thiểu)**

1. **Xóa Weather Collector Function**

   ```bash
   aws lambda delete-function --function-name weather-data-collector
   ```

2. **Xóa Weather Processor Function**

   ```bash
   aws lambda delete-function --function-name weather-data-processor
   ```

3. **Loại bỏ Function Trigger**
   - Xóa EventBridge rules
   - Loại bỏ S3 event notification

### 6.5 Tài nguyên CloudWatch

**Ưu tiên: Thấp (Free tier bao gồm hầu hết việc sử dụng)**

1. **Xóa Log Group**

   ```bash
   aws logs delete-log-group --log-group-name /aws/lambda/weather-data-collector
   aws logs delete-log-group --log-group-name /aws/lambda/weather-data-processor
   ```

2. **Xóa EventBridge Rule**

   ```bash
   aws events delete-rule --name weather-collection-schedule
   ```

3. **Loại bỏ Custom Metric** (nếu có)

### 6.6 IAM Role và Policy

**Ưu tiên: Trung bình (Bảo mật và tổ chức)**

1. **Detach Policy từ Role**

   ```bash
   aws iam detach-role-policy --role-name weather-lambda-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
   aws iam detach-role-policy --role-name weather-lambda-role --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
   ```

2. **Xóa IAM Role**

   ```bash
   aws iam delete-role --role-name weather-lambda-role
   aws iam delete-role --role-name quicksight-athena-role
   ```

3. **Loại bỏ Custom Policy** (nếu đã tạo)

### 6.7 Systems Manager Parameter

**Ưu tiên: Thấp (Không có chi phí liên tục)**

1. **Xóa Parameter Store Value**
   ```bash
   aws ssm delete-parameter --name /weather-etl/openweathermap/api-key
   ```

## Tóm tắt Chi phí Cuối cùng

Sau khi hoàn thành dọn dẹp, tổng chi phí workshop ước tính của bạn sẽ là:

- **Lambda Execution**: ~$0.10 (trong Free Tier)
- **S3 Storage**: ~$0.20 (dữ liệu tối thiểu được lưu trữ)
- **Athena Queries**: ~$0.15 (truy vấn dataset nhỏ)
- **QuickSight**: ~$3-4 (nếu sử dụng free trial)
- **Data Transfer**: ~$0.05 (tối thiểu)

**Tổng Chi phí Workshop**: Dưới $5 (không bao gồm đăng ký QuickSight)

## Bước tiếp theo và Cải tiến

### Cải tiến Ngay lập tức

1. **Thu thập Dữ liệu Nâng cao**

   - Thêm nhiều thành phố và thông số thời tiết
   - Triển khai xử lý lỗi và retry
   - Thêm xác thực và kiểm tra chất lượng dữ liệu

2. **Xử lý Nâng cao**

   - Triển khai tổng hợp dữ liệu và xu hướng
   - Thêm cảnh báo và thông báo thời tiết
   - Bao gồm so sánh thời tiết lịch sử

3. **Phân tích Tốt hơn**
   - Tạo truy vấn Athena phức tạp hơn
   - Triển khai dự đoán machine learning
   - Thêm phân tích streaming real-time

### Cân nhắc Production

1. **Cải tiến Bảo mật**

   - Triển khai IAM policy least-privilege
   - Thêm mã hóa khi lưu trữ và truyền tải
   - Sử dụng AWS Secrets Manager cho API key

2. **Giám sát và Cảnh báo**

   - Thiết lập CloudWatch dashboard toàn diện
   - Triển khai SNS notification cho lỗi
   - Thêm custom metric và alarm

3. **Tối ưu hóa Hiệu suất**
   - Tối ưu hóa bộ nhớ và timeout setting Lambda
   - Triển khai S3 lifecycle policy
   - Thêm caching layer khi thích hợp

### Tài nguyên Học tập

- **AWS Documentation**: Tìm hiểu sâu hơn về các dịch vụ cụ thể
- **AWS Training**: Tham gia các khóa học AWS chính thức về data engineering
- **Community**: Tham gia diễn đàn và nhóm người dùng AWS
- **Certification**: Cân nhắc AWS Certified Data Analytics specialty

## Bước Xác minh

Sau khi dọn dẹp, xác minh mọi thứ đã được loại bỏ:

1. **Kiểm tra AWS Console**

   - Xem lại từng service console để tìm tài nguyên còn lại
   - Kiểm tra billing dashboard cho các khoản phí liên tục

2. **Xác minh CLI**

   ```bash
   aws s3 ls | grep weather
   aws lambda list-functions | grep weather
   aws iam list-roles | grep weather
   ```

3. **Giám sát Chi phí**
   - Thiết lập billing alert cho các dự án tương lai
   - Xem lại AWS Cost Explorer cho bất kỳ khoản phí bất ngờ nào

## Kết luận

Bạn đã xây dựng thành công và giờ đây dọn dẹp một pipeline ETL thời tiết serverless hoàn chỉnh! Trải nghiệm này cung cấp nền tảng vững chắc để xây dựng các hệ thống xử lý dữ liệu phức tạp hơn.

**Điểm chính:**

- Kiến trúc serverless tiết kiệm chi phí và có thể mở rộng
- Dọn dẹp tài nguyên đúng cách là cần thiết cho quản lý chi phí
- AWS cung cấp các công cụ mạnh mẽ cho thu thập, xử lý và trực quan hóa dữ liệu
- Pipeline ETL có thể được xây dựng từng bước và kiểm tra ở mỗi giai đoạn

**Cảm ơn bạn đã hoàn thành workshop này!** Các kỹ năng bạn đã học có thể được áp dụng cho nhiều thách thức data engineering trong thế giới thực.

---

**Workshop Hoàn thành** - Nhớ xác minh tất cả tài nguyên đã được dọn dẹp để tránh các khoản phí bất ngờ.
