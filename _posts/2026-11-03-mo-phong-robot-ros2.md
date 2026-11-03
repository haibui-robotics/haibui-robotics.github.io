---
title: 'Mô phỏng Robot với ROS 2 (TF2 - URDF - RViz2 - Gazebo)'
date: 2026-11-03
permalink: /posts/2026/11/mo-phong-robot-ros2/
tags:
  - ROS 2
  - Simulation
  - URDF
  - Xacro
  - TF2
  - RViz2
  - Gazebo
---

Bài viết giới thiệu nền tảng mô phỏng Robot trong ROS 2: từ thiết kế cấu trúc 3D (`URDF`/`Xacro`), quản lý hệ trục tọa độ (`TF2`), trực quan hóa dữ liệu (`RViz2`) cho đến mô phỏng vật lý trong môi trường ảo (`Gazebo Sim`).

---

## 1. Mục tiêu

Học cách xây dựng mô hình 3D của robot, trực quan hóa dữ liệu cảm biến và đưa robot vào môi trường vật lý ảo để kiểm thử thuật toán điều khiển trước khi mang xuống chạy trên phần cứng thực tế.

---

## 2. Các khái niệm cốt lõi (Kiến thức cần học)

* **TF2 (Transform): Hệ trục tọa độ**
  * Mỗi bộ phận trên robot (tâm robot, bánh xe, cảm biến LiDAR, camera) đều có một hệ trục tọa độ riêng.
  * TF giúp ROS 2 tính toán khoảng cách và góc xoay tương đối giữa các bộ phận này *(Ví dụ: tính chính xác LiDAR đang nằm cách tâm bánh xe bao nhiêu cm)*. Tất cả tạo thành một **TF Tree (Cây tọa độ)**.

* **URDF (Unified Robot Description Format): Bản thiết kế Robot**
  * Là một file định dạng XML dùng để khai báo hình dáng và cấu trúc của robot.
  * Gồm 2 thành phần chính:
    * **Link:** Các khối vật lý cứng (ví dụ: thân xe, bánh xe, cánh tay).
    * **Joint:** Khớp nối giữa các Link (ví dụ: khớp xoay của bánh xe, khớp cố định của LiDAR).

* **Xacro: Bản nâng cấp của URDF**
  * Code URDF thuần rất dài. **Xacro** cho phép bạn lập trình trong file XML bằng cách dùng biến (*variables*) và hàm (*macros*), giúp cấu trúc file mô phỏng gọn gàng và dễ chỉnh sửa hơn.

* **RViz2: Công cụ Trực quan hóa (Visualization)**
  * Là nơi để bạn *"nhìn bằng con mắt của robot"*.
  * RViz2 **không** mô phỏng vật lý (không có trọng lực hay ma sát). Nó chỉ dùng để hiển thị dữ liệu thô: mô hình 3D, tọa độ TF, điểm quét LiDAR, hình ảnh Camera mà hệ thống ROS 2 đang nhận được.

* **Gazebo: Công cụ Mô phỏng Vật lý (Simulation)**
  * Thế giới ảo giống như ngoài đời thật, có trọng lực, ma sát, va chạm và ánh sáng. *(Với ROS 2 Jazzy, hệ thống thường dùng bản Gazebo Sim / Harmonic)*.
  * Bạn đưa mô hình URDF vào đây, gắn thêm các plugin (như plugin động cơ, plugin LiDAR) để robot chạy giả lập và tương tác với các vật cản ảo.

---

## 3. Danh sách Thực hành (Tutorials)

*(Nhấn vào từng bài để xem hướng dẫn chi tiết, source code và video YouTube)*

1. [Tutorial 15: Khái niệm TF2 và cách dùng lệnh Terminal để xem cây tọa độ (TF Tree)](/posts/2026/11/ros2-tutorial-15/)
2. [Tutorial 16: Viết bản thiết kế URDF đầu tiên: Khai báo Thân xe (Base) và Bánh xe (Links & Joints)](/posts/2026/11/ros2-tutorial-16/)
3. [Tutorial 17: Viết code thông minh hơn: Chuyển đổi URDF sang Xacro (Sử dụng Macros và Biến)](/posts/2026/11/ros2-tutorial-17/)
4. [Tutorial 18: Hiển thị robot trên RViz2 (Sử dụng robot_state_publisher và joint_state_publisher_gui)](/posts/2026/11/ros2-tutorial-18/)
5. [Tutorial 19: Chuẩn bị cho Gazebo: Khai báo thuộc tính vật lý (Khối lượng - Inertia, Khung va chạm - Collision) cho URDF](/posts/2026/11/ros2-tutorial-19/)
6. [Tutorial 20: Cài đặt Gazebo Sim và "Spawn" (thả) mô hình robot vào thế giới ảo](/posts/2026/11/ros2-tutorial-20/)
7. [Tutorial 21: Làm cho robot di chuyển (Thêm plugin điều khiển động cơ Drive vào Gazebo)](/posts/2026/11/ros2-tutorial-21/)
8. [Tutorial 22: Gắn cảm biến Lidar 2D vào robot mô phỏng và trực quan hóa điểm quét (LaserScan) trên RViz2](/posts/2026/11/ros2-tutorial-22/)
9. [Tutorial 23: Gắn cảm biến Camera (hoặc Depth Camera) và xem hình ảnh trực tiếp](/posts/2026/11/ros2-tutorial-23/)
