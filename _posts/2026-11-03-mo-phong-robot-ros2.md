---
title: 'Mô phỏng Robot với ROS 2 (TF2 - URDF - RViz2 - Gazebo)'
date: 2026-11-03
permalink: /posts/2026/11/mo-phong-robot-ros2/
tags:
  - ROS 2
  - Simulation
  - URDF
  - Xacro
  - CAD to URDF
  - TF2
  - RViz2
  - Gazebo
---

Bài viết giới thiệu nền tảng mô phỏng Robot toàn diện trong ROS 2: từ thiết kế cấu trúc 3D (`URDF`/`Xacro`), chuyển đổi bản vẽ cơ khí 3D (`CAD to URDF`), quản lý hệ trục tọa độ (`TF2`), trực quan hóa dữ liệu (`RViz2`) cho đến mô phỏng vật lý và cảm biến (`LiDAR`, `Camera`, `Depth Camera`, `IMU`) trong môi trường ảo (`Gazebo Sim`).

---

## 1. Mục tiêu

Học cách xây dựng mô hình 3D của robot (từ các khối hình học cơ bản đến xuất từ phần mềm CAD như SolidWorks/Fusion 360), tích hợp đầy đủ hệ thống cảm biến ảo (LiDAR, Camera, IMU), trực quan hóa dữ liệu trên RViz2 và đưa robot vào môi trường vật lý ảo Gazebo để kiểm thử thuật toán trước khi chạy trên phần cứng thực tế.

---

## 2. Các khái niệm cốt lõi (Kiến thức cần học)

* **TF2 (Transform): Hệ trục tọa độ**
  * Mỗi bộ phận trên robot (tâm robot, bánh xe, cảm biến LiDAR, camera, IMU) đều có một hệ trục tọa độ riêng.
  * TF giúp ROS 2 tính toán khoảng cách và góc xoay tương đối giữa các bộ phận này *(Ví dụ: tính chính xác LiDAR đang nằm cách tâm bánh xe bao nhiêu cm)*. Tất cả tạo thành một **TF Tree (Cây tọa độ)**.

* **URDF (Unified Robot Description Format): Bản thiết kế Robot**
  * Là một file định dạng XML dùng để khai báo hình dáng và cấu trúc của robot.
  * Gồm 2 thành phần chính:
    * **Link:** Các khối vật lý cứng (ví dụ: thân xe, bánh xe, cánh tay).
    * **Joint:** Khớp nối giữa các Link (ví dụ: khớp xoay của bánh xe, khớp cố định của LiDAR).

* **Xacro & CAD to URDF (Meshes `.stl` / `.dae`):**
  * **Xacro:** Cho phép lập trình trong file XML bằng cách dùng biến (*properties*), hàm (*macros*) và tách nhỏ file (*include*), giúp cấu trúc file mô phỏng gọn gàng và dễ bảo trì.
  * **CAD to URDF:** Với các robot thực tế có hình dáng cơ khí phức tạp, ta thiết kế trên SolidWorks / Fusion 360 / Onshape rồi xuất sang các file lưới 3D (`.stl`, `.dae`) cùng thông số khối lượng, tâm quán tính chính xác để nhúng vào URDF/Xacro.

* **RViz2: Công cụ Trực quan hóa (Visualization)**
  * Là nơi để bạn *"nhìn bằng con mắt của robot"*.
  * RViz2 **không** mô phỏng vật lý (không có trọng lực hay ma sát). Nó chỉ dùng để hiển thị dữ liệu thô: mô hình 3D, tọa độ TF, điểm quét LiDAR (`LaserScan`), đám mây điểm 3D (`PointCloud2`), hình ảnh Camera mà hệ thống ROS 2 đang nhận được.

* **Gazebo Sim & Hệ thống Plugin Cảm biến (Simulation):**
  * Thế giới ảo giống như ngoài đời thật, có trọng lực, ma sát, quán tính, va chạm và ánh sáng *(với ROS 2 Jazzy là Gazebo Sim / Harmonic)*.
  * Thông qua các **Gazebo Plugins** và cầu nối **`ros_gz_bridge`**, bạn có thể gắn động cơ ảo (`DiffDrive`), cảm biến quét laser (`gpu_lidar`), Camera 2D & Depth Camera (`camera` / `rgbd_camera`), cảm biến gia tốc & góc nghiêng (`imu`) cũng như tự thiết kế môi trường thử nghiệm (`.world` / `.sdf`).

---

## 3. Danh sách Thực hành (Tutorials)

*(Nhấn vào từng bài để xem hướng dẫn chi tiết, source code và video YouTube)*

### Phần A: Hệ trục tọa độ TF2, URDF, Xacro & Hiển thị RViz2
1. [Tutorial 15: Khái niệm TF2 và cách dùng lệnh Terminal để xem cây tọa độ (TF Tree)](/posts/2026/11/ros2-tutorial-15/)
2. [Tutorial 16: Viết bản thiết kế URDF đầu tiên (Khai báo Base & Wheels)](/posts/2026/11/ros2-tutorial-16/)
3. [Tutorial 17: Viết code thông minh hơn với Xacro (Biến, Hàm và Module hóa)](/posts/2026/11/ros2-tutorial-17/)
4. [Tutorial 18: Hiển thị robot trên RViz2 (Sử dụng robot_state_publisher và joint_state_publisher_gui)](/posts/2026/11/ros2-tutorial-18/)

### Phần B: Đưa Robot vào môi trường vật lý Gazebo Sim
5. [Tutorial 19: Chuẩn bị cho Gazebo: Khai báo thuộc tính vật lý (Khối lượng - Inertia & Khung va chạm - Collision)](/posts/2026/11/ros2-tutorial-19/)
6. [Tutorial 20: Cài đặt Gazebo Sim, World và "Spawn" (thả) robot vào thế giới ảo](/posts/2026/11/ros2-tutorial-20/)
7. [Tutorial 21: Làm cho robot di chuyển trong Gazebo (Plugins & ros_gz_bridge)](/posts/2026/11/ros2-tutorial-21/)

### Phần C: Tích hợp Cảm biến ảo (LiDAR, Camera, Depth Camera, IMU)
8. [Tutorial 22: Gắn cảm biến LiDAR 2D vào Gazebo và hiển thị trên RViz2](/posts/2026/11/ros2-tutorial-22/)
9. [Tutorial 23: Gắn cảm biến Camera 2D và bí ẩn của "Optical Frame"](/posts/2026/11/ros2-tutorial-23/)
10. [Tutorial 24: Gắn cảm biến Depth Camera (RGB-D) và hiển thị đám mây điểm 3D (PointCloud2) trên RViz2](/posts/2026/11/ros2-tutorial-24/)
11. [Tutorial 25: Gắn cảm biến IMU (Gia tốc & Con quay hồi chuyển) vào Gazebo và đọc dữ liệu trên ROS 2](/posts/2026/11/ros2-tutorial-25/)

### Phần D: Nâng cao – Chuyển đổi CAD sang URDF & Thiết kế World tùy chỉnh
12. [Tutorial 26: Từ CAD sang URDF: Xuất mô hình 3D từ SolidWorks / Fusion 360 (.stl, .dae) và tích hợp vào Xacro](/posts/2026/11/ros2-tutorial-26/)
13. [Tutorial 27: Tự thiết kế môi trường mô phỏng (Custom World .world/.sdf) với Gazebo Building Editor](/posts/2026/11/ros2-tutorial-27/)
