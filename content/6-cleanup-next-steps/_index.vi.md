---
title: "Dọn dẹp Tài nguyên"
date: 2025-01-07T09:00:00+00:00
weight: 60
chapter: false
pre: "<b>6. </b>"
---

Sau khi hoàn thành workshop thực hành, việc dọn dẹp tài nguyên là rất quan trọng để ngăn ngừa các khoản phí bất ngờ. M


### 6.1 Tài nguyên QuickSight

1. **Xóa Dashboard**

Chúng ta sẽ dọn dẹp theo thứ tự ngược lại của quá trình tạo để tránh các lỗi phụ thuộc (ví dụ: không thể xóa S3 bucket nếu nó vẫn còn được sử dụng bởi một dịch vụ khác).

### **Bước 1: Hủy đăng ký Amazon QuickSight**

QuickSight là dịch vụ có phí đăng ký hàng tháng, vì vậy hãy ưu tiên hủy nó trước.

1.  **Truy cập QuickSight**:
    - Mở AWS Console, tìm và chọn **QuickSight**.
    - Nhấp vào **Go to QuickSight** để vào giao diện quản lý.
2.  **Xóa các tài sản (Analyses, Dashboards, Datasets)**:
    - Trong menu bên trái, đi đến từng mục **Dashboards**, **Analyses**, và **Datasets**.
    - Với mỗi mục, chọn tất cả các tài sản liên quan đến workshop (ví dụ: "Dashboard Phân tích Thời tiết") và nhấp **Delete**. Xác nhận xóa.
3.  **Hủy đăng ký (Unsubscribe)**:
    - Ở góc trên bên phải, nhấp vào **biểu tượng profile** của bạn và chọn **Manage QuickSight**.
    - Trong menu bên trái, chọn **Your Account**.
    - Nhấp vào **Manage account**. Một hộp thoại xác nhận sẽ hiện ra. Confirm và tiến hành xóa tài khoản

### **Bước 2: Xóa tài nguyên Amazon Athena**

Athena tự nó không tốn phí, nhưng các kết quả truy vấn được lưu trong S3 thì có.

1.  **Truy cập Athena**:
    - Mở AWS Console, tìm và chọn **Athena**.
2.  **Xóa bảng (Drop Table)**:
    - Trong **Query editor**, đảm bảo bạn đã chọn database `weather_analytics`.
    - Chạy lệnh sau để xóa bảng:
      ```sql
      DROP TABLE IF EXISTS weather_analytics.current_weather;
      ```
3.  **Xóa cơ sở dữ liệu (Drop Database)**:
    - Sau khi xóa bảng, chạy lệnh sau:
      ```sql
      DROP DATABASE IF EXISTS weather_analytics;
      ```

### **Bước 3: Xóa Amazon S3 Buckets**

S3 tính phí lưu trữ, vì vậy việc xóa bucket là rất quan trọng.

{{% notice danger %}}
Bạn phải làm trống bucket (delete all objects) trước khi có thể xóa bucket.
{{% /notice %}}

1.  **Truy cập S3**:
    - Mở AWS Console, tìm và chọn **S3**.
2.  **Làm trống và xóa từng Bucket**:
    - Thực hiện các bước sau cho cả 3 bucket:
      - `your-weather-raw-bucket-[ID]`
      - `your-weather-processed-bucket-[ID]`
      - `your-athena-query-results-bucket-[ID]`
    - **Các bước cho mỗi bucket**:
      1.  Nhấp vào tên bucket để mở nó.
      2.  Chọn tất cả các đối tượng và thư mục bên trong.
      3.  Nhấp vào nút **Delete**.
      4.  Trong màn hình xác nhận, nhập `permanently delete` và nhấp **Delete objects**.
      5.  Sau khi bucket trống, quay lại danh sách các bucket.
      6.  Chọn bucket (đã trống) và nhấp **Delete**.
      7.  Trong màn hình xác nhận, nhập tên bucket và nhấp **Delete bucket**.

### **Bước 4: Xóa tài nguyên Scheduling và Lambda**

1.  **Xóa EventBridge (CloudWatch Events) Rule**:
    - Mở AWS Console, tìm và chọn **Amazon EventBridge**.
    - Trong menu bên trái, chọn **Rules**.
    - Chọn rule bạn đã tạo (ví dụ: `weather-collection-schedule`).
    - Nhấp **Delete** và xác nhận.
2.  **Xóa Lambda Functions**:
    - Mở AWS Console, tìm và chọn **Lambda**.
    - Xóa từng hàm một:
      - Chọn hàm `weather-data-collector`. Nhấp **Actions** > **Delete**. Xác nhận xóa.
      - Chọn hàm `weather-data-processor`. Nhấp **Actions** > **Delete**. Xác nhận xóa.

### **Bước 5: Xóa CloudWatch Log Groups**

Lambda tự động tạo các log group. Chúng chiếm dung lượng và có thể phát sinh chi phí nhỏ.

1.  **Truy cập CloudWatch**:
    - Mở AWS Console, tìm và chọn **CloudWatch**.
2.  **Xóa Log Groups**:
    - Trong menu bên trái, chọn **Log groups**.
    - Tìm và chọn các log group sau:
      - `/aws/lambda/weather-data-collector`
      - `/aws/lambda/weather-data-processor`
    - Nhấp **Actions** > **Delete log group(s)** và xác nhận.

### **Bước 6: Xóa IAM Roles và Policies**

Đây là bước quan trọng để dọn dẹp các quyền không cần thiết.

1.  **Truy cập IAM**:
    - Mở AWS Console, tìm và chọn **IAM**.
2.  **Xóa IAM Role cho Lambda**:
    - Trong menu bên trái, chọn **Roles**.
    - Tìm role bạn đã tạo cho Lambda (ví dụ: `weather-lambda-role`).
    - **Lưu ý**: Nếu role này có các policy đính kèm, bạn cần **detach** chúng trước khi có thể xóa role. Thông thường, khi xóa role, AWS sẽ tự động detach các policy do bạn tạo.
    - Chọn role, nhấp **Delete** và xác nhận.
3.  **Xóa IAM Role cho QuickSight** (nếu bạn đã tạo riêng):
    - Lặp lại quy trình trên cho role của QuickSight (ví dụ: `quicksight-athena-role`).

## Xác minh lần cuối

Sau khi hoàn thành các bước trên, hãy kiểm tra lại một lần nữa:

1.  **Kiểm tra Billing Dashboard**: Truy cập **AWS Billing** và xem có chi phí phát sinh bất thường nào không trong vài giờ tới.
2.  **Kiểm tra các dịch vụ**: Lướt qua console của S3, Lambda, IAM để đảm bảo không còn tài nguyên nào có tên liên quan đến workshop.

