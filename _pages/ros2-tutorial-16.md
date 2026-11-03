---
layout: single
title: "Tutorial 16: Viết bản thiết kế URDF đầu tiên (Khai báo Base & Wheels)"
permalink: /posts/2026/11/ros2-tutorial-16/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 15 - Khái niệm TF2 & TF Tree](/posts/2026/11/ros2-tutorial-15/) | [Bài tiếp: Tutorial 18 - Hiển thị robot trên RViz2 ➡](/posts/2026/11/ros2-tutorial-18/)

---

## 1. Cấu trúc cốt lõi của file URDF

**URDF (Unified Robot Description Format)** là file XML dùng để định nghĩa hình dáng và cấu trúc vật lý của robot. Một file URDF cơ bản gồm các thẻ chính sau:

* **`<link>`**: Đại diện cho một bộ phận cứng của robot (ví dụ: thân xe, bánh xe). Bên trong link thường có:
  * **`<visual>`**: Khai báo hình dáng (`box`, `cylinder`, `sphere`, `mesh`) và màu sắc để hiển thị.
  * **`<collision>`**: Khai báo khung va chạm vật lý (thường đơn giản hơn `visual` để giảm tải tính toán).
  * **`<inertial>`**: Khai báo khối lượng và quán tính (sẽ dùng cho Gazebo ở các bài sau).
* **`<joint>`**: Khai báo khớp nối giữa 2 Link. Cần chỉ định rõ `<parent>` (link cha), `<child>` (link con) và `<origin>` (khoảng cách/góc xoay giữa gốc tọa độ của 2 link).
* **`<material>`**: Khai báo màu sắc (mã RGBA) để tái sử dụng nhiều lần.

### Các loại Joint (Khớp nối) phổ biến:
* **`fixed`**: Cố định hoàn toàn *(Ví dụ: hàn LiDAR vào thân xe)*. Không cần trục xoay (`<axis>`).
* **`continuous`**: Khớp xoay liên tục không giới hạn góc *(Ví dụ: bánh xe)*.
* **`revolute`**: Khớp xoay có giới hạn góc *(Ví dụ: cánh tay robot, servo)*. Cần thêm thẻ `<limit>` để xác định góc min/max, lực và vận tốc.
* **`prismatic`**: Khớp trượt tịnh tiến theo một trục *(Ví dụ: cơ cấu nâng hạ)*. Cần thẻ `<limit>`.

---

## 2. Thực hành: Viết file URDF cho xe 2 bánh

Chúng ta sẽ viết mã cho một robot gồm 1 thân xe dạng hộp chữ nhật, 2 bánh chính (hình trụ) và 1 bánh xe tự lựa tự do (hình cầu). Tạo một file có tên `my_robot.urdf` và dán đoạn mã sau vào:

```xml
<?xml version="1.0"?>
<robot name="my_robot">
    <!-- 1. Định nghĩa màu sắc -->
    <material name="green">
        <color rgba="0 0.5 0.2 1"/>
    </material>
    <material name="blue">
        <color rgba="0 0 0.5 1"/>
    </material>

    <!-- 2. Khai báo các Links -->
    <link name="base"/> <!-- Link ảo làm gốc chuẩn -->
    
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

    <!-- 3. Khai báo các Joints -->
    <joint name="joint_base" type="fixed">
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

*(Lưu ý: Lúc này chúng ta mới chỉ viết ra bản vẽ tĩnh. Sang [Tutorial 18](/posts/2026/11/ros2-tutorial-18/), chúng ta mới học cách đóng gói và hiển thị hình hài thực tế của nó lên màn hình).*

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 15 - Khái niệm TF2 & TF Tree](/posts/2026/11/ros2-tutorial-15/) | [Bài tiếp: Tutorial 18 - Hiển thị robot trên RViz2 ➡](/posts/2026/11/ros2-tutorial-18/)
