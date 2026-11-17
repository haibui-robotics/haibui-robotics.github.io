---
layout: single
title: "Tutorial 22: Gắn cảm biến LiDAR 2D vào Gazebo và hiển thị trên RViz2 (Dự án amr_ws – Commit V3)"
permalink: /posts/2026/11/ros2-tutorial-22/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 21 - Làm cho robot di chuyển trong Gazebo](/posts/2026/11/ros2-tutorial-21/) | [Bài tiếp: Tutorial 23 - Gắn cảm biến Camera 2D & Optical Frame ➡](/posts/2026/11/ros2-tutorial-23/)

---

## 1. Mục tiêu và Khái niệm

**LiDAR 2D** là cảm biến quan trọng bậc nhất giúp robot tự hành (AMR) quét bản đồ môi trường (**SLAM**) và phát hiện vật cản (**Navigation**). Nó phát ra các tia laser xoay $360^\circ$ quanh mặt phẳng ngang và đo thời gian phản xạ để tính khoảng cách tới các bức tường.

Trong dự án **[`amr_ws`](/portfolio/amr-ws/)** *(commit **`V3-lidar+cam+imu`**)*, phần khung hình học 3D của LiDAR (`lidar_link` với file lưới `lidar_link.STL` và khớp cố định `lidar_joint`) đã được xuất sẵn từ SolidWorks nằm trong file `urdf/amr.urdf.xacro`. Do đó, ở bài này chúng ta chỉ cần:

1. **Viết file `urdf/lidar.xacro`:** Gắn trực tiếp cảm biến `<sensor name="gpu_lidar" type="gpu_lidar">` vào `lidar_link` và nhúng vào `amr.urdf.xacro`.
2. **Cập nhật `config/gazebo_bridge.yaml`:** Mở cầu nối chuyển dữ liệu quét `/scan` từ Gazebo Sim sang ROS 2.
3. **Hiển thị tia quét `LaserScan` trên RViz2.**

---

## 2. Nguồn tra cứu Code mẫu `<sensor>` chuẩn của Gazebo (SDFormat)

Khi cần tra cứu cú pháp XML chuẩn của mọi loại cảm biến trong Gazebo (`gpu_lidar`, `camera`, `imu`, `rgbd_camera`...), bạn có thể lấy trực tiếp từ kho mã nguồn **SDFormat** chính thức của Open Robotics:

* 🔗 **Kho mã nguồn SDFormat:** [https://github.com/gazebosim/sdformat/tree/main](https://github.com/gazebosim/sdformat/tree/main)
* 🔗 **File mẫu tổng hợp tất cả các loại `<sensor>` (`sensors.sdf`):** [https://github.com/gazebosim/sdformat/blob/main/test/sdf/sensors.sdf](https://github.com/gazebosim/sdformat/blob/main/test/sdf/sensors.sdf)

---

## 3. Viết file `urdf/lidar.xacro` (Commit V3)

Trong `amr.urdf.xacro`, `lidar_link` và `lidar_joint` đã có sẵn từ bản vẽ CAD:

```xml
  <!-- Đã có sẵn trong urdf/amr.urdf.xacro -->
  <link name="lidar_link">
    ...
    <visual>
      <geometry>
        <mesh filename="package://amr_description/meshes/lidar_link.STL" />
      </geometry>
    </visual>
    <collision>
      <geometry>
        <mesh filename="package://amr_description/meshes/lidar_link.STL" />
      </geometry>
    </collision>
  </link>
  <joint name="lidar_joint" type="fixed">
    <origin xyz="0.0077825 3.5032E-05 0.1756" rpy="0 0 0" />
    <parent link="base_link" />
    <child link="lidar_link" />
  </joint>
```

Bây giờ, ta tạo file `urdf/lidar.xacro` để gắn cảm biến `gpu_lidar` vào `lidar_link`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <gazebo reference="lidar_link">
        <sensor name="gpu_lidar" type="gpu_lidar">   <!--loai gpu_lidar -> dung gpu de tinh toan va cham cua cac tia laser thay vi dung CPU (tranh giat lag)-->
            <always_on>true</always_on>   <!--luon luon chay-->
            <visualize>true</visualize>  <!--hien thi mau sac-->
            <update_rate>10</update_rate>  <!--tan so quet 10 lan / giay -->

            <topic>scan</topic>   <!--phat du lieu len scan-->
            <gz_frame_id>lidar_link</gz_frame_id>  <!--tia laser duoc phat ra tu lidar_link-->

            <lidar>
                <scan>
                    <horizontal>
                        <samples>360</samples> <!--quet 360 tia laser-->
                        <resolution>1</resolution> <!--moi tia cach 1 do-->
                        <min_angle>-3.14159</min_angle>  <!--goc quet nho nhat-->
                        <max_angle>3.14159</max_angle>  <!--goc quet lon nhat-->
                    </horizontal>
                </scan>

                <range>
                    <min>0.15</min>  <!--khoang cach nho nhat 15cm-->
                    <max>5.0</max>  <!--khoang cach lon nhat 5m-->
                    <resolution>0.01</resolution>  <!--do phan giai cua khoang cach-->
                </range>

                <noise>
                    <type>gaussian</type>
                    <mean>0.0</mean>
                    <stddev>0.01</stddev>  <!--them nhieu sai so 1cm, giong thuc te de AMCL,SLAM xu ly loc nhieu-->
                </noise>

            </lidar>
           
        </sensor>
    </gazebo>

</robot>
```

Sau đó mở đầu file `urdf/amr.urdf.xacro` và thêm dòng nhúng `lidar.xacro`:

```xml
<robot name="amr_description" xmlns:xacro="http://www.ros.org/wiki/xacro">
  <xacro:include filename="$(find amr_description)/urdf/lidar.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/gazebo_sim_controll.xacro"/>
```

---

## 4. Cập nhật Cầu nối `config/gazebo_bridge.yaml`

Gazebo phát dữ liệu quét laser lên topic `/scan` với kiểu `gz.msgs.LaserScan`. Mở file `config/gazebo_bridge.yaml` (từ commit `V2`) và nối thêm khối `/scan` vào cuối file:

```yaml
# ... (5 khối /clock, /cmd_vel, /odom, /tf, /joint_states đã có ở Tutorial 21) ...

- ros_topic_name: "/scan"
  gz_topic_name: "/scan"
  ros_type_name: "sensor_msgs/msg/LaserScan"
  gz_type_name: "gz.msgs.LaserScan"
  direction: GZ_TO_ROS
```

---

## 5. Kiểm tra và Hiển thị `LaserScan` trên RViz2

1. **Build và khởi chạy mô phỏng:**
   ```bash
   cd ~/amr_ws
   colcon build --packages-select amr_description
   source install/setup.bash
   ros2 launch amr_description gazebo.launch.xml
   ```
2. **Hiển thị điểm quét LiDAR trên RViz2:**
   * Nhấn nút **Add** ở góc dưới bên trái RViz2 → chọn tab **By Topic**.
   * Tìm đến `/scan` và chọn **LaserScan**.
   * *(Mẹo)* Tăng mục **Size (m)** trong thuộc tính của `LaserScan` lên `0.03` – `0.05` để các điểm chấm đỏ hiện rõ dọc theo các kệ hàng trong nhà kho!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 21 - Làm cho robot di chuyển trong Gazebo](/posts/2026/11/ros2-tutorial-21/) | [Bài tiếp: Tutorial 23 - Gắn cảm biến Camera 2D & Optical Frame ➡](/posts/2026/11/ros2-tutorial-23/)
