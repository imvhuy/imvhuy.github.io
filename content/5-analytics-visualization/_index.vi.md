---
title: "Trực quan hóa Dữ liệu với QuickSight"
date: 2025-01-07T09:00:00+00:00
weight: 50
chapter: false
pre: "<b>5. </b>"
---

Khi đã có dữ liệu thời tiết được lưu trữ và có thể truy vấn thông qua Athena thì sẽ đến với việc trực quan hóa dữ liệu. Trong phần này, ta sẽ sử dụng Amazon QuickSight để xây dựng dashboard tương tác giúp dữ liệu thời tiết trở nên sinh động.

## Tổng quan

Amazon QuickSight là dịch vụ business intelligence (BI) của AWS giúp dễ dàng tạo và xuất bản các dashboard tương tác. Bạn sẽ kết nối QuickSight với nguồn dữ liệu Athena và tạo các trực quan hóa tiết lộ các mẫu và thông tin chi tiết từ dữ liệu thời tiết.

## Bước 1: Thiết lập Amazon QuickSight

### 1.1 Đăng ký QuickSight

1. Truy cập QuickSight
  - Đăng nhập AWS Console.
  - Ở thanh tìm kiếm, nhập QuickSight và chọn Amazon QuickSight.
2. Nhấp **Sign up for QuickSight**
3. Chọn **Standard Edition** (bao gồm dùng thử miễn phí 30 ngày)
4. Nhập thông tin tài khoản:
   - **Account name**: `weather-analytics-[ten-ban]`
   - **Notification email**: Địa chỉ email của bạn
5. Nhấp **Finish**


![Quicksight](/images/analytics-visualization/5b1.png)
![Quicksight](/images/analytics-visualization/5b2.png)
### 1.2 Cấu hình Quyền QuickSight

1. Trong QuickSight, nhấp **biểu tượng profile** (góc phải trên)
2. Chọn **Manage QuickSight**
3. Chọn **Security & permissions**
4. Nhấp **Manage**
5. Kích hoạt các dịch vụ sau:
   - **Amazon Athena**
   - **Amazon S3**
6. Cho S3, nhấp **Select S3 buckets**
7. Chọn bucket dữ liệu thời tiết:
   - `your-weather-processed-bucket`
   - `your-athena-query-results-bucket`
8. Nhấp **Save**

![Quicksight](/images/analytics-visualization/5b3.png)
## Bước 2: Tạo Data Source và Dataset

### 2.1 Kết nối với Athena

1. Trong trang chủ QuickSight, nhấp **Datasets**
2. Nhấp **New dataset**
3. Chọn **Athena** làm data source
4. Cấu hình kết nối:
   - **Data source name**: `Weather-Data-Athena`
   - **Athena workgroup**: `primary` (mặc định)
5. Nhấp **Create data source**

![Quicksight](/images/analytics-visualization/5b4.png)


### 2.2 Tạo Dataset từ Weather Table

1. Chọn database: `weather_analytics`
2. Chọn table: `current_weather`
3. Chọn **Directly query your data**
4. Nhấp **Visualize**
![Quicksight](/images/analytics-visualization/5b5.png)
![Quicksight](/images/analytics-visualization/5b6.png)


{{% notice tip %}}
Nếu dataset của bạn nhỏ (< 1GB), bạn có thể chọn "Import to SPICE" để có hiệu suất tốt hơn. SPICE là engine tính toán trong bộ nhớ của QuickSight.
{{% /notice %}}

## Bước 3: Tạo Trực quan hóa Thời tiết

### 3.1 Biểu đồ Đường Xu hướng Nhiệt độ

1. **Tạo phân tích mới**:

   - Nhấp **+ Add** → **Add visual**
   - Chọn **Line chart**

2. **Cấu hình biểu đồ**:

   - **X-axis**: Kéo `data_collection_date` vào X-axis
   - **Value**: Kéo `temperature_celsius` vào Value
   - **Color**: Kéo `city_name` vào Color

3. **Tùy chỉnh biểu đồ**:
   - Nhấp **visual** → **Format visual**
   - **Title**: "Xu hướng Nhiệt độ theo Thành phố"
   - **Y-axis label**: "Nhiệt độ (°C)"
   - **Legend**: Đặt ở dưới cùng

![Quicksight](/images/analytics-visualization/5b7.png)
### 3.2 Biểu đồ Tròn Điều kiện Thời tiết

1. **Thêm visual mới**:

   - Nhấp **+ Add** → **Add visual**
   - Chọn **Pie chart**

2. **Cấu hình biểu đồ**:

   - **Group/Color**: Kéo `weather_main` vào Group/Color
   - **Value**: Kéo `city_name` vào Value
   - **Aggregate**: Đổi thành **Count**

3. **Tùy chỉnh**:
   - **Title**: "Phân bố Điều kiện Thời tiết"
   - **Legend**: Hiển thị phần trăm

![Quicksight](/images/analytics-visualization/5b8.png)

### 3.3 Biểu đồ Cột So sánh Nhiệt độ Thành phố

1. **Thêm visual mới**:

   - Chọn **Vertical bar chart**

2. **Cấu hình**:

   - **X-axis**: Kéo `city_name` vào X-axis
   - **Value**: Kéo `temperature_celsius` vào Value
   - **Aggregate**: Đổi thành **Average**

3. **Tùy chỉnh**:
   - **Title**: "Nhiệt độ Trung bình theo Thành phố"
   - **Y-axis label**: "Nhiệt độ Trung bình (°C)"
   - Sắp xếp theo giá trị (giảm dần)

![Quicksight](/images/analytics-visualization/5b9.png)

### 3.4 Biểu đồ đường so sánh giữa temperature_celsius và feels_like_celsius

1. **Thêm visual mới**:

   - Chọn **Line chart**

2. **Cấu hình**:

   - **X-axis**: Kéo `timestamp` vào X-axis
   - **Value**: Kéo `temperature_celsius` và `feels_like_celsius` vào Value
   - **Aggregate**: Đổi thành **Average**

3. **Tùy chỉnh**:
   - **Title**: "So sánh nhiệt đồ ngoài trời và nhiệt độ cảm nhận"
   - **X-axis label**: "Thời gian"

![Quicksight](/images/analytics-visualization/5b10.png)

### 3.5 Chỉ số Hiệu suất Chính (KPIs)

Tạo các ô KPI cho dữ liệu thời tiết mới nhất:

#### KPI Nhiệt độ Hiện tại

1. **Thêm visual mới** → **KPI**
2. **Cấu hình**:
   - **Value**: `temperature_celsius`
   - **Aggregate**: **Average**
3. **Filter**: Thêm filter cho ngày mới nhất
4. **Title**: "Nhiệt độ Trung bình Hiện tại"

#### KPI Độ ẩm

1. **Thêm KPI visual**
2. **Cấu hình**:
   - **Value**: `humidity_percent`
   - **Aggregate**: **Average**
3. **Title**: "Độ ẩm Trung bình Hiện tại"


## Bước 4: Xây dựng Dashboard Toàn diện

### 4.1 Tổ chức Bố cục Dashboard

1. **Thay đổi kích thước và sắp xếp visuals**:

   - Đặt KPIs ở trên cùng theo hàng ngang
   - Biểu đồ xu hướng nhiệt độ ở khu vực chính
   - Biểu đồ tròn và biểu đồ cột cạnh nhau bên dưới

2. **Thêm tiêu đề dashboard**:
   - Nhấp **+ Add** → **Add title**
   - Text: "Dashboard Phân tích Thời tiết"
   - Style: Lớn, căn giữa

### 4.2 Thêm Bộ lọc Tương tác

1. **Thêm bộ lọc ngày**:

   - Nhấp **Filter** pane (bên trái)
   - Nhấp **Create one** → Chọn `data_collection_date`
   - Filter type: **Date range**
   - Default: 7 ngày gần nhất

2. **Thêm bộ lọc thành phố**:

   - Tạo filter cho `city_name`
   - Filter type: **Multi-select dropdown**
   - Hiển thị tất cả thành phố theo mặc định

3. **Thêm bộ lọc điều kiện thời tiết**:
   - Tạo filter cho `weather_main`
   - Filter type: **Multi-select dropdown**

### 4.3 Áp dụng Styling Dashboard

1. **Chọn color theme**:

   - Nhấp **Themes** (menu trên)
   - Chọn **Midnight** hoặc **Classic**

2. **Tùy chỉnh màu sắc**:

   - Cho biểu đồ nhiệt độ: Sử dụng gradient xanh-đỏ
   - Cho điều kiện thời tiết: Sử dụng màu riêng biệt cho mỗi điều kiện

3. **Thêm text mô tả**:
   - Nhấp **+ Add** → **Add text box**
   - Thêm insights hoặc hướng dẫn cho người dùng dashboard

## Bước 5: Trực quan hóa Nâng cao

### Biểu đồ Hướng Gió (Sử dụng Calculated Fields)

1. **Tạo calculated field**:

   - Nhấp **+ Add** → **Add calculated field**
   - **Name**: `wind_direction_category`
   - **Formula**:

   ```
   ifelse(
     wind_direction_deg >= 337.5 OR wind_direction_deg < 22.5, "N",
     wind_direction_deg >= 22.5 AND wind_direction_deg < 67.5, "NE",
     wind_direction_deg >= 67.5 AND wind_direction_deg < 112.5, "E",
     wind_direction_deg >= 112.5 AND wind_direction_deg < 157.5, "SE",
     wind_direction_deg >= 157.5 AND wind_direction_deg < 202.5, "S",
     wind_direction_deg >= 202.5 AND wind_direction_deg < 247.5, "SW",
     wind_direction_deg >= 247.5 AND wind_direction_deg < 292.5, "W",
     "NW"
   )
   ```

2. **Tạo biểu đồ hướng gió**:
   - **Visual type**: Donut chart
   - **Group**: `wind_direction_category`
   - **Value**: Count of records
   - **Title**: "Phân bố Hướng Gió"

![Quicksight](/images/analytics-visualization/5b11.png)

## Bước 6: Xuất bản và Chia sẻ Dashboard

### 6.1 Xuất bản Dashboard

1. Nhấp **Share** (góc phải trên)
2. Nhấp **Publish dashboard**
3. **Dashboard name**: "Dashboard Phân tích Thời tiết"
4. **Description**: "Trực quan hóa dữ liệu thời tiết tương tác hiển thị xu hướng nhiệt độ, mẫu thời tiết và so sánh thành phố"
5. Nhấp **Publish dashboard**

### 6.2 Thiết lập Quyền Chia sẻ

1. Trong dashboard đã xuất bản, nhấp **Share**
2. **Add users or groups**:
   - Nhập địa chỉ email của người dùng để chia sẻ
   - Đặt quyền: **Viewer** hoặc **Co-owner**
3. Nhấp **Share**

### 6.3 Tạo Dashboard Công khai (Tùy chọn)

1. Nhấp **Share** → **Manage dashboard access**
2. Kích hoạt **Make dashboard public**
3. Sao chép URL công khai để chia sẻ bên ngoài

{{% notice warning %}}
Chỉ công khai dashboard nếu dữ liệu không chứa thông tin nhạy cảm.
{{% /notice %}}
![Quicksight](/images/analytics-visualization/5b12.png)
## Bước 7: Thiết lập Làm mới Dữ liệu Tự động

### 7.1 Cấu hình Làm mới Dataset

1. Đi đến **Datasets**
2. Chọn weather dataset của bạn
3. Nhấp **Refresh** → **Schedule refresh**
4. **Cấu hình lịch**:
   - **Frequency**: Daily
   - **Time**: Sáng sớm (ví dụ: 6 AM)
   - **Time zone**: Múi giờ địa phương của bạn
5. **Lưu lịch**

### 7.2 Giám sát Trạng thái Làm mới

1. Kiểm tra lịch sử làm mới trong **Dataset settings**
2. Thiết lập **thông báo** cho làm mới thất bại
3. Giám sát chỉ báo độ tươi của dữ liệu trong dashboard
![Quicksight](/images/analytics-visualization/5b13.png)
## Bước 8: Tối ưu hóa Dashboard và Best Practices


Kết quả:

- Dashboard thời tiết tương tác với nhiều trực quan hóa
- Làm mới dữ liệu tự động từ ETL pipeline
- Insights có thể chia sẻ cho phân tích thời tiết
- Hiểu biết về best practices của QuickSight




