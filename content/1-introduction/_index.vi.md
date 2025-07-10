---
title: "Giới thiệu"
date: 2025-07-09T09:00:00+00:00
weight: 10
chapter: false
pre: "<b>1. </b>"
---

## Tổng quan Workshop

Trong workshop này, em sẽ tạo một pipeline dữ liệu thời tiết đơn giản nhưng hoàn chỉnh, minh họa các khái niệm ETL cốt lõi sử dụng công nghệ serverless của AWS. 

## Mục tiêu 
- Xây dựng hệ thống thu thập dữ liệu serverless sử dụng AWS Lambda
- Triển khai quy trình chuyển đổi và xử lý dữ liệu
- Lưu trữ và truy vấn dữ liệu sử dụng Amazon S3 và Athena
- Tạo trực quan hóa với Amazon QuickSight
- Áp dụng các best practices của AWS cho tối ưu chi phí và dọn dẹp tài nguyên

## Tổng quan Kiến trúc

Pipeline ETL thời tiết của chúng ta tuân theo kiến trúc serverless đơn giản này:

```
OpenWeatherMap API → Lambda Collector → S3 Raw Data → Lambda Processor → S3 Processed Data → Athena Analytics → QuickSight Dashboard
```

**Các thành phần chính:**

- **Nguồn dữ liệu**: OpenWeatherMap API cho dữ liệu thời tiết thời gian thực
- **Thu thập**: AWS Lambda function để lấy dữ liệu thời tiết
- **Lưu trữ**: Amazon S3 cho cả dữ liệu thô và đã xử lý
- **Xử lý**: AWS Lambda cho chuyển đổi dữ liệu
- **Phân tích**: Amazon Athena cho truy vấn SQL
- **Trực quan hóa**: Amazon QuickSight cho dashboard

## Các Module Workshop

Workshop này được tổ chức thành 6 module:

### **Module 1: Giới thiệu và Kiến trúc**

- Tổng quan workshop và mục tiêu học tập
- Thiết kế kiến trúc và giới thiệu các dịch vụ AWS
- Yêu cầu tiên quyết và thiết lập

### **Module 2: Thu thập Dữ liệu Thời tiết với OpenWeatherMap**

- Thiết lập tài khoản OpenWeatherMap API
- Tạo Lambda function cho thu thập dữ liệu
- Cấu hình tự động lấy dữ liệu
- Kiểm tra và giám sát quá trình thu thập

### **Module 3: Xử lý Dữ liệu Serverless với Lambda**

- Xây dựng Lambda function chuyển đổi dữ liệu
- Chuyển đổi JSON thời tiết thô sang định dạng phân tích
- Triển khai xác thực và làm giàu dữ liệu
- Thiết lập triggers xử lý

### **Module 4: Phân tích Dữ liệu với Amazon Athena**

- Tạo cấu trúc data lake S3
- Thiết lập bảng và schema Athena
- Viết truy vấn SQL cho phân tích thời tiết
- Khám phá các mẫu và thông tin chi tiết từ dữ liệu

### **Module 5: Trực quan hóa Dữ liệu với QuickSight**

- Thiết lập Amazon QuickSight
- Tạo dashboard thời tiết
- Xây dựng trực quan hóa tương tác
- Chia sẻ và xuất bản dashboard

### **Module 6: Dọn dẹp Tài nguyên và Bước tiếp theo**

- Danh sách kiểm tra dọn dẹp toàn diện
- Chiến lược tối ưu chi phí
- Đề xuất cải tiến và mở rộng
- Tài nguyên học tập bổ sung

## Yêu cầu Tiên quyết

Trước khi bắt đầu workshop này, hãy đảm bảo bạn có:

- **Tài khoản AWS** với quyền truy cập quản trị
- **Hiểu biết cơ bản** về AWS console
- **Hiểu biết** về các khái niệm lập trình cơ bản
- **Tài khoản OpenWeatherMap** (tier miễn phí là đủ)

## Ước tính Chi phí

Workshop này được thiết kế để tiết kiệm chi phí:

- **OpenWeatherMap API**: Miễn phí (lên đến 1,000 cuộc gọi/ngày)
- **AWS Lambda**: ~$1-2 (trong phạm vi free tier)
- **Amazon S3**: ~$1-2 cho lưu trữ
- **Amazon Athena**: ~$2-3 cho truy vấn
- **Amazon QuickSight**: ~$3-4 (dùng thử miễn phí 30 ngày)

**Tổng chi phí ước tính**: Dưới $10 cho toàn bộ workshop

## Bắt đầu

Sẵn sàng bắt đầu? Hãy bắt đầu với **Module 2: Thu thập Dữ liệu Thời tiết với OpenWeatherMap** nơi bạn sẽ thiết lập nguồn dữ liệu và tạo Lambda function đầu tiên.

---

**Lưu ý**: Nhớ tuân theo các thủ tục dọn dẹp trong Module 6 để tránh các khoản phí liên tục sau khi hoàn thành workshop.
