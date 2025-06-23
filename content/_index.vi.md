---
title: "Xây dựng Hệ thống ETL Data Pipeline cho E-commerce trên AWS"
date: 2025-06-23
weight: 1
chapter: false
---

# Xây dựng ETL Data Pipeline cho E-commerce trên AWS

#### Tổng quan

Workshop này sẽ xây dựng một **hệ thống ETL (Extract, Transform, Load) data pipeline** cho nền tảng thương mại điện tử sử dụng các dịch vụ AWS và dữ liệu được lấy từ DummyJSON API, thực hiện tạo ra một hệ thống xử lý dữ liệu đơn giản, hiệu quả chi phí để thu thập, biến đổi và phân tích dữ liệu e-commerce thực tế bao gồm sản phẩm, người dùng và giỏ hàng.

#### Bài Workshop này sẽ thực hiện để xây dựng data pineline có khả năng:
- **Thu thập** dữ liệu e-commerce thực tế từ DummyJSON API
- **Xử lý** và biến đổi dữ liệu sử dụng AWS Lambda
- **Lưu trữ** dữ liệu có cấu trúc trong S3 Data Lake
- **Phân tích** dữ liệu sử dụng Amazon Athena
- **Trực quan hóa** thông tin kinh doanh thông qua QuickSight dashboards

![Kiến trúc ETL Pipeline](/images/etl/image.png?featherlight=false&width=90pc)

#### Các dịch vụ AWS sẽ dùng ở workshop này
**Thu thập Dữ liệu:**
- **AWS Lambda** - Serverless compute để thu thập và xử lý dữ liệu
- **CloudWatch Events** - Thực thi theo lịch trình và tự động hóa

**Lưu trữ Dữ liệu:**
- **Amazon S3** - Object storage có thể mở rộng cho data lake
- **S3 Intelligent Tiering** - Tối ưu hóa chi phí lưu trữ dữ liệu

**Phân tích & Trực quan hóa:**
- **Amazon Athena** - Dịch vụ truy vấn tương tác cho dữ liệu S3
- **Amazon QuickSight** - Business intelligence và visualization

**Giám sát & Quản lý:**
- **Amazon CloudWatch** - Monitoring, logging và alerting
- **AWS IAM** - Quản lý danh tính và quyền truy cập

**Tích hợp Bên ngoài:**
- **DummyJSON API** - Dữ liệu e-commerce 

#### Các trường hợp sử dụng kinh doanh
ETL pipeline này thể hiện các kịch bản phân tích e-commerce thực tế:
- **Phân tích Sản phẩm** - Phân tích hiệu suất sản phẩm và thông tin kho hàng
- **Phân tích Khách hàng** - Nhân khẩu học và mô hình hành vi người dùng
- **Phân tích Bán hàng** - Phân tích giỏ hàng và theo dõi chuyển đổi
- **Nghiên cứu Thị trường** - xu hướng danh mục sản phẩm và phân tích giá
- **Business Intelligence** - Dashboard điều hành và báo cáo KPI

#### Các thành phần kiến trúc
1. **Nguồn dữ liệu** - DummyJSON API (sản phẩm, người dùng, giỏ hàng, bài viết, bình luận)
2. **Lớp thu thập** - Lambda functions được lên lịch để lấy dữ liệu
3. **Lớp xử lý** - Lambda functions để biến đổi dữ liệu và phân vùng
4. **Lớp lưu trữ** - S3 Data Lake với cấu trúc tối ưu
5. **Lớp phân tích** - Athena để truy vấn SQL và QuickSight để visualization
6. **Giám sát** - CloudWatch cho logging, metrics và alerting

#### Mục đích
- Hiểu kiến trúc data pipeline serverless hiện đại
- Thành thạo AWS Lambda để thu thập và xử lý dữ liệu
- Xây dựng khả năng phân tích batch sử dụng dữ liệu thực
- Triển khai monitoring và tối ưu hóa chi phí
- Tạo dashboard kinh doanh tương tác với QuickSight
- Tích hợp external APIs vào AWS data pipelines
- Tối ưu hóa chi phí với ít dịch vụ AWS (dưới $3/tháng)

#### Thời gian workshop

- **Tổng thời gian**: 4-6 giờ
- **Mức độ kỹ năng**: Beginner đến Intermediate
- **Chi phí**: Dưới $3 sử dụng AWS Free Tier (kiến trúc đơn giản hóa)

#### Yêu cầu tiên quyết

- Tài khoản AWS hoạt động với quyền quản trị
- Hiểu biết cơ bản về khái niệm cloud computing
- Quen thuộc với định dạng dữ liệu JSON và REST APIs
- Kết nối Internet để truy cập DummyJSON API
- Tùy chọn: Kiến thức cơ bản Python hoặc SQL

#### Các module workshop

1. [Giới thiệu & Thiết kế kiến trúc](1-introduction-architecture/)
2. [Thu thập Dữ liệu với Lambda](2-data-collection-lambda/)
3. [Xử lý và Biến đổi Dữ liệu](3-data-processing/)
4. [Thiết lập S3 Data Lake](4-s3-data-lake/)
5. [Phân tích với Amazon Athena](5-analytics-athena/)
6. [Visualization với QuickSight](6-visualization-quicksight/)
7. [Giám sát và Tối ưu hóa](7-monitoring-optimization/)
8. [Dọn dẹp và Bước tiếp theo](8-cleanup-next-steps/)
