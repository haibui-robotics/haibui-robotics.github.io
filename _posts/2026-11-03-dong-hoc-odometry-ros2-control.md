---
title: 'Động học, Odometry và Điều khiển Robot (ros2_control)'
date: 2026-11-03
permalink: /posts/2026/11/dong-hoc-odometry-ros2-control/
tags:
  - ROS 2
  - Kinematics
  - Odometry
  - ros2_control
  - EKF
  - Sensor Fusion
---

Bài viết đi sâu vào cơ chế chuyển đổi lệnh điều khiển thành chuyển động bánh xe (**Kinematics**), ước tính vị trí robot (**Odometry**), kiến trúc điều khiển chuẩn **ros2_control** và thuật toán dung hợp cảm biến **Extended Kalman Filter (EKF)**.

---

## 1. Mục tiêu

Hiểu cách biến các lệnh điều khiển (như tiến, lùi, xoay) thành chuyển động thực tế của bánh xe. Nắm vững cách tính toán vị trí của robot trong không gian dựa trên cảm biến (**Odometry**) và cách xử lý nhiễu bằng thuật toán kết hợp cảm biến (**Sensor Fusion**).

---

## 2. Các khái niệm cốt lõi (Kiến thức cần học)

* **Kinematics (Động học Robot):**
  * Là các phương trình toán học chuyển đổi giữa tốc độ của cả robot (Vận tốc thẳng $v$, vận tốc góc $\omega$) và tốc độ của từng bánh xe riêng lẻ.
  * Mỗi loại khung gầm có bộ phương trình khác nhau *(Ví dụ: Xe 2 bánh độc lập - Differential Drive, Xe 3/4 bánh đa hướng - Omni/Mecanum, Xe lái như ô tô - Ackermann)*.

* **Topic `/cmd_vel` và Message `Twist`:**
  * Đây là tiêu chuẩn của ROS 2 để ra lệnh di chuyển cho mọi loại robot. `Twist` chứa dữ liệu về vận tốc tuyến tính (`Linear` - $x, y, z$) và vận tốc góc (`Angular` - $x, y, z$).

* **Odometry (Đo đạc vị trí / Nội suy chuyển động):**
  * Là quá trình ước tính vị trí hiện tại của robot ($x, y$, góc $\text{yaw}$) dựa trên dữ liệu trả về từ cảm biến vận tốc (như Encoder ở bánh xe).
  * **Nhược điểm chí mạng:** Odometry bằng bánh xe luôn bị sai số tích lũy (**Drift**) do bánh xe bị trượt, ma sát, bề mặt gồ ghề. Càng đi xa, robot tính toán vị trí càng sai.

* **`ros2_control` Framework:**
  * Là một hệ thống chuẩn của ROS 2 giúp tách biệt giữa phần cứng (*Hardware*) và phần mềm điều khiển (*Controller*).
  * Thay vì tự viết code tính toán động học và giao tiếp phần cứng thủ công, bạn chỉ cần nạp cấu hình, `ros2_control` sẽ tự động dịch lệnh `/cmd_vel` thành tốc độ bánh xe.

* **Sensor Fusion (Kết hợp cảm biến) & EKF:**
  * Để khắc phục điểm yếu của Odometry bánh xe, ta phải kết hợp nó với cảm biến gia tốc/góc nghiêng (**IMU**).
  * **Extended Kalman Filter (EKF)** là một thuật toán toán học giúp dung hợp dữ liệu từ nhiều cảm biến ($\text{Encoder} + \text{IMU}$), lọc nhiễu và đưa ra một Odometry mới chính xác, mượt mà hơn rất nhiều. Trong ROS 2, gói `robot_localization` đảm nhận việc này.

---

## 3. Danh sách Thực hành (Tutorials)

*(Nhấn vào từng bài để xem hướng dẫn chi tiết, source code và video YouTube)*

1. [Tutorial 24: Giới thiệu tiêu chuẩn điều khiển: Topic /cmd_vel, gói tin Twist và cách dùng teleop_twist_keyboard để lái robot bằng bàn phím](/posts/2026/11/ros2-tutorial-24/)
2. [Tutorial 25: Động học xe 2 bánh (Differential Drive): Bản chất toán học đằng sau chuyển động](/posts/2026/11/ros2-tutorial-25/)
3. [Tutorial 26: Giới thiệu ros2_control và cách tích hợp diff_drive_controller vào robot mô phỏng Gazebo](/posts/2026/11/ros2-tutorial-26/)
4. [Tutorial 27: Xuất dữ liệu Odometry từ bánh xe và hiển thị quỹ đạo di chuyển (Trajectory) của robot trên màn hình RViz2](/posts/2026/11/ros2-tutorial-27/)
5. [Tutorial 28: Khái niệm "Drift" (Trượt): Trực tiếp quan sát sai số của Odometry trên RViz2 khi robot xoay tròn nhiều vòng](/posts/2026/11/ros2-tutorial-28/)
6. [Tutorial 29: Giới thiệu cảm biến IMU: Khai báo plugin IMU trong Gazebo và đọc dữ liệu gia tốc, góc xoay](/posts/2026/11/ros2-tutorial-29/)
7. [Tutorial 30: Kết hợp cảm biến (Sensor Fusion): Cài đặt và cấu hình node ekf_node từ gói robot_localization](/posts/2026/11/ros2-tutorial-30/)
8. [Tutorial 31: Dung hợp Encoder và IMU: Cấu hình ma trận EKF để tạo ra Topic /odometry/filtered chính xác tuyệt đối](/posts/2026/11/ros2-tutorial-31/)
9. [(Tùy chọn nâng cao nếu bạn dùng Omni) Tutorial 32: Cấu hình động học và điều khiển cho robot 3 bánh đa hướng (Kiwi Drive / Omni Drive)](/posts/2026/11/ros2-tutorial-32/)
