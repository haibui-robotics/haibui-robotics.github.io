---
title: 'Xây dựng bản đồ (Mapping) và Định vị toàn cục (Localization)'
date: 2026-11-03
permalink: /posts/2026/11/mapping-va-localization-ros2/
tags:
  - ROS 2
  - SLAM
  - slam_toolbox
  - Localization
  - AMCL
  - Mapping
  - Navigation
---

Bài viết trình bày cách robot tự động khám phá môi trường, xây dựng bản đồ 2D (**Occupancy Grid Map / SLAM**) với `slam_toolbox` và định vị toàn cục (**Localization**) bằng thuật toán lọc hạt **AMCL**.

---

## 1. Mục tiêu

Học cách robot tự động khám phá và vẽ lại bản đồ của một môi trường chưa biết. Sau khi có bản đồ, học cách robot sử dụng bản đồ đó kết hợp với cảm biến (**LiDAR**) để luôn biết chính xác mình đang đứng ở đâu, ngay cả khi bị nhấc bổng hoặc đi lạc.

---

## 2. Các khái niệm cốt lõi (Kiến thức cần học)

* **Occupancy Grid Map (Bản đồ lưới chiếm dụng):**
  * Trong ROS 2, bản đồ không phải là hình ảnh JPG thông thường mà là một ma trận điểm ảnh (*Grid*).
  * Mỗi điểm ảnh (*pixel*) mang 1 trong 3 giá trị: **Trắng** (Không gian trống, đi qua được), **Đen** (Vật cản, tường), **Xám** (Chưa khám phá).
  * Dữ liệu bản đồ được phát trên Topic `/map` và lưu dưới dạng 2 file: file hình ảnh (`.pgm`/`.png`) và file cấu hình (`.yaml`).

* **SLAM (Simultaneous Localization and Mapping - Lập bản đồ và Định vị đồng thời):**
  * Đây là bài toán *"con gà - quả trứng"*. Để vẽ map chính xác, robot cần biết mình đang ở đâu. Để biết mình đang ở đâu, robot cần có map.
  * **SLAM** giải quyết cả hai cùng lúc: Vừa dùng LiDAR để vẽ tường, vừa dùng chính bức tường đó để tinh chỉnh vị trí của robot.
  * Trong ROS 2, gói tiêu chuẩn hiện nay cho SLAM 2D là `slam_toolbox` (thay thế cho `gmapping` của ROS 1).

* **Localization (Định vị) và AMCL:**
  * Khi đã có bản đồ lưu sẵn, robot chuyển sang chế độ Định vị.
  * **AMCL (Adaptive Monte Carlo Localization):** Thuật toán phổ biến nhất. Nó tung ra hàng ngàn *"hạt"* (*particles*) ngẫu nhiên trên bản đồ (mỗi hạt là một giả thuyết về vị trí robot). Khi robot di chuyển và dùng LiDAR quét môi trường, AMCL sẽ loại bỏ dần các hạt sai, giữ lại cụm hạt khớp với bản đồ nhất để chốt vị trí hiện tại.

* **TF Tree trong Navigation (`map` → `odom` → `base_link`):**
  * Ở giai đoạn trước ([Động học & Odometry](/posts/2026/11/dong-hoc-odometry-ros2-control/)), ta có TF `odom` → `base_link` (do cảm biến bánh xe và IMU tính toán, nhưng bị sai số tích lũy).
  * Ở giai đoạn này, **AMCL** sẽ tính toán và tạo ra một TF mới là `map` → `odom`. TF này đóng vai trò như một lực *"kéo"* hệ tọa độ `odom` về đúng vị trí thực tế trên bản đồ, từ đó triệt tiêu hoàn toàn sai số **Drift**.

---

## 3. Danh sách Thực hành (Tutorials)

*(Nhấn vào từng bài để xem hướng dẫn chi tiết, source code và video YouTube)*

1. [Tutorial 33: Bản đồ trong ROS 2 là gì? Phân tích cấu trúc file .yaml và .pgm](/posts/2026/11/ros2-tutorial-33/)
2. [Tutorial 34: Giới thiệu gói slam_toolbox: Cài đặt và cấu hình các thông số cơ bản cho việc vẽ map 2D](/posts/2026/11/ros2-tutorial-34/)
3. [Tutorial 35: Chạy SLAM trong Gazebo: Lái robot đi vòng quanh để tạo bản đồ và hiển thị quá trình vẽ trực tiếp trên RViz2](/posts/2026/11/ros2-tutorial-35/)
4. [Tutorial 36: Lưu bản đồ vĩnh viễn (Map Saver) bằng thư viện nav2_map_server](/posts/2026/11/ros2-tutorial-36/)
5. [Tutorial 37: Giới thiệu Lifecycle Node: Dùng map_server để tải (load) bản đồ đã lưu lên Topic /map](/posts/2026/11/ros2-tutorial-37/)
6. [Tutorial 38: Giới thiệu thuật toán Lọc Hạt (Particle Filter) và bản chất của AMCL](/posts/2026/11/ros2-tutorial-38/)
7. [Tutorial 39: Cấu hình và chạy node amcl để định vị robot](/posts/2026/11/ros2-tutorial-39/)
8. [Tutorial 40: Trực quan hóa đám mây hạt (Particle Cloud) trên RViz2: Cung cấp 2D Pose Estimate (vị trí khởi tạo ban đầu) và xem các hạt hội tụ khi robot di chuyển](/posts/2026/11/ros2-tutorial-40/)
9. [Tutorial 41: Kiểm chứng sự hiệu chỉnh TF: Quan sát cách AMCL tạo ra TF map -> odom để sửa lỗi trượt (drift) của bánh xe](/posts/2026/11/ros2-tutorial-41/)
