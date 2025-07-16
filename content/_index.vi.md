---
title: "Xây dựng Pipeline ETL Thời tiết Serverless"
date: 2025-07-09T09:00:00+00:00
weight: 1
chapter: false
---

# Xây dựng ETL Data Pipeline cho phân tích thời tiết trên AWS

## Tổng quan Workshop

Trong workshop này, em sẽ tạo một pipeline dữ liệu thời tiết đơn giản, minh họa các khái niệm ETL cốt lõi sử dụng công nghệ serverless của AWS.

Workshop này trình bày cách xây dựng một pipeline ETL đơn giản sử dụng công nghệ serverless của AWS:

- **Thu thập** dữ liệu thời tiết từ OpenWeatherMap API sử dụng AWS Lambda
- **Xử lý** và chuyển đổi dữ liệu thô thành định dạng phân tích
- **Lưu trữ** dữ liệu trong Amazon S3 cho cả dữ liệu thô và đã xử lý
- **Phân tích** dữ liệu sử dụng Amazon Athena với truy vấn SQL
- **Trực quan hóa** insights thông qua Amazon QuickSight dashboard
- **Dọn dẹp** tài nguyên để tối ưu chi phí

### Công nghệ sử dụng:
- AWS Lambda, S3, Athena, QuickSight kết hợp với OpenWeatherMap API để xây dựng pipeline ETL serverless thu thập và phân tích dữ liệu thời tiết.

## Các phần chính

### 1. [Giới thiệu](1-introduction/)
- Tổng quan workshop và mục tiêu học tập
- Thiết kế kiến trúc và giới thiệu các dịch vụ AWS
- Yêu cầu tiên quyết và thiết lập

### 2. [Thu thập Dữ liệu Thời tiết với OpenWeatherMap](2-data-collection-openweathermap/)
- Thiết lập tài khoản OpenWeatherMap API
- Tạo Lambda function cho thu thập dữ liệu
- Cấu hình tự động lấy dữ liệu
- Kiểm tra và giám sát quá trình thu thập

### 3. [Xử lý Dữ liệu Serverless với Lambda](3-serverless-processing-lambda/)
- Xây dựng Lambda function chuyển đổi dữ liệu
- Chuyển đổi JSON thời tiết thô sang định dạng phân tích
- Thiết lập triggers xử lý

### 4. [Phân tích Dữ liệu với Amazon Athena](4-data-storage-solutions/)
- Tạo cấu trúc data lake S3
- Thiết lập bảng và schema Athena
- Viết truy vấn SQL cho phân tích thời tiết
- Khám phá các mẫu và thông tin chi tiết từ dữ liệu

### 5. [Trực quan hóa Dữ liệu với QuickSight](5-analytics-visualization/)
- Thiết lập Amazon QuickSight
- Tạo dashboard thời tiết
- Xây dựng trực quan hóa tương tác
- Chia sẻ và xuất bản dashboard

### 6. [Dọn dẹp Tài nguyên](6-cleanup-next-steps/)