---
title: "Hệ Thống Robot Tự Hành AMR 3 Bánh Omni"
excerpt: "Thiết kế robot di động đa hướng sử dụng ROS 2, LiDAR 2D và vi điều khiển ESP32.<br/><img src='/images/amr-preview.png'>"
collection: portfolio
---

### 1. Tổng quan dự án
Dự án xây dựng mô hình robot AMR sử dụng cấu hình bánh xe Omni 120 độ...

### 2. Thông số kỹ thuật
* **Bộ xử lý trung tâm:** Raspberry Pi / Laptop điều khiển qua ROS 2 Jazzy.
* **Vi điều khiển điều khiển động cơ:** ESP32 giao tiếp micro-ROS.
* **Cảm biến:** LiDAR quét 2D, IMU 6 trục và Camera nhận diện Apriltag.

### 3. Kết quả đạt được
- Robot bám quỹ đạo ổn định với sai số < 5%.
- Tích hợp thành công bản đồ SLAM và điều hướng tự động.
