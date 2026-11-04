---
layout: single
title: "Tutorial 22: Gắn cảm biến LiDAR 2D vào Gazebo và hiển thị trên RViz2"
permalink: /posts/2026/11/ros2-tutorial-22/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 21 - Làm cho robot di chuyển trong Gazebo](/posts/2026/11/ros2-tutorial-21/)

---

## 1. Mục tiêu và Khái niệm

**LiDAR 2D** là một trong những cảm biến quan trọng nhất để robot định vị và tránh vật cản. Nó phát ra các tia laser xung quanh mặt phẳng ngang và đo thời gian tia sáng phản xạ lại để tính toán khoảng cách.

Trong bài này, chúng ta sẽ thực hiện 3 việc:

1. **Thiết kế LiDAR (URDF):** Tạo khối trụ đại diện cho LiDAR và gắn lên đỉnh xe.
2. **Mô phỏng tia quét (Gazebo Plugin):** Gắn module giả lập cảm biến `gpu_lidar` vào Gazebo để sinh ra dữ liệu đo đạc (điểm ảnh).
3. **Trực quan hóa vùng quét (Tùy chọn):** Vẽ một vòng tròn đỏ bán trong suốt quanh xe để dễ quan sát phạm vi hoạt động 5m của LiDAR trên mô phỏng.

---

## 2. Viết file `lidar.xacro`

Tạo một file mới tên là `lidar.xacro` và định nghĩa cấu trúc của cảm biến:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <!-- 1. Định nghĩa khối trụ đại diện cho cảm biến Lidar -->
    <xacro:property name="lidar_radius" value="0.04" />
    <xacro:property name="lidar_length" value="0.03" />

    <link name="lidar_link">
        <visual>
            <geometry>
                 <cylinder radius="${lidar_radius}" length="${lidar_length}"/>
            </geometry>
            <material name="grey" />
        </visual>
        <collision>
            <geometry>
                <cylinder radius="${lidar_radius}" length="${lidar_length}" />
            </geometry>
        </collision>
        <!-- Khối lượng rất nhẹ cho Lidar -->
        <xacro:cylinder_inertia m="0.1" r="${lidar_radius}" h="${lidar_length}"
                           o_xyz="0 0 0" o_rpy="0 0 0" />
    </link>

    <!-- Khớp nối cố định Lidar vào giữa thân xe (base_link), nhô cao 0.2m -->
    <joint name="base_lidar_joint" type="fixed">
        <parent link="base_link" />
        <child link="lidar_link" />
        <origin xyz="0 0 0.2" rpy="0 0 0" />
    </joint>

    <!-- 2. (Trang trí) Vòng tròn đỏ bán trong suốt hiển thị vùng quét 5m -->
    <xacro:property name="lidar_zone_radius" value="5.0" />
    <xacro:property name="lidar_zone_thickness" value="0.002" />

    <material name="red_transparent">
        <color rgba="1.0 0.0 0.0 0.15" />
    </material>

    <link name="lidar_zone_link">
        <visual>
            <geometry>
                <cylinder radius="${lidar_zone_radius}" length="${lidar_zone_thickness}" />
            </geometry>
            <material name="red_transparent" />
        </visual>
        <!-- Gazebo yêu cầu mọi Link phải có khối lượng, nên ta cho số cực nhỏ -->
        <inertial>
            <mass value="0.001" />
            <inertia ixx="1e-7" ixy="0" ixz="0" iyy="1e-7" iyz="0" izz="1e-7" />
        </inertial>
    </link>

    <joint name="lidar_zone_joint" type="fixed">
        <parent link="lidar_link" />
        <child link="lidar_zone_link" />
        <origin xyz="0 0 0" rpy="0 0 0" />
    </joint>

    <!-- Định nghĩa màu bán trong suốt cho Gazebo (Vì Gazebo không đọc thẻ <material> của URDF) -->
    <gazebo reference="lidar_zone_link">
        <visual>
            <material>
                <!-- Tham số Alpha (số cuối cùng) là 0.15 để tạo độ trong suốt -->
                <ambient>1.0 0.0 0.0 0.15</ambient>
                <diffuse>1.0 0.0 0.0 0.15</diffuse>
                <specular>0.1 0.1 0.1 1.0</specular>
            </material>
        </visual>
    </gazebo>

    <!-- 3. Gắn Plugin Cảm biến Lidar ảo (gpu_lidar) vào Gazebo -->
    <gazebo reference="lidar_link">
        <sensor name="gpu_lidar" type="gpu_lidar">
            <!-- Tốc độ quét: 10 Hz (10 vòng/giây) -->
            <update_rate>10</update_rate>
            <!-- Phát dữ liệu đo đạc lên Topic tên là 'scan' -->
            <topic>scan</topic>
            <gz_frame_id>lidar_link</gz_frame_id>
            <lidar>
                <scan>
                    <horizontal>
                        <samples>360</samples>          <!-- 360 điểm ảnh mỗi vòng -->
                        <resolution>1</resolution>  
                        <min_angle>-3.14159</min_angle> <!-- Quét từ -180 độ... -->
                        <max_angle>3.14159</max_angle>  <!-- ...đến +180 độ (Tổng 360 độ) -->
                    </horizontal>
                </scan>
                <range>
                    <min>0.15</min>        <!-- Khoảng cách gần nhất phát hiện được (Điểm mù) -->
                    <max>5.0</max>         <!-- Khoảng cách xa nhất (5 mét) -->
                    <resolution>0.01</resolution> <!-- Sai số độ phân giải -->
                </range>
                <!-- Thêm nhiễu Gauss (Gaussian noise) để mô phỏng sự thiếu hoàn hảo của cảm biến thật -->
                <noise>
                    <type>gaussian</type>
                    <mean>0.0</mean>
                    <stddev>0.01</stddev> <!-- Độ lệch chuẩn 1cm -->
                </noise>
            </lidar>
            <always_on>true</always_on>
            <!-- Hiển thị tia laser màu xanh trong Gazebo (để dễ debug) -->
            <visualize>true</visualize>
        </sensor>
    </gazebo>

</robot>
```

*(Hãy nhớ mở file `my_robot.urdf.xacro` gốc ra và gọi `<xacro:include filename="lidar.xacro"/>` vào nhé!)*

---

## 3. Cập nhật Bridge để truyền dữ liệu LiDAR

Gazebo đang phát kết quả quét lên topic có tên `/scan` (với kiểu dữ liệu `gz.msg.LaserScan`). Chúng ta cần mở file `config/gazebo_bridge.yaml` đã tạo ở [Tutorial 21](/posts/2026/11/ros2-tutorial-21/), và bổ sung thêm đoạn cầu nối này để đẩy dữ liệu về ROS 2:

```yaml
# ... (Các cấu hình cmd_vel, tf, joint_states đã có) ...

# Cầu nối dữ liệu Lidar (LaserScan)
- ros_topic_name: "/scan"
  gz_topic_name: "/scan"
  ros_type_name: "sensor_msgs/msg/LaserScan"
  gz_type_name: "gz.msg.LaserScan"
  direction: GZ_TO_ROS
```

---

## 4. Hiển thị LaserScan trên RViz2

1. **Khởi chạy lại hệ thống:**
   ```bash
   ros2 launch my_mobilerobot gazebo.launch.xml
   ```
2. Lúc này, trong Gazebo bạn sẽ thấy một vòng tròn đỏ lớn 5m bao quanh xe, kèm theo các tia sáng màu xanh lam nếu chúng quét trúng tường/vật cản.
3. **Để xem điểm quét trên RViz2:**
   * Nhấn nút **Add** ở góc dưới bên trái.
   * Chọn mục **By Topic**.
   * Chọn Topic `/scan` (Loại **LaserScan**).
   * *(Mẹo)* Bạn có thể thay đổi mục **Size** (trong thẻ `LaserScan` bên cột trái) lên mức `0.05m` để các điểm hiển thị to và rõ ràng hơn.

Bây giờ, hãy mở Terminal điều khiển phím (`teleop`), lái xe đi vòng quanh nhà kho và quan sát các chấm đỏ trên RViz thay đổi hình dáng khớp với các bức tường nhé!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 21 - Làm cho robot di chuyển trong Gazebo](/posts/2026/11/ros2-tutorial-21/)
