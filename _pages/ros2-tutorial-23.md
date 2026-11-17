---
layout: single
title: "Tutorial 23: Gắn cảm biến Camera 2D và bí ẩn của 'Optical Frame' (Dự án amr_ws – Commit V3)"
permalink: /posts/2026/11/ros2-tutorial-23/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo](/posts/2026/11/ros2-tutorial-22/) | [Bài tiếp: Tutorial 24 - Gắn cảm biến Depth Camera (RGB-D) ➡](/posts/2026/11/ros2-tutorial-24/)

---

## 1. Mục tiêu và Khái niệm cốt lõi (`Optical Frame` là gì?)

Trong bài này *(thuộc commit **`V3-lidar+cam+imu`** của dự án **[`amr_ws`](/portfolio/amr-ws/)***)*, chúng ta sẽ kích hoạt chiếc **Camera RGB** (`cam_link`) gắn ở mũi trước của xe AMR để thu nhận hình ảnh màu trực tiếp từ môi trường Gazebo Sim.

Khái niệm quan trọng nhất khi làm việc với Camera trong ROS 2 là sự khác biệt về **Hệ trục tọa độ (Coordinate Frame)**:

* **Chuẩn của ROS 2 (Robot Frame):** Trục $X$ hướng về phía trước, Trục $Y$ hướng sang trái, Trục $Z$ hướng lên trời.
* **Chuẩn của Xử lý ảnh / OpenCV (Optical Frame):** Trục $Z$ hướng thẳng về phía trước (theo chiều ánh sáng đi vào ống kính), Trục $X$ sang phải, Trục $Y$ hướng xuống đất.

Để hình ảnh và thông số ống kính (`CameraInfo`) không bị xoay ngược khi đưa sang các thuật toán thị giác máy tính hay SLAM, ta tạo thêm một Link ảo **`cam_link_optical`** gắn vào `cam_link` và xoay một góc $\text{Roll} = -90^\circ$ (`-pi/2`), $\text{Yaw} = -90^\circ$ (`-pi/2`).

---

## 2. Nguồn tra cứu Code mẫu `<sensor>` chuẩn của Gazebo (SDFormat)

* 🔗 **Kho mã nguồn SDFormat:** [https://github.com/gazebosim/sdformat/tree/main](https://github.com/gazebosim/sdformat/tree/main)
* 🔗 **File mẫu tổng hợp tất cả các loại `<sensor>` (`sensors.sdf`):** [https://github.com/gazebosim/sdformat/blob/main/test/sdf/sensors.sdf](https://github.com/gazebosim/sdformat/blob/main/test/sdf/sensors.sdf)

---

## 3. Khai báo `cam_link_optical` và viết file `urdf/camera.xacro` (Commit V3)

### Bước 1: Khai báo `cam_link_optical` trong `urdf/amr.urdf.xacro`
Trong file `amr.urdf.xacro`, ngay bên dưới `cam_link` và `cam_joint` (đã có sẵn từ bản vẽ CAD `cam_link.STL`), ta thêm `cam_link_optical` và `cam_optical_joint`:

```xml
  <!-- Khớp gắn cam_link vào thân xe base_link (đã có sẵn trong amr.urdf.xacro) -->
  <joint name="cam_joint" type="fixed">
    <origin xyz="0.098593 0.00095321 0.119" rpy="0 0 0" />
    <parent link="base_link" />
    <child link="cam_link" />
    <axis xyz="0 0 0" />
  </joint>

  <!-- [THÊM] Link ảo Optical Frame cho xử lý ảnh -->
  <link name="cam_link_optical" />

  <!-- Khớp nối ảo này xoay trục Roll -90 độ và Yaw -90 độ -->
  <joint name="cam_optical_joint" type="fixed">
    <origin xyz="0 0 0" rpy="${-pi/2} 0 ${-pi/2}"/>
    <parent link="cam_link"/>
    <child link="cam_link_optical"/>
  </joint>
```

### Bước 2: Tạo file `urdf/camera.xacro`
Tạo file `urdf/camera.xacro` gắn cảm biến `camera` vào `cam_link` và trỏ `<optical_frame_id>` về `cam_link_optical`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <gazebo reference="cam_link">
        <sensor name="camera" type="camera">
            <always_on>true</always_on>
            <visualize>true</visualize>
            <update_rate>30</update_rate>  <!--chup va gui 30 khung/giay-->

            <topic>camera/image_raw</topic>   <!--du lieu pixel tho(ma tran anh)-->
            <gz_frame_id>cam_link</gz_frame_id>

            <camera>
                <horizontal_fov>1.3962634</horizontal_fov>  <!--goc nhin ngang-->

                <image>
                    <width>640</width>      <!--do phan giai 640x480 pixel-->
                    <height>480</height>
                    <format>R8G8B8</format>
                </image>

                <clip>    <!-- quy dinh vung nhin thay-->
                    <near>0.1</near>  <!--10cm-->
                    <far>15</far>  <!--15m-->
                </clip>
                <noise>
                    <type>gaussian</type>
                    <mean>0.0</mean>
                    <stddev>0.007</stddev>  <!--them nhieu sai so 0.7% giong thuc te de AMCL,SLAM xu ly loc nhieu-->
                </noise>
                <!--Chuẩn của Xử lý ảnh (Camera/OpenCV) Trục Z hướng thẳng về phía trước (theo chiều ánh sáng đi vào mắt), Trục X sang phải, Trục Y hướng xuống đất.-->
                <optical_frame_id>cam_link_optical</optical_frame_id>
                <camera_info_topic>camera/camera_info</camera_info_topic>  <!--dua ma tran pixel thanh anh 3D thuc te-->
            </camera>
        </sensor>
    </gazebo>

</robot>
```

Sau đó nhúng `camera.xacro` vào đầu file `urdf/amr.urdf.xacro`:

```xml
<robot name="amr_description" xmlns:xacro="http://www.ros.org/wiki/xacro">
  <xacro:include filename="$(find amr_description)/urdf/lidar.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/camera.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/gazebo_sim_controll.xacro"/>
```

---

## 4. Cập nhật Cầu nối `config/gazebo_bridge.yaml`

Mở file `config/gazebo_bridge.yaml` và thêm 2 cầu nối cho dữ liệu ảnh thô (`/camera/image_raw`) và thông số nội tại ống kính (`/camera/camera_info`):

```yaml
# ... (Các cấu hình đã có từ bài trước) ...

# Cầu nối dữ liệu hình ảnh thô (Image)
- ros_topic_name: "/camera/image_raw"
  gz_topic_name: "/camera/image_raw"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msgs.Image"
  direction: GZ_TO_ROS

# Cầu nối thông tin cấu hình ống kính (Tiêu cự, ma trận nội...)
- ros_topic_name: "/camera/camera_info"
  gz_topic_name: "/camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"
  direction: GZ_TO_ROS
```

---

## 5. Hiển thị luồng Video trực tiếp trên RViz2

1. **Build và khởi chạy lại Launch file:**
   ```bash
   cd ~/amr_ws
   colcon build --packages-select amr_description
   source install/setup.bash
   ros2 launch amr_description gazebo.launch.xml
   ```
2. **Xem hình ảnh Camera trên RViz2:**
   * Nhấn nút **Add** ở góc dưới bên trái RViz2 → chọn tab **By Topic**.
   * Tìm đến `/camera/image_raw` và chọn **Image**.
   * Một khung hình trực tiếp góc nhìn thứ nhất từ mũi xe AMR sẽ xuất hiện trên màn hình RViz2 với tốc độ 30 FPS!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo](/posts/2026/11/ros2-tutorial-22/) | [Bài tiếp: Tutorial 24 - Gắn cảm biến Depth Camera (RGB-D) ➡](/posts/2026/11/ros2-tutorial-24/)
