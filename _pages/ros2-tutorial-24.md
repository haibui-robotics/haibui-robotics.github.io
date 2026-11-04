---
layout: single
title: "Tutorial 24: Gắn cảm biến Depth Camera (RGB-D) và hiển thị đám mây điểm 3D (PointCloud2) trên RViz2"
permalink: /posts/2026/11/ros2-tutorial-24/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 23 - Gắn cảm biến Camera 2D & Optical Frame](/posts/2026/11/ros2-tutorial-23/) | [Bài tiếp: Tutorial 25 - Gắn cảm biến IMU vào Gazebo ➡](/posts/2026/11/ros2-tutorial-25/)

---

## 1. Mục tiêu và Khái niệm cốt lõi (Tại sao cần Depth Camera?)

Ở [Tutorial 22](/posts/2026/11/ros2-tutorial-22/) và [Tutorial 23](/posts/2026/11/ros2-tutorial-23/), robot của chúng ta đã có **LiDAR 2D** và **Camera 2D (RGB)**. Tuy nhiên, trong môi trường thực tế (nhà kho, văn phòng, bệnh viện), hai cảm biến này vẫn có điểm mù chí mạng:

* **Điểm mù của LiDAR 2D:** Chỉ quét được một lát cắt ngang duy nhất ở độ cao gắn cảm biến (ví dụ $z = 0.2\text{m}$). Nếu trước mặt xe là một chiếc bàn (gầm bàn trống nhưng mặt bàn thấp hơn đỉnh robot) hoặc một càng xe nâng đang nhô ra trên không, tia LiDAR 2D sẽ đi xuyên qua gầm và robot sẽ đâm sầm vào mặt bàn!
* **Hạn chế của Camera 2D (RGB):** Nhìn thấy ảnh màu phẳng nhưng không biết vật thể đó cách đầu xe bao nhiêu mét.

**Depth Camera (Camera chiều sâu / RGB-D)** — điển hình ngoài thực tế là dòng **Intel RealSense D435i / D455** hoặc **Orbbec Astra** — giải quyết triệt để bài toán này bằng cách cung cấp đồng thời 3 luồng dữ liệu:

1. **RGB Image (`sensor_msgs/msg/Image`):** Ảnh màu 2D thông thường.
2. **Depth Image (`sensor_msgs/msg/Image`):** Ảnh chiều sâu (mỗi điểm ảnh lưu khoảng cách thực tế tính bằng mét từ camera tới bề mặt vật cản).
3. **PointCloud2 (`sensor_msgs/msg/PointCloud2`):** **Đám mây điểm 3D** — tập hợp hàng chục ngàn điểm có tọa độ $(x, y, z)$ và màu sắc thực trong không gian 3 chiều, giúp robot tái tạo nguyên vẹn hình khối 3D của môi trường phía trước.

---

## 2. Viết file `depth_camera.xacro` (Cảm biến `rgbd_camera`)

Trong Gazebo Sim (Harmonic), loại cảm biến `rgbd_camera` cho phép giả lập một chiếc Camera RGB-D hoàn chỉnh.

Tương tự như Camera 2D ở Tutorial 23, dữ liệu chiều sâu và đám mây điểm `PointCloud2` sử dụng hệ trục quang học (**Optical Frame**: trục $Z$ hướng thẳng về phía trước). Vì vậy, ta cần tạo `depth_camera_link_optical` xoay góc $(-\pi/2, 0, -\pi/2)$ và gắn `<gz_frame_id>` vào trục optical này để đám mây điểm 3D hiển thị đúng hướng trên RViz2.

Tạo file `urdf/depth_camera.xacro`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <!-- 1. Kích thước cụm cảm biến Depth Camera (mô phỏng Intel RealSense D435) -->
    <xacro:property name="depth_cam_length" value="0.025" />
    <xacro:property name="depth_cam_width" value="0.09" />
    <xacro:property name="depth_cam_height" value="0.025" />

    <link name="depth_camera_link">
        <visual>
            <geometry>
                <box size="${depth_cam_length} ${depth_cam_width} ${depth_cam_height}" />
            </geometry>
            <material name="blue" />
        </visual>
        <collision>
            <geometry>
                <box size="${depth_cam_length} ${depth_cam_width} ${depth_cam_height}" />
            </geometry>
        </collision>
        <xacro:box_inertia m="0.08" x="${depth_cam_length}" y="${depth_cam_width}" z="${depth_cam_height}"
                           o_xyz="0 0 0" o_rpy="0 0 0" />
    </link>

    <!-- Gắn cố định Depth Camera ở mũi trước thân xe -->
    <joint name="base_depth_camera_joint" type="fixed">
        <parent link="base_link" />
        <child link="depth_camera_link" />
        <origin xyz="${(base_length + depth_cam_length) / 2.0} 0 ${base_height / 2.0}" rpy="0 0 0" />
    </joint>

    <!-- 2. Khung tọa độ quang học (Optical Frame) để PointCloud2 không bị lật trục -->
    <link name="depth_camera_link_optical" />

    <joint name="depth_camera_optical_joint" type="fixed">
        <parent link="depth_camera_link" />
        <child link="depth_camera_link_optical" />
        <origin xyz="0 0 0" rpy="${-pi/2} 0 ${-pi/2}" />
    </joint>

    <!-- 3. Plugin Cảm biến RGB-D (rgbd_camera) trong Gazebo Sim -->
    <gazebo reference="depth_camera_link">
        <sensor name="depth_camera" type="rgbd_camera">
            <always_on>true</always_on>
            <!-- Tần số quét 3D: 15 Hz (tối ưu hiệu năng CPU/GPU) -->
            <update_rate>15</update_rate>
            <visualize>true</visualize>
            <!-- Tiền tố Topic phát ra trong Gazebo -->
            <topic>depth_camera</topic>
            <!-- Quan trọng: Gắn frame_id vào trục Optical -->
            <gz_frame_id>depth_camera_link_optical</gz_frame_id>
            <camera>
                <!-- Góc mở ngang ~85 độ (1.48 rad) -->
                <horizontal_fov>1.4835</horizontal_fov>
                <image>
                    <width>320</width>
                    <height>240</height>
                    <format>R8G8B8</format>
                </image>
                <!-- Khoảng cách đo chiều sâu: từ 0.2m đến 6.0m -->
                <clip>
                    <near>0.2</near>
                    <far>6.0</far>
                </clip>
                <optical_frame_id>depth_camera_link_optical</optical_frame_id>
            </camera>
        </sensor>
    </gazebo>

</robot>
```

*(Đừng quên nhúng `<xacro:include filename="depth_camera.xacro" />` vào file `my_robot.urdf.xacro` gốc của bạn).*

---

## 3. Cập nhật Cầu nối `ros_gz_bridge`

Khi cảm biến `rgbd_camera` chạy với `<topic>depth_camera</topic>`, Gazebo sẽ tự động sinh ra 4 topic con:
* `/depth_camera/image` (Ảnh màu RGB)
* `/depth_camera/depth_image` (Bản đồ chiều sâu 2D)
* `/depth_camera/camera_info` (Ma trận thông số ống kính)
* `/depth_camera/points` (Đám mây điểm 3D – kiểu `gz.msg.PointCloudPacked`)

Hãy mở file `config/gazebo_bridge.yaml` và thêm cấu hình cầu nối cho 4 topic này:

```yaml
# ... (Các cấu hình cmd_vel, tf, joint_states, scan đã có) ...

# 1. Ảnh màu từ Depth Camera
- ros_topic_name: "/depth_camera/image"
  gz_topic_name: "/depth_camera/image"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msg.Image"
  direction: GZ_TO_ROS

# 2. Ảnh bản đồ chiều sâu (Depth Map)
- ros_topic_name: "/depth_camera/depth_image"
  gz_topic_name: "/depth_camera/depth_image"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msg.Image"
  direction: GZ_TO_ROS

# 3. Thông số nội tại của Camera
- ros_topic_name: "/depth_camera/camera_info"
  gz_topic_name: "/depth_camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msg.CameraInfo"
  direction: GZ_TO_ROS

# 4. Đám mây điểm 3D (PointCloud2)
- ros_topic_name: "/depth_camera/points"
  gz_topic_name: "/depth_camera/points"
  ros_type_name: "sensor_msgs/msg/PointCloud2"
  gz_type_name: "gz.msg.PointCloudPacked"
  direction: GZ_TO_ROS
```

---

## 4. Hiển thị Đám mây điểm 3D (`PointCloud2`) trên RViz2

1. **Build và khởi chạy mô phỏng:**
   ```bash
   colcon build
   source install/setup.bash
   ros2 launch my_mobilerobot gazebo.launch.xml
   ```
2. **Thêm hiển thị 3D trên RViz2:**
   * Nhấn nút **Add** (góc dưới bên trái RViz2) → chuyển sang tab **By Topic**.
   * Tìm đến `/depth_camera/points` và chọn **PointCloud2**.
   * Bạn cũng có thể thêm `/depth_camera/depth_image` (**Image**) để xem ảnh phân bố khoảng cách dạng thang độ xám (vật càng gần càng tối, càng xa càng sáng).
3. **Tinh chỉnh hiển thị trong mục `PointCloud2`:**
   * **Style:** Chọn `Points` hoặc `Flat Squares`.
   * **Size (Pixels / Meters):** Đặt khoảng `0.02` để các điểm dày đặc và rõ nét hơn.
   * **Color Transformer:** Chọn `RGB8` (để hiển thị màu sắc thực của vật thể) hoặc `AxisColor` (tô màu cầu vồng theo khoảng cách/độ cao).

Thử đặt một chiếc bàn hoặc kệ hàng trước đầu xe trong Gazebo, bạn sẽ thấy toàn bộ hình dáng 3D của chiếc bàn hiện lên sống động trong không gian RViz2!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 23 - Gắn cảm biến Camera 2D & Optical Frame](/posts/2026/11/ros2-tutorial-23/) | [Bài tiếp: Tutorial 25 - Gắn cảm biến IMU vào Gazebo ➡](/posts/2026/11/ros2-tutorial-25/)
