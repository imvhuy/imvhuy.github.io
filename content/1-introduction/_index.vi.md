---
title: "Giới thiệu"
date: 2025-07-09T09:00:00+00:00
weight: 10
chapter: false
pre: "<b>1. </b>"
---

## Xây dựng ETL Data Pipeline cho phân tích thời tiết trên AWS
## Mục tiêu 
- Xây dựng hệ thống thu thập dữ liệu serverless sử dụng AWS Lambda
- Triển khai quy trình chuyển đổi và xử lý dữ liệu
- Lưu trữ và truy vấn dữ liệu sử dụng Amazon S3 và Athena
- Tạo trực quan hóa với Amazon QuickSight
- Áp dụng các best practices của AWS cho tối ưu chi phí và dọn dẹp tài nguyên

## Tổng quan Kiến trúc

Pipeline ETL thời tiết của chúng ta tuân theo kiến trúc serverless đơn giản này:
![ETL Pipeline Architecture](/images/etl/architecture.png)

**Các thành phần chính:**

- **Nguồn dữ liệu**: OpenWeatherMap API cho dữ liệu thời tiết thời gian thực
- **Thu thập**: AWS Lambda function để lấy dữ liệu thời tiết
- **Lưu trữ**: Amazon S3 cho cả dữ liệu thô và đã xử lý
- **Xử lý**: AWS Lambda cho chuyển đổi dữ liệu
- **Phân tích**: Amazon Athena cho truy vấn SQL
- **Trực quan hóa**: Amazon QuickSight cho dashboard
