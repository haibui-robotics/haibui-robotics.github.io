---
title: 'Tổng Quan Kiến Trúc & Kinh Nghiệm Phát Triển Hệ Thống Điều Khiển với ROS 2'
date: 2026-11-03
permalink: /posts/2026/11/kien-thuc-ros2/
tags:
  - ROS 2
  - Robotics
  - Control Systems
  - C++
---

Bài viết này tổng hợp các kiến trúc cốt lõi, công cụ và kinh nghiệm thực tế của tôi trong quá trình nghiên cứu, mô phỏng và phát triển hệ thống điều khiển robot với ROS 2 (Jazzy/Humble).

---

### 1. Kiến Trúc Cốt Lõi Của ROS 2 (Core Concepts)
* **Node & Lifecycle Node:** Cơ chế quản lý trạng thái của từng tiến trình (Unconfigured, Inactive, Active, Finalized), giúp hệ thống robot công nghiệp khởi động an toàn và xử lý sự cố có kiểm soát.
* **Mô hình Giao tiếp (Communication Paradigms):**
  * *Topics (Publish/Subscribe):* Truyền dữ liệu cảm biến tần số cao (LiDAR `/scan`, IMU `/imu/data`, Odometry `/odom`) với cấu hình QoS (Best Effort / Reliable, Transient Local) tối ưu hóa độ trễ qua mạng không dây.
  * *Services (Request/Response):* Các thao tác kích hoạt ngắt quãng như reset odometry, bật/tắt thiết bị ngoại vi.
  * *Action Servers (Goal/Feedback/Result):* Xử lý các tác vụ tốn thời gian như điều hướng di chuyển điểm đến, đóng mở cơ cấu gắp, trả về tiến độ liên tục và cho phép hủy ngang (preempt).

---

### 2. Mô Hình Hóa Robot & Mô Phỏng (URDF, Xacro & Gazebo)
* **Xacro & Khung Tọa Độ TF2:**
  * Thiết kế cây tọa độ (`/map` $\rightarrow$ `/odom` $\rightarrow$ `/base_link` $\rightarrow$ `/lidar_link`).
  * Sử dụng macro trong Xacro để tham số hóa kích thước robot, tính toán ma trận quán tính (Inertia Matrix) cho từng liên kết động học.
* **Mô phỏng vật lý với Gazebo Sim (Harmonic):**
  * Tích hợp plugin truyền động cho bánh xe (Differential drive, Omni 3-wheel kinematics).
  * Mô phỏng cảm biến LiDAR 2D và Camera chiều sâu gắn trực tiếp trên khung robot.

---

### 3. Tích Hợp Phần Cứng & Giao Tiếp Nhúng (Hardware Interface & micro-ROS)
* **ros2_control Framework:**
  * Xây dựng custom `hardware_interface` (C++) để đồng bộ hóa việc đọc encoder và ghi tín hiệu điều khiển động cơ với tần số vòng lặp cố định (ví dụ 50Hz - 100Hz).
* **micro-ROS trên Vi điều khiển (ESP32 / STM32):**
  * Giao tiếp hai chiều với máy tính nhúng (SBC) qua chuẩn Serial/UART hoặc Wi-Fi UDP.
  * Đồng bộ thời gian giữa microcontroller và hệ thống ROS 2 master.

---

### 4. Xử Lý Cảm Biến & Định Vị (Localization & Navigation)
* **Sensor Fusion (EKF):**
  * Ứng dụng package `robot_localization` để kết hợp dữ liệu IMU (gia tốc góc, gia tốc thẳng) với Wheel Odometry, loại bỏ sai số trượt bánh.
* **SLAM & Nav2:**
  * Quét bản đồ môi trường 2D với Cartographer hoặc Slam-Toolbox.
  * Cấu hình bộ điều hướng Nav2 (Costmap, Global Planner, Local Controller như DWB / MPPI) để robot tự tránh vật cản và bám đường mục tiêu.
