---
layout: single
title: "Tutorial 16: Viết bản thiết kế URDF đầu tiên (Links & Joints)"
permalink: /posts/2026/11/ros2-tutorial-16/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 15 - Khái niệm TF2 & TF Tree](/posts/2026/11/ros2-tutorial-15/)

---

## 1. Cấu trúc cốt lõi của file URDF

**URDF (Unified Robot Description Format)** là file XML dùng để định nghĩa hình dáng và cấu trúc vật lý của robot. Một file URDF cơ bản gồm các thẻ chính sau:

* **`<link>`**: Đại diện cho một bộ phận cứng của robot (ví dụ: thân xe, bánh xe). Bên trong link thường có:
  * **`<visual>`**: Khai báo hình dáng (`box`, `cylinder`, `sphere`, `mesh`) và màu sắc để hiển thị.
  * **`<collision>`**: Khai báo khung va chạm vật lý (thường đơn giản hơn `visual` để giảm tải tính toán).
  * **`<inertial>`**: Khai báo khối lượng và quán tính (dùng cho Gazebo).
* **`<joint>`**: Khai báo khớp nối giữa 2 Link. Cần chỉ định rõ `<parent>` (link cha), `<child>` (link con) và `<origin>` (khoảng cách/góc xoay giữa 2 link).
* **`<material>`**: Khai báo màu sắc (mã RGBA) để tái sử dụng nhiều lần.

### Các loại Joint (Khớp nối) phổ biến:
* **`fixed`**: Cố định hoàn toàn *(Ví dụ: hàn LiDAR vào thân xe)*. Không cần trục xoay (`<axis>`).
* **`continuous`**: Khớp xoay liên tục không giới hạn *(Ví dụ: bánh xe)*.
* **`revolute`**: Khớp xoay có giới hạn góc *(Ví dụ: cánh tay robot, servo)*. Cần thêm thẻ `<limit>` để xác định góc min/max, lực và vận tốc.
* **`prismatic`**: Khớp trượt tịnh tiến theo một trục *(Ví dụ: thang máy mini)*. Cần thẻ `<limit>`.

---

## 2. Xây dựng cấu trúc Package mô phỏng

Trước khi viết code, bạn cần tạo một package (ví dụ: `my_robot_description`) với cấu trúc thư mục chuẩn mực như sau để dễ quản lý:

```text
my_robot_description/
├── CMakeLists.txt / package.xml
├── urdf/
│   └── my_robot.urdf         # File mô tả cấu trúc robot
├── meshes/                   # (Tùy chọn) Chứa file 3D CAD (.stl) nếu có
├── launch/
│   └── display.launch.xml    # File khởi chạy hệ thống và RViz
└── rviz/
    └── urdf_config.rviz      # File lưu cấu hình giao diện RViz2
```

---

## 3. Ví dụ thực hành: Viết file URDF cho xe 2 bánh

Tạo file `urdf/my_robot.urdf`. Robot này gồm 1 thân xe, 2 bánh chính và 1 bánh xe tự lựa (*caster wheel*).

```xml
<?xml version="1.0"?>
<robot name="my_robot">
    <!-- Định nghĩa màu sắc -->
    <material name="green">
        <color rgba="0 0.5 0.2 1"/>
    </material>
    <material name="blue">
        <color rgba="0 0 0.5 1"/>
    </material>

    <!-- Khai báo các Links -->
    <link name="base"/>
    
    <link name="base_link">
        <visual>
            <geometry>
                <box size="0.6 0.4 0.2"/>
            </geometry>
            <!-- Dịch khối hộp lên 0.1m theo trục Z -->
            <origin xyz="0 0 0.1" rpy="0 0 0"/>
            <material name="green"/>
        </visual>
    </link>

    <link name="right_wheel">
        <visual>
            <geometry>
                <cylinder radius="0.1" length="0.05"/>
            </geometry>
            <origin xyz="0 0 0" rpy="1.57 0 0"/>
            <material name="blue"/>
        </visual>
    </link>

    <link name="left_wheel">
        <visual>
            <geometry>
                <cylinder radius="0.1" length="0.05"/>
            </geometry>
            <origin xyz="0 0 0" rpy="1.57 0 0"/>
            <material name="blue"/>
        </visual>
    </link>

    <link name="caster_wheel">
        <visual>
            <geometry>
                <sphere radius="0.05"/>
            </geometry>
            <origin xyz="0 0 0" rpy="0 0 0"/>
            <material name="blue"/>
        </visual>
    </link>

    <!-- Khai báo các Joints -->
    <joint name="joint" type="fixed">
        <parent link="base"/>
        <child link="base_link"/>
        <origin xyz="0 0 0.1" rpy="0 0 0"/>
    </joint>

    <joint name="right_joint" type="continuous">
        <parent link="base_link"/>
        <child link="right_wheel"/>
        <!-- Vị trí bánh phải so với tâm thân xe -->
        <origin xyz="-0.2 -0.225 0" rpy="0 0 0"/>
        <!-- Trục xoay là trục Y -->
        <axis xyz="0 1 0"/> 
    </joint>

    <joint name="left_joint" type="continuous">
        <parent link="base_link"/>
        <child link="left_wheel"/>
        <origin xyz="-0.2 0.225 0" rpy="0 0 0"/>
        <axis xyz="0 1 0"/>
    </joint>

    <joint name="font_joint" type="fixed">
        <parent link="base_link"/>
        <child link="caster_wheel"/>
        <origin xyz="0.2 0 -0.05" rpy="0 0 0"/>
    </joint>
</robot>
```

---

## 4. Hiểu về các Node phụ trợ hiển thị

Để hiển thị URDF lên RViz2, chúng ta không thể chỉ ném file URDF vào RViz. Cần có 2 Node trung gian đóng vai trò biên dịch:

* **`robot_state_publisher`**: Đây là bộ não tính toán **Động học thuận (Forward Kinematics)**. Nó nhận file URDF (dạng text), đọc cấu trúc cha-con, tính toán vị trí 3D chính xác của từng bộ phận và phát liên tục lên cây tọa độ TF (`/tf` và `/tf_static`). Nếu không có Node này, RViz hay thuật toán Nav2 sẽ hoàn toàn *"mù"* về cấu trúc robot.
* **`joint_state_publisher_gui`**: Cung cấp một bảng điều khiển có các thanh trượt (*slider*). Khi bạn kéo thanh trượt, Node này phát dữ liệu góc xoay của bánh xe/khớp nối lên Topic `/joint_states`. Node `robot_state_publisher` sẽ lắng nghe topic này để cập nhật TF ngay lập tức.

---

## 5. Viết Launch File để khởi chạy toàn hệ thống

Tạo file `launch/display.launch.xml` để tự động hóa việc mở mô hình. File launch này sẽ gọi biến môi trường, khởi chạy 2 Node phụ trợ và mở RViz2 cùng lúc.

```xml
<launch>
    <!-- 1. Tạo các biến đường dẫn tạm thời -->
    <let name="urdf_path" value="$(find-pkg-share my_robot_description)/urdf/my_robot.urdf" />
    <let name="rviz_config_path" value="$(find-pkg-share my_robot_description)/rviz/urdf_config.rviz"/>

    <!-- 2. Khởi chạy robot_state_publisher -->
    <!-- Sử dụng lệnh 'xacro' để parse file URDF truyền vào tham số robot_description -->
    <node pkg="robot_state_publisher" exec="robot_state_publisher">
        <param name="robot_description" value="$(command 'xacro $(var urdf_path)')" />
    </node>

    <!-- 3. Khởi chạy bảng điều khiển thanh trượt cho các khớp -->
    <node pkg="joint_state_publisher_gui" exec="joint_state_publisher_gui"/>

    <!-- 4. Mở giao diện 3D RViz2 kèm file cấu hình lưu sẵn -->
    <node pkg="rviz2" exec="rviz2" name="rviz2" output="screen" args="-d $(var rviz_config_path)"/>
</launch>
```

Để xem kết quả, bạn mở Terminal, build package (bằng `colcon build`) và chạy lệnh:

```bash
ros2 launch my_robot_description display.launch.xml
```

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 15 - Khái niệm TF2 & TF Tree](/posts/2026/11/ros2-tutorial-15/)
