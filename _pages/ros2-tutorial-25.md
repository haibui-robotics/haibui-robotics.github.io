---
layout: single
title: "Tutorial 25: Gắn cảm biến IMU (Gia tốc & Con quay hồi chuyển) vào Gazebo và đọc dữ liệu trên ROS 2 (Dự án amr_ws – Commit V3)"
permalink: /posts/2026/11/ros2-tutorial-25/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 24 - Gắn cảm biến Depth Camera (RGB-D)](/posts/2026/11/ros2-tutorial-24/) | [Bài tiếp: Tutorial 27 - Thiết kế & Tích hợp Thế giới ảo (World) ➡](/posts/2026/11/ros2-tutorial-27/)

---

## 1. Mục tiêu và Khái niệm cốt lõi (IMU là gì?)

Nếu Camera và LiDAR đóng vai trò là *"đôi mắt"* giúp robot quan sát thế giới bên ngoài, thì **IMU (Inertial Measurement Unit – Cảm biến đo lường quán tính)** chính là *"hệ thống tiền đình"* bên trong cơ thể robot.

Khi bánh xe robot đi vào mặt sàn trơn và bị trượt (**Wheel Odometry Drift**), IMU sẽ đo trực tiếp vận tốc góc và gia tốc thực tế của thân xe để bộ lọc **EKF (Extended Kalman Filter)** hiệu chỉnh lại vị trí.

Một cảm biến IMU tiêu chuẩn trong ROS 2 (gói tin `sensor_msgs/msg/Imu`) cung cấp 3 nhóm dữ liệu chính ở tần số cao (**100 Hz**):
1. **Angular Velocity (Vận tốc góc – $\text{rad/s}$):** Đo bằng **Con quay hồi chuyển (Gyroscope)** quanh 3 trục $X$ (*Roll*), $Y$ (*Pitch*), $Z$ (*Yaw*).
2. **Linear Acceleration (Gia tốc tuyến tính – $\text{m/s}^2$):** Đo bằng **Gia tốc kế (Accelerometer)** dọc theo 3 trục $X, Y, Z$ (bao gồm gia tốc trọng trường $g \approx 9.8\text{ m/s}^2$ theo trục $Z$).
3. **Orientation (Góc hướng 3D – Quaternion $x, y, z, w$):** Góc xoay tổng hợp của thân robot trong không gian.

---

## 2. Nguồn tra cứu Code mẫu `<sensor>` chuẩn của Gazebo (SDFormat)

* 🔗 **Kho mã nguồn SDFormat:** [https://github.com/gazebosim/sdformat/tree/main](https://github.com/gazebosim/sdformat/tree/main)
* 🔗 **File mẫu tổng hợp tất cả các loại `<sensor>` (`sensors.sdf`):** [https://github.com/gazebosim/sdformat/blob/main/test/sdf/sensors.sdf](https://github.com/gazebosim/sdformat/blob/main/test/sdf/sensors.sdf)

---

## 3. Viết file `urdf/imu.xacro` (Commit V3)

Trong dự án **[`amr_ws`](/portfolio/amr-ws/)** *(commit **`V3-lidar+cam+imu`**)*, khâu `imu_link` (kèm file 3D `imu_link.STL`) và khớp cố định `imu_joint` đã được khai báo sẵn trong `urdf/amr.urdf.xacro`:

```xml
  <!-- Đã có sẵn trong urdf/amr.urdf.xacro -->
  <link name="imu_link">
    ...
    <visual>
      <geometry>
        <mesh filename="package://amr_description/meshes/imu_link.STL" />
      </geometry>
    </visual>
  </link>
  <joint name="imu_joint" type="fixed">
    <origin xyz="-0.014822 0.039175 0.08002" rpy="0 0 0" />
    <parent link="base_link" />
    <child link="imu_link" />
  </joint>
```

Ta chỉ cần tạo file `urdf/imu.xacro` để gắn cảm biến `<sensor name="imu" type="imu">` (tần số `100 Hz` kèm nhiễu Gaussian thực tế cho cả 3 trục vận tốc góc và gia tốc tuyến tính) vào `imu_link`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
  
    <gazebo reference="imu_link">
        <sensor name="imu" type="imu">
            <always_on>true</always_on>
            <update_rate>100</update_rate>
            <gz_frame_id>imu_link</gz_frame_id>
            <topic>imu</topic>
            <imu>
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

Sau đó nhúng `imu.xacro` vào đầu file `urdf/amr.urdf.xacro` (hoàn thiện bộ 4 file `<xacro:include>` của commit `V3`):

```xml
<robot name="amr_description" xmlns:xacro="http://www.ros.org/wiki/xacro">
  <xacro:include filename="$(find amr_description)/urdf/lidar.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/imu.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/camera.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/gazebo_sim_controll.xacro"/>
```

*(Lưu ý: Trong file `worlds/small_warehouse.world` của `amr_ws` đã được khai báo sẵn `<plugin filename="gz-sim-imu-system" name="gz::sim::systems::Imu"/>` để kích hoạt cảm biến IMU này).*

---

## 4. Cập nhật Cầu nối `config/gazebo_bridge.yaml` (Commit V3)

Mở file `config/gazebo_bridge.yaml` và thêm khối cầu nối `/imu` vào cuối file:

```yaml
# ... (Các cấu hình đã có từ bài trước) ...

- ros_topic_name: "/imu"
  gz_topic_name: "/imu"
  ros_type_name: "sensor_msgs/msg/Imu"
  gz_type_name: "gz.msgs.IMU"
  direction: GZ_TO_ROS
```

---

## 5. Kiểm tra dữ liệu IMU trên Terminal

1. **Build và khởi chạy mô phỏng:**
   ```bash
   cd ~/amr_ws
   colcon build --packages-select amr_description
   source install/setup.bash
   ros2 launch amr_description gazebo.launch.xml
   ```
2. **Kiểm tra tần số và gói tin `/imu`:**
   ```bash
   ros2 topic hz /imu      # Kiểm tra tần số phát (~100 Hz)
   ros2 topic echo /imu    # Quan sát dữ liệu vận tốc góc và gia tốc
   ```

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 24 - Gắn cảm biến Depth Camera (RGB-D)](/posts/2026/11/ros2-tutorial-24/) | [Bài tiếp: Tutorial 27 - Thiết kế & Tích hợp Thế giới ảo (World) ➡](/posts/2026/11/ros2-tutorial-27/)
