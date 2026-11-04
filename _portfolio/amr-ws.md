---
title: "Dự án Xe Tự Hành AMR (amr_ws) – Từ Mô phỏng Gazebo đến Phần cứng Thực tế với ROS 2"
excerpt: "Xây dựng hệ thống Robot di động tự hành (AMR) hoàn chỉnh trên ROS 2: Thiết kế 3D URDF/Meshes, điều khiển động học với ros2_control, viết Hardware Interface giao tiếp vi điều khiển (PID + Encoder + IMU MPU6050) và định vị EKF Sensor Fusion."
collection: portfolio
permalink: /portfolio/amr-ws/
---

## 1. Giới thiệu Dự án (`amr_ws`)

**`amr_ws` (Autonomous Mobile Robot Workspace)** là dự án phát triển hệ thống xe tự hành vi sai (Differential Drive AMR) toàn diện từ con số 0 trên nền tảng **ROS 2 (C++ & Python)**.

Điểm đặc biệt của dự án là tính **đồng nhất giữa Mô phỏng (Simulation) và Thực tế (Real Robot)** thông qua kiến trúc **`ros2_control`**: cùng một bộ thuật toán điều khiển và định vị có thể chạy mượt mà cả trên môi trường ảo **Gazebo** lẫn trên phần cứng thực tế (Máy tính nhúng giao tiếp Vi điều khiển điều khiển động cơ DC Encoder và cảm biến IMU MPU6050).

* 🔗 **Mã nguồn GitHub (`amr_ws`):** [github.com/haibui-robotics](https://github.com/haibui-robotics) *(Đang cập nhật từng module)*
* 📺 **Video Demo & Hướng dẫn:** [Kênh YouTube Hải Bùi](https://www.youtube.com/@HaiBui-tl7gq)

---

## 2. Cấu trúc tổng thể Workspace (`amr_ws`)

Toàn bộ dự án được chia thành **8 ROS 2 Packages** chuyên biệt theo đúng chuẩn công nghiệp:

```text
amr_ws/
└── src/
    ├── amr_bringup/                     # [1] Gói khởi chạy tổng hợp (Simulation & Real Robot)
    │   ├── CMakeLists.txt
    │   ├── package.xml
    │   └── launch/
    │       ├── simulated_robot.launch.py
    │       └── real_robot.launch.py
    │
    ├── amr_description/                 # [2] Thiết kế 3D CAD Meshes, URDF/Xacro & Gazebo
    │   ├── CMakeLists.txt
    │   ├── package.xml
    │   ├── launch/
    │   │   ├── display.launch.py
    │   │   └── gazebo.launch.py
    │   ├── meshes/
    │   │   ├── base_link.STL
    │   │   ├── wheel_left_link.STL
    │   │   ├── wheel_right_link.STL
    │   │   ├── caster_front_link.STL
    │   │   ├── caster_rear_link.STL
    │   │   └── imu_link.STL
    │   ├── rviz/
    │   │   └── display.rviz
    │   └── urdf/
    │       ├── amr.urdf.xacro
    │       ├── amr_gazebo.xacro
    │       └── amr_ros2_control.xacro
    │
    ├── amr_controller/                  # [3] Động học thuận/nghịch, Wheel Odometry & Teleop
    │   ├── CMakeLists.txt
    │   ├── package.xml
    │   ├── config/
    │   │   ├── amr_controllers.yaml
    │   │   ├── joy_config.yaml
    │   │   └── joy_teleop.yaml
    │   ├── include/amr_controller/
    │   ├── src/                         # Phiên bản C++ hiệu năng cao
    │   │   ├── simple_controller.cpp
    │   │   └── noisy_controller.cpp
    │   ├── amr_controller/              # Phiên bản Python
    │   │   ├── simple_controller.py
    │   │   └── noisy_controller.py
    │   └── launch/
    │       ├── controller.launch.py
    │       └── joystick_teleop.launch.py
    │
    ├── amr_firmware/                    # [4] Giao tiếp Nhúng, Hardware Interface & PID Động cơ
    │   ├── CMakeLists.txt
    │   ├── package.xml
    │   ├── amr_interface.xml
    │   ├── include/amr_firmware/
    │   ├── src/
    │   │   ├── amr_interface.cpp        # Custom ros2_control Hardware Interface (C++)
    │   │   ├── simple_serial_receiver.cpp
    │   │   └── simple_serial_transmitter.cpp
    │   ├── amr_firmware/
    │   │   ├── mpu6050_driver.py        # Driver đọc cảm biến IMU MPU6050 qua I2C
    │   │   ├── simple_serial_receiver.py
    │   │   └── simple_serial_transmitter.py
    │   ├── firmware/                    # Code nạp trực tiếp cho Vi điều khiển (MCU)
    │   │   ├── robot_control/           # Vòng kín PID điều khiển 2 động cơ + đọc Encoder
    │   │   ├── robot_control_inverted/
    │   │   ├── simple_encoder_reader/
    │   │   ├── simple_motor_control/
    │   │   ├── simple_serial_receiver/
    │   │   └── simple_serial_transmitter/
    │   └── launch/
    │       └── hardware_interface.launch.py
    │
    ├── amr_localization/                # [5] Định vị & Dung hợp cảm biến (Kalman Filter & EKF)
    │   ├── CMakeLists.txt
    │   ├── package.xml
    │   ├── config/
    │   │   └── ekf.yaml                 # Cấu hình ma trận nhiễu robot_localization
    │   ├── include/amr_localization/
    │   ├── src/
    │   │   ├── kalman_filter.cpp
    │   │   └── imu_republisher.cpp
    │   ├── amr_localization/
    │   │   ├── kalman_filter.py
    │   │   └── imu_republisher.py
    │   └── launch/
    │       └── local_localization.launch.py
    │
    ├── amr_msgs/                        # [6] Định nghĩa Custom Services & Messages
    │   ├── CMakeLists.txt
    │   ├── package.xml
    │   └── srv/
    │       ├── AddTwoInts.srv
    │       └── GetTransform.srv
    │
    ├── amr_cpp_examples/                # [7] Các Node nền tảng viết bằng C++ (rclcpp)
    │   └── src/
    │       ├── simple_publisher.cpp
    │       ├── simple_subscriber.cpp
    │       ├── simple_service_server.cpp
    │       ├── simple_service_client.cpp
    │       ├── simple_parameter.cpp
    │       ├── simple_lifecycle_node.cpp
    │       ├── simple_tf_kinematics.cpp
    │       └── simple_turtlesim_kinematics.cpp
    │
    └── amr_py_examples/                 # [8] Các Node nền tảng viết bằng Python (rclpy)
        └── amr_py_examples/
            ├── simple_publisher.py
            ├── simple_subscriber.py
            ├── simple_service_server.py
            ├── simple_service_client.py
            ├── simple_parameter.py
            ├── simple_lifecycle_node.py
            ├── simple_tf_kinematics.py
            └── simple_turtlesim_kinematics.py
```

---

## 3. Phân tích Chi tiết Từng Phân hệ Kỹ thuật

### 3.1. Mô hình hóa 3D & Vật lý Gazebo (`amr_description`)
* **Thiết kế cơ khí (CAD Meshes):** Sử dụng các file lưới 3D `.STL` xuất từ CAD cho khung xe (`base_link.STL`), 2 bánh chủ động (`wheel_left_link.STL`, `wheel_right_link.STL`), 2 bánh tự lựa (`caster_front_link.STL`, `caster_rear_link.STL`) và cảm biến (`imu_link.STL`).
* **Tách biệt cấu hình Xacro:**
  * `amr.urdf.xacro`: Khai báo cấu trúc hình học, khối lượng, ma trận quán tính (`inertial`) và khung va chạm (`collision`).
  * `amr_gazebo.xacro`: Cấu hình hệ số ma sát và gắn các plugin vật lý Gazebo (bao gồm cảm biến IMU giả lập).
  * `amr_ros2_control.xacro`: Định nghĩa giao diện phần cứng (`command_interface` vận tốc cho 2 bánh xe và `state_interface` vị trí/vận tốc từ Encoder), tự động chuyển đổi giữa plugin mô phỏng Gazebo và driver phần cứng thật.

---

### 3.2. Động học, Odometry & Điều khiển (`amr_controller`)
* **Động học xe vi sai (Differential Drive Kinematics):**
  * **Động học nghịch (Inverse Kinematics):** Chuyển đổi lệnh vận tốc tổng hợp của xe $(v, \omega)$ từ Topic `/cmd_vel` sang vận tốc góc mục tiêu của bánh trái và bánh phải $(\omega_L, \omega_R)$ theo công thức:
    $$\begin{bmatrix} \omega_R \\ \omega_L \end{bmatrix} = \begin{bmatrix} \frac{1}{r} & \frac{L}{2r} \\ \frac{1}{r} & -\frac{1}{2r}L \end{bmatrix} \begin{bmatrix} v \\ \omega \end{bmatrix}$$
  * **Động học thuận & Wheel Odometry (`simple_controller`):** Đọc góc quay thực tế từ `/joint_states`, tích phân vận tốc để ước tính vị trí $(x, y, \theta)$ của robot trong hệ trục `odom` và phát đồng thời gói tin `nav_msgs/msg/Odometry` cùng biến đổi động TF `odom` → `base_footprint`.
* **Mô phỏng sai số thực tế (`noisy_controller`):** Cộng thêm nhiễu Gaussian vào số đo góc quay bánh xe để tái hiện hiện tượng trượt bánh (**Wheel Slip / Drift**), phục vụ việc kiểm chứng hiệu quả của bộ lọc Kalman.
* **Điều khiển tay cầm (`joystick_teleop.launch.py`):** Cho phép điều khiển robot mượt mà qua tay cầm không dây hoặc bàn phím.

---

### 3.3. Giao tiếp Nhúng & Điều khiển Động cơ (`amr_firmware`)
* **Custom `ros2_control` Hardware Interface (`amr_interface.cpp`):**
  * Viết bằng C++ kế thừa từ `hardware_interface::SystemInterface` và xuất plugin qua `pluginlib`.
  * Trong mỗi chu kỳ vòng lặp điều khiển (`read()` / `write()`), hệ thống gửi lệnh vận tốc góc mong muốn của 2 bánh xe xuống Vi điều khiển qua chuẩn **UART/Serial** (`libserial`) và đọc ngược về vận tốc thực tế đo từ Encoder.
* **Firmware trên Vi điều khiển (`firmware/robot_control`):**
  * Đọc xung vuông hai kênh A/B từ **Quadrature Encoder** bằng ngắt ngoài (External Interrupt) để xác định chiều quay và vận tốc góc thực tế (rad/s).
  * Thực thi thuật toán **PID vòng kín** độc lập cho từng bánh xe, điều chế độ rộng xung **PWM** cấp cho mạch cầu H (Motor Driver) để bám sát vận tốc mục tiêu từ ROS 2.
* **Driver IMU thực tế (`mpu6050_driver.py`):** Giao tiếp trực tiếp với cảm biến **MPU6050** qua bus **I2C** trên máy tính nhúng, đọc thanh ghi gia tốc góc và gia tốc dài, quy đổi đơn vị và phát lên topic `/imu/out`.

---

### 3.4. Định vị & Dung hợp Cảm biến (`amr_localization`)
* **Lọc Kalman 1 chiều (`kalman_filter.cpp` / `.py`):** Minh họa trực quan nguyên lý toán học của bộ lọc Kalman (hai bước *Predict* & *Measurement Update*) khi kết hợp vận tốc góc từ bánh xe bị nhiễu với vận tốc góc từ Gyroscope của IMU.
* **Extended Kalman Filter (`ekf.yaml` – `robot_localization`):**
  * Kết hợp dữ liệu Odometry bị trượt (`odom_noisy`) với dữ liệu góc xoay và vận tốc góc từ cảm biến IMU (`imu_republisher`).
  * Xuất ra quỹ đạo đã lọc nhiễu `/odometry/filtered` và cây tọa độ TF chính xác cao, khắc phục triệt để sai số tích lũy khi robot xoay hoặc đi trên địa hình trơn.

---

### 3.5. Khởi chạy Tích hợp (`amr_bringup`)
Chỉ với một lệnh duy nhất, toàn bộ hệ thống sẽ được khởi chạy đồng bộ:

* **Chế độ Mô phỏng trên Gazebo:**
  ```bash
  colcon build
  source install/setup.bash
  ros2 launch amr_bringup simulated_robot.launch.py
  ```
* **Chế độ Chạy trên Robot Thực tế:**
  ```bash
  ros2 launch amr_bringup real_robot.launch.py
  ```

---

## 4. Lộ trình Phát triển & Cập nhật Code

Dự án đang được hoàn thiện và cập nhật mã nguồn theo từng giai đoạn:
- [x] **Giai đoạn 1:** Xây dựng các gói nền tảng C++/Python (`amr_cpp_examples`, `amr_py_examples`, `amr_msgs`).
- [x] **Giai đoạn 2:** Thiết kế mô hình 3D URDF/Xacro với CAD Meshes và mô phỏng trên Gazebo (`amr_description`).
- [x] **Giai đoạn 3:** Lập trình bộ điều khiển động học, tính toán Odometry và tích hợp `ros2_control` (`amr_controller`).
- [x] **Giai đoạn 4:** Thiết kế bộ lọc Kalman & EKF Sensor Fusion (`amr_localization`).
- [ ] **Giai đoạn 5 (Đang cập nhật lên Web):** Đưa chi tiết mã nguồn Firmware PID, `amr_interface.cpp` và tích hợp SLAM/Nav2 tự hành hoàn toàn.
