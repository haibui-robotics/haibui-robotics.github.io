---
layout: single
title: "Tutorial 25: Gắn cảm biến IMU (Gia tốc & Con quay hồi chuyển) vào Gazebo và đọc dữ liệu trên ROS 2"
permalink: /posts/2026/11/ros2-tutorial-25/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 24 - Gắn cảm biến Depth Camera (RGB-D)](/posts/2026/11/ros2-tutorial-24/) | [Bài tiếp: Tutorial 27 - Thiết kế & Tích hợp Thế giới ảo (World) ➡](/posts/2026/11/ros2-tutorial-27/)

---

## 1. Mục tiêu và Khái niệm cốt lõi (IMU là gì?)

Nếu Camera và LiDAR đóng vai trò là *"đôi mắt"* giúp robot quan sát thế giới bên ngoài, thì **IMU (Inertial Measurement Unit – Cảm biến đo lường quán tính)** chính là *"hệ thống tiền đình"* bên trong cơ thể robot.

Khi bánh xe robot đi vào vũng dầu hoặc sàn gạch trơn, bánh xe vẫn quay nhưng xe đứng yên (gây ra sai số trượt **Wheel Odometry Drift**). Lúc này, IMU sẽ cho robot biết thực sự thân xe có đang xoay hay tăng tốc hay không.

Một cảm biến IMU tiêu chuẩn trong ROS 2 (gói tin `sensor_msgs/msg/Imu`) cung cấp 3 nhóm dữ liệu chính với tần số rất cao (thường từ **100 Hz – 200 Hz**):

1. **Angular Velocity (Vận tốc góc – $\text{rad/s}$):** Đo bằng **Con quay hồi chuyển (Gyroscope)** quanh 3 trục $X$ (*Roll*), $Y$ (*Pitch*), $Z$ (*Yaw*). Với robot di động trên mặt phẳng, vận tốc góc quanh trục $Z$ ($\omega_z$) là thông số vàng để xác định góc quay của xe.
2. **Linear Acceleration (Gia tốc tuyến tính – $\text{m/s}^2$):** Đo bằng **Gia tốc kế (Accelerometer)** dọc theo 3 trục $X, Y, Z$. Khi xe đứng yên trên mặt phẳng ngang, trục $Z$ vẫn luôn đo được gia tốc trọng trường $g \approx 9.81\text{ m/s}^2$.
3. **Orientation (Góc hướng 3D – Quaternion $x, y, z, w$):** Góc xoay tổng hợp của thân robot trong không gian 3 chiều.

> **Tại sao trong mô phỏng ta phải cố tình thêm Nhiễu (`Gaussian Noise`)?**  
> Ngoài thực tế, không có cảm biến IMU nào (như MPU6050, ICM-20948 hay BNO055) là hoàn hảo; chúng luôn bị nhiễu trắng (*white noise*) và trôi điểm không (*bias*). Việc cấu hình `<noise type="gaussian">` trong Gazebo giúp dữ liệu giả lập sát với phần cứng thật nhất, làm tiền đề để kiểm thử thuật toán lọc **Extended Kalman Filter (EKF)** ở giai đoạn sau.

---

## 2. Viết file `imu.xacro`

Tạo file `urdf/imu.xacro` để khai báo một chip IMU nhỏ (kích thước $2\text{cm} \times 2\text{cm} \times 2\text{cm}$) gắn cố định lên thân xe (`base_link`), kèm theo cấu hình cảm biến `imu` tần số **100 Hz** và nhiễu Gaussian cho cả 3 trục vận tốc góc lẫn gia tốc dài:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <!-- 1. Kích thước khối chip IMU (2cm x 2cm x 2cm) -->
    <xacro:property name="imu_size" value="0.02" />

    <link name="imu_link">
        <visual>
            <geometry>
                <box size="${imu_size} ${imu_size} ${imu_size}"/>
            </geometry>
            <material name="blue" />
        </visual>
        <collision>
            <geometry>
                <box size="${imu_size} ${imu_size} ${imu_size}"/>
            </geometry>
        </collision>
        <xacro:box_inertia m="0.1" x="${imu_size}" y="${imu_size}" z="${imu_size}"
                           o_xyz="0 0 0" o_rpy="0 0 0" />
    </link>

    <!-- 2. Gắn cố định IMU lên thân xe (base_link) -->
    <joint name="imu_joint" type="fixed">
        <parent link="base_link" />
        <child link="imu_link" />
        <origin xyz="0.1 0 0.2" rpy="0 0 0" />
    </joint>

    <!-- 3. Cấu hình cảm biến IMU trong Gazebo (Tần số 100 Hz + Nhiễu Gaussian) -->
    <gazebo reference="imu_link">
        <sensor name="imu" type="imu">
            <always_on>true</always_on>
            <update_rate>100</update_rate>
            <gz_frame_id>imu_link</gz_frame_id>
            <topic>imu</topic>
            <imu>
                <!-- Nhiễu vận tốc góc (Gyroscope - rad/s) -->
                <angular_velocity>
                    <x>
                        <noise type="gaussian">
                            <mean>0.0</mean>
                            <stddev>2e-4</stddev>
                        </noise>
                    </x>
                    <y>
                        <noise type="gaussian">
                            <mean>0.0</mean>
                            <stddev>2e-4</stddev>
                        </noise>
                    </y>
                    <z>
                        <noise type="gaussian">
                            <mean>0.0</mean>
                            <stddev>2e-4</stddev>
                        </noise>
                    </z>
                </angular_velocity>

                <!-- Nhiễu gia tốc tuyến tính (Accelerometer - m/s^2) -->
                <linear_acceleration>
                    <x>
                        <noise type="gaussian">
                            <mean>0.0</mean>
                            <stddev>1.7e-2</stddev>
                        </noise>
                    </x>
                    <y>
                        <noise type="gaussian">
                            <mean>0.0</mean>
                            <stddev>1.7e-2</stddev>
                        </noise>
                    </y>
                    <z>
                        <noise type="gaussian">
                            <mean>0.0</mean>
                            <stddev>1.7e-2</stddev>
                        </noise>
                    </z>
                </linear_acceleration>
            </imu>
        </sensor>
    </gazebo>

</robot>
```

### Lưu ý quan trọng để Gazebo kích hoạt cảm biến IMU:
1. Mở file gốc `my_robot.urdf.xacro` và nhúng file IMU vào:
   ```xml
   <xacro:include filename="imu.xacro" />
   ```
2. Đảm bảo trong file plugin Gazebo của robot (ví dụ `mobile_base.gazebo.xacro` hoặc trong file `.world`) đã nạp hệ thống xử lý IMU của Gazebo Sim:
   ```xml
   <gazebo>
       <plugin filename="gz-sim-imu-system" name="gz::sim::systems::Imu"/>
   </gazebo>
   ```

---

## 3. Cập nhật Cầu nối `ros_gz_bridge`

Gazebo phát dữ liệu IMU lên topic nội bộ `imu` với kiểu `gz.msg.IMU`. Hãy mở file `config/gazebo_bridge.yaml` và thêm cấu hình ánh xạ sang chuẩn `sensor_msgs/msg/Imu` của ROS 2:

```yaml
# ... (Các cấu hình cmd_vel, tf, joint_states, scan, camera đã có) ...

# Cầu nối dữ liệu cảm biến quán tính IMU
- ros_topic_name: "/imu"
  gz_topic_name: "/imu"
  ros_type_name: "sensor_msgs/msg/Imu"
  gz_type_name: "gz.msg.IMU"
  direction: GZ_TO_ROS
```

---

## 4. Kiểm tra dữ liệu IMU trên Terminal & RViz2

1. **Build và khởi chạy lại mô phỏng:**
   ```bash
   colcon build
   source install/setup.bash
   ros2 launch my_mobilerobot gazebo.launch.xml
   ```
2. **Kiểm tra tần số và nội dung gói tin IMU qua Terminal:**
   * Kiểm tra tốc độ phát (kỳ vọng ~100 Hz):
     ```bash
     ros2 topic hz /imu
     ```
   * Xem trực tiếp dữ liệu góc, vận tốc góc và gia tốc:
     ```bash
     ros2 topic echo /imu
     ```
   * Bạn sẽ thấy khi xe đứng yên, `linear_acceleration.z` dao động nhẹ quanh mức `9.80` – `9.82` (nhờ lớp nhiễu Gaussian `stddev = 1.7e-2` mà ta đã cài đặt), và khi bạn điều khiển xe xoay tròn, giá trị `angular_velocity.z` sẽ thay đổi rõ rệt.

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 24 - Gắn cảm biến Depth Camera (RGB-D)](/posts/2026/11/ros2-tutorial-24/) | [Bài tiếp: Tutorial 27 - Thiết kế & Tích hợp Thế giới ảo (World) ➡](/posts/2026/11/ros2-tutorial-27/)
