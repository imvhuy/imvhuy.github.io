+++
title = "Thiết lập OpenWeatherMap API"
date = 2025-01-03T08:30:00+07:00
weight = 2
+++

# Thiết lập OpenWeatherMap API

Trong phần đơn giản hóa này, chúng ta sẽ nhanh chóng thiết lập tài khoản OpenWeatherMap API để thu thập dữ liệu thời tiết cho pipeline của chúng ta.

## Bước 1: Đăng ký OpenWeatherMap

1. **Tạo tài khoản**
   - Truy cập [https://openweathermap.org](https://openweathermap.org) và nhấp vào "Sign Up"
   - Hoàn thành đăng ký với email của bạn
   - Xác minh email và đăng nhập

![Create account](/images/data-collection/21b1.png)

## Bước 2: Lấy API Key

1. **Truy cập API Keys**
   - Sau khi đăng nhập, đi đến phần "API keys"
   - Ghi chú API key mặc định hoặc tạo key mới với tên "weather-data-collection"

![Get API key](/images/data-collection/21b2.png)

{{% notice info %}}
Gói miễn phí bao gồm 1,000 lệnh gọi API mỗi ngày và 60 lệnh gọi mỗi phút, từng này là đủ cho workshop của tôi.
{{% /notice %}}

## Bước 3: Kiểm tra API Key

Kiểm tra API key của bạn bằng một trong các phương pháp sau:

**Phương pháp trình duyệt:**

```
https://api.openweathermap.org/data/2.5/weather?lat=10.7769&lon=106.7009&appid=YOUR_API_KEY
```

![check API key](/images/data-collection/21b31.png)

**Phương pháp cURL:**

```bash
curl "https://api.openweathermap.org/data/2.5/weather?lat=10.7769&lon=106.7009&appid=YOUR_API_KEY"
```

![check API key](/images/data-collection/21b32.png)

Bạn sẽ thấy phản hồi JSON với dữ liệu thời tiết hiện tại của Thành phố Hồ Chí Minh.

## Các API Endpoints chính sẽ sử dụng

```
# Thời tiết hiện tại
https://api.openweathermap.org/data/2.5/weather?q={city}&appid={API key}

# Dự báo 5 ngày (khoảng thời gian 3 giờ)
https://api.openweathermap.org/data/2.5/forecast?q={city}&appid={API key}
```

## Các bước tiếp theo

Vậy là xong! Bạn đã có API key OpenWeatherMap hoạt động được lưu trữ an toàn trong Parameter Store. Trong phần tiếp theo, chúng ta sẽ tạo một Lambda function để thu thập dữ liệu thời tiết.

{{% notice success %}}

**Đã hoàn thành:**

- Tạo tài khoản OpenWeatherMap
- Lấy API key
- Lưu trữ API key trong AWS Parameter Store
  
{{% /notice %}}
