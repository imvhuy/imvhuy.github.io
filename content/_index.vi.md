---
title: "Xây dựng Pipeline ETL Thời tiết Serverless"
date: 2025-07-09T09:00:00+00:00
weight: 1
chapter: false
---

## Tổng quan Workshop

Trong workshop này, em sẽ tạo một pipeline dữ liệu thời tiết đơn giản nhưng hoàn chỉnh, minh họa các khái niệm ETL cốt lõi sử dụng công nghệ serverless của AWS. 

## Những gì bạn sẽ Xây dựng

Workshop này trình bày cách xây dựng một pipeline ETL đơn giản sử dụng công nghệ serverless của AWS:

- **Thu thập** dữ liệu thời tiết từ OpenWeatherMap API sử dụng AWS Lambda
- **Xử lý** và chuyển đổi dữ liệu thô thành định dạng phân tích
- **Lưu trữ** dữ liệu trong Amazon S3 cho cả dữ liệu thô và đã xử lý
- **Phân tích** dữ liệu sử dụng Amazon Athena với truy vấn SQL
- **Trực quan hóa** insights thông qua Amazon QuickSight dashboard
- **Dọn dẹp** tài nguyên để tối ưu chi phí

## Kiến trúc Tổng quan

Pipeline ETL thời tiết của chúng ta tuân theo kiến trúc serverless đơn giản này:

```
OpenWeatherMap API → Lambda Collector → S3 Raw Data → Lambda Processor → S3 Processed Data → Athena Analytics → QuickSight Dashboard
```

## Các Module Workshop

Workshop này được tổ chức thành 6 module:

### 1. [Giới thiệu và Kiến trúc](1-introduction/)

Tổng quan workshop và mục tiêu học tập, thiết kế kiến trúc và giới thiệu các dịch vụ AWS, yêu cầu tiên quyết và thiết lập.

### 2. [Thu thập Dữ liệu Thời tiết với OpenWeatherMap](2-data-collection-openweathermap/)

Thiết lập tài khoản OpenWeatherMap API, tạo Lambda function cho thu thập dữ liệu, cấu hình tự động lấy dữ liệu, và kiểm tra giám sát quá trình thu thập.

### 3. [Xử lý Dữ liệu Serverless với Lambda](3-serverless-processing-lambda/)

Xây dựng Lambda function chuyển đổi dữ liệu, chuyển đổi JSON thời tiết thô sang định dạng phân tích, triển khai xác thực và làm giàu dữ liệu.

### 4. [Phân tích Dữ liệu với Amazon Athena](4-data-storage-solutions/)

Tạo cấu trúc data lake S3, thiết lập bảng và schema Athena, viết truy vấn SQL cho phân tích thời tiết, và khám phá các mẫu dữ liệu.

### 5. [Trực quan hóa Dữ liệu với QuickSight](5-analytics-visualization/)

Thiết lập Amazon QuickSight, tạo dashboard thời tiết, xây dựng trực quan hóa tương tác, và chia sẻ xuất bản dashboard.

### 6. [Dọn dẹp Tài nguyên và Bước tiếp theo](6-cleanup-next-steps/)

Danh sách kiểm tra dọn dẹp toàn diện, chiến lược tối ưu chi phí, đề xuất cải tiến và mở rộng, và tài nguyên học tập bổ sung.

## Mục tiêu Học tập

Sau khi hoàn thành workshop này, bạn sẽ:

- Xây dựng hệ thống thu thập dữ liệu serverless sử dụng AWS Lambda
- Triển khai quy trình chuyển đổi và xử lý dữ liệu
- Lưu trữ và truy vấn dữ liệu sử dụng Amazon S3 và Athena
- Tạo trực quan hóa với Amazon QuickSight
- Áp dụng các best practices của AWS cho tối ưu chi phí và dọn dẹp tài nguyên

## Yêu cầu Tiên quyết

Trước khi bắt đầu workshop này, hãy đảm bảo bạn có:

- **Tài khoản AWS** với quyền truy cập quản trị
- **Tài khoản OpenWeatherMap** (tier miễn phí là đủ)
- **Hiểu biết cơ bản** về AWS console
- **Hiểu biết** về các khái niệm lập trình cơ bản
- **Hiểu biết** về định dạng dữ liệu JSON