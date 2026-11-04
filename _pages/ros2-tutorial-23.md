---
layout: single
title: "Tutorial 23: Gắn cảm biến Camera 2D và bí ẩn của 'Optical Frame'"
permalink: /posts/2026/11/ros2-tutorial-23/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo](/posts/2026/11/ros2-tutorial-22/)

---

## 1. Mục tiêu và Khái niệm cốt lõi (`Optical Frame` là gì?)

Trong bài này, chúng ta sẽ gắn một chiếc **Camera RGB** (tương tự webcam) lên phía trước đầu xe để robot có thể "nhìn" thấy môi trường xung quanh dưới dạng hình ảnh màu.

Khái niệm quan trọng nhất bạn cần nắm khi làm việc với Camera trong ROS 2 là sự khác biệt về **Hệ trục tọa độ (Coordinate Frame)**:

* **Chuẩn của ROS 2 (Robot Frame):** Trục $X$ chỉ về phía trước, Trục $Y$ chỉ sang trái, Trục $Z$ chỉ lên trời.
* **Chuẩn của Xử lý ảnh / OpenCV (Optical Frame):** Trục $Z$ chỉ về chiều sâu (phía trước), Trục $X$ chỉ sang phải, Trục $Y$ chỉ xuống đất.

Nếu chúng ta gắn thẳng ảnh vào chuẩn của ROS, hình ảnh thu được trên RViz2 sẽ bị méo hoặc xoay lộn ngược. Để giải quyết, chúng ta phải tạo ra một Link ảo có tên là `camera_link_optical` và xoay nó đi một góc $-90^\circ$ (`-pi/2`) để ép hệ trục của ROS khớp với hệ trục của Camera.

---

## 2. Viết file `camera.xacro`

Tạo một file mới tên là `camera.xacro` và chèn đoạn code thiết kế cảm biến Camera kèm Plugin Gazebo vào:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <!-- 1. Định nghĩa kích thước hộp Camera -->
    <xacro:property name="camera_length" value="0.01" />
    <xacro:property name="camera_width" value="0.1" />
    <xacro:property name="camera_height" value="0.05" />

    <!-- Khối hộp đại diện cho thân Camera -->
    <link name="camera_link">
        <visual>
            <geometry>
                <box size="${camera_length} ${camera_width} ${camera_height}" />
            </geometry>
            <material name="grey" />
        </visual>
        <collision>
            <geometry>
                <box size="${camera_length} ${camera_width} ${camera_height}" />
            </geometry>
        </collision>
        <xacro:box_inertia m="0.1" x="${camera_length}" y="${camera_width}" z="${camera_height}"
                           o_xyz="0 0 0" o_rpy="0 0 0" />
    </link>

    <!-- Gắn cố định Camera vào mép trước của thân xe (base_link) -->
    <joint name="base_camera_joint" type="fixed">
        <parent link="base_link" />
        <child link="camera_link" />
        <!-- Tính toán để đẩy camera ra đúng mép trước thân xe -->
        <origin xyz="${(base_length + camera_length) / 2.0} 0 ${base_height / 2.0}" rpy="0 0 0" />
    </joint>

    <!-- 2. Tạo Optical Frame ảo để xoay trục tọa độ cho xử lý ảnh -->
    <link name="camera_link_optical" />

    <!-- Khớp nối ảo này xoay trục Roll -90 độ và Yaw -90 độ -->
    <joint name="camera_optical_joint" type="fixed">
        <origin xyz="0 0 0" rpy="${-pi/2} 0 ${-pi/2}"/>
        <parent link="camera_link"/>
        <child link="camera_link_optical"/>
    </joint>

    <!-- 3. Plugin Cảm biến Camera giả lập cho Gazebo -->
    <gazebo reference="camera_link">
        <sensor name="camera" type="camera">
            <camera>
                <!-- Góc nhìn ngang (FOV): ~80 độ (1.39 radian) -->
                <horizontal_fov>1.3962634</horizontal_fov>
                <!-- Độ phân giải ảnh 640x480, hệ màu RGB -->
                <image>
                    <width>640</width>
                    <height>480</height>
                    <format>R8G8B8</format>
                </image>
                <!-- Tầm nhìn: từ 10cm đến 15m -->
                <clip>
                    <near>0.1</near>
                    <far>15</far>
                </clip>
                <!-- Thêm nhiễu Gauss cho giống thực tế -->
                <noise>
                    <type>gaussian</type>
                    <mean>0.0</mean>
                    <stddev>0.007</stddev>
                </noise>
                <!-- Quan trọng: Ép Gazebo xuất ảnh theo trục Optical thay vì trục Link vật lý -->
                <optical_frame_id>camera_link_optical</optical_frame_id>
                <camera_info_topic>camera/camera_info</camera_info_topic>
            </camera>
            <always_on>1</always_on>
            <!-- Tốc độ khung hình: 20 FPS -->
            <update_rate>20</update_rate>
            <!-- Cho phép hiển thị khung camera trong Gazebo UI -->
            <visualize>true</visualize>
            <!-- Topic phát dữ liệu ảnh -->
            <topic>camera/image_raw</topic>            
        </sensor>
    </gazebo>

</robot>
```

*(Bạn nhớ mở file `my_robot.urdf.xacro` gốc và thêm dòng `<xacro:include filename="camera.xacro"/>` vào cùng với các bộ phận khác).*

---

## 3. Cập nhật Bridge để truyền dữ liệu Hình ảnh

Hình ảnh là dạng dữ liệu rất nặng. Gazebo phát ảnh ra topic tên là `/camera/image_raw`. Để ROS 2 nhận được những hình ảnh này, ta phải khai báo thêm 2 cầu nối vào file `config/gazebo_bridge.yaml`: một cho dữ liệu hình ảnh (`Image`) và một cho thông số ống kính (`CameraInfo`).

```yaml
# ... (Các cấu hình cmd_vel, tf, joint_states, scan đã có) ...

# Cầu nối dữ liệu hình ảnh thô (Image)
- ros_topic_name: "/camera/image_raw"
  gz_topic_name: "/camera/image_raw"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msg.Image"
  direction: GZ_TO_ROS

# Cầu nối thông tin cấu hình ống kính (Tiêu cự, ma trận nội...)
- ros_topic_name: "/camera/camera_info"
  gz_topic_name: "/camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msg.CameraInfo"
  direction: GZ_TO_ROS
```

---

## 4. Hiển thị luồng Video trực tiếp trên RViz2

1. **Khởi chạy lại Launch file của bạn:**
   ```bash
   ros2 launch my_mobilerobot gazebo.launch.xml
   ```
2. Bên trong Gazebo, bạn sẽ thấy một khối vuông xám gắn trước mũi xe. Nếu bạn đặt một vật cản (như cái hộp) trước mặt robot, camera sẽ ghi hình vật thể đó.
3. **Để xem luồng video trực tiếp trên RViz2:**
   * Nhấn nút **Add** ở góc dưới bên trái giao diện RViz.
   * Chọn mục **By Topic**.
   * Tìm đến Topic `/camera/image_raw` và chọn mục **Image** bên dưới nó.
   * Một cửa sổ thu nhỏ sẽ hiện ra góc trái màn hình hiển thị trực tiếp camera hành trình từ góc nhìn của robot. *(Để hiển thị mượt mà nhất, hãy đảm bảo cột **Reliability Policy** trong RViz2 khớp với QoS của Gazebo, thường là `Best Effort` hoặc `Reliable`).*

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo](/posts/2026/11/ros2-tutorial-22/)
