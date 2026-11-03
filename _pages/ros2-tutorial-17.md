---
layout: single
title: "Tutorial 17: Viết code thông minh hơn với Xacro (Biến, Hàm và Module hóa)"
permalink: /posts/2026/11/ros2-tutorial-17/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 16 - Viết bản thiết kế URDF đầu tiên](/posts/2026/11/ros2-tutorial-16/) | [Bài tiếp: Tutorial 18 - Hiển thị robot trên RViz2 ➡](/posts/2026/11/ros2-tutorial-18/)

---

## 1. Tại sao phải chuyển từ URDF sang Xacro?

Viết file URDF thuần (như ở [Tutorial 16](/posts/2026/11/ros2-tutorial-16/)) bộc lộ nhiều điểm yếu khi robot trở nên phức tạp:

* **Hardcode (Số cứng):** Nếu muốn đổi bán kính bánh xe từ `0.1` sang `0.15`, bạn phải tìm và sửa thủ công ở hàng loạt chỗ.
* **Lặp code:** Bánh trái và bánh phải có cấu trúc hình học giống hệt nhau, chỉ khác mỗi cái tên và tọa độ. Việc copy-paste làm file dài hàng ngàn dòng.
* **Gom chung 1 file:** Cả thông số vật liệu, thân xe, cánh tay, LiDAR đều nhét vào 1 file duy nhất khiến việc bảo trì rất khó khăn.

**Xacro (XML Macros)** giải quyết tất cả. Nó cung cấp các tính năng của một ngôn ngữ lập trình như: **Biến (Properties)**, **Hàm (Macros)**, **Toán học (`${pi/2}`)** và **Tách file (Include)**.

*(Lưu ý: Để sử dụng Xacro, thẻ gốc của bạn phải khai báo không gian tên: `<robot name="my_robot" xmlns:xacro="http://www.ros.org/wiki/xacro">`)*

---

## 2. Các tính năng cốt lõi của Xacro

### A. Properties (Biến không đổi)
Giúp bạn khai báo kích thước ở một nơi và sử dụng lại ở mọi chỗ.

```xml
<!-- Khai báo biến -->
<xacro:property name="wheel_radius" value="0.1" />
<xacro:property name="wheel_length" value="0.05" />

<!-- Sử dụng biến (đặt trong ngoặc nhọn ${}) -->
<cylinder radius="${wheel_radius}" length="${wheel_length}"/>
```

### B. Macros (Hàm tái sử dụng)
Bạn có thể định nghĩa một khối XML và truyền tham số cho nó. Ví dụ: Tạo hàm sinh ra Link cho bánh xe.

```xml
<!-- Khai báo Macro có tên wheel_link, nhận tham số prefix -->
<xacro:macro name="wheel_link" params="prefix">
    <link name="${prefix}_wheel_link">
        <visual>
            <geometry>
                <cylinder radius="0.1" length="0.05"/>
            </geometry>
            <!-- Xacro hỗ trợ toán học, dùng ${pi/2} thay vì số xấp xỉ 1.57 -->
            <origin xyz="0 0 0" rpy="${pi/2} 0 0"/>
            <material name="blue"/>
        </visual>
    </link>
</xacro:macro>

<!-- Gọi Macro (Tự động sinh ra 2 khối code cho bánh trái và bánh phải) -->
<xacro:wheel_link prefix="right" />
<xacro:wheel_link prefix="left" />
```

### C. Include (Nhúng file)
Cho phép bạn chia nhỏ hệ thống. Ví dụ: file màu sắc riêng, file khung gầm riêng, file LiDAR riêng.

---

## 3. Thực hành: Tái cấu trúc Robot sang hệ thống Xacro

Chúng ta sẽ chia lại file `my_robot.urdf` của bài trước thành 3 file `.xacro` riêng biệt để thấy sự gọn gàng. Đặt tất cả trong thư mục `urdf/`.

### File 1: `common_properties.xacro`
Chỉ chứa các định nghĩa về màu sắc/vật liệu dùng chung cho toàn bộ dự án.

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
    
    <material name="green">
        <color rgba="0 0.5 0.2 1"/>
    </material>
    
    <material name="blue">
        <color rgba="0 0 0.5 1"/>
    </material>

</robot>
```

### File 2: `mobile_base.xacro`
Chứa toàn bộ cấu trúc thân xe và bánh xe. Ở đây ta áp dụng Macro để tối ưu code bánh xe.

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <!-- Các biến kích thước -->
    <xacro:property name="base_length" value="0.6" />
    <xacro:property name="base_width" value="0.4" />
    <xacro:property name="base_height" value="0.2" />
    <xacro:property name="wheel_radius" value="0.1" />
    <xacro:property name="wheel_length" value="0.05" />

    <!-- Macro: Định nghĩa khối hình học chung cho bánh xe -->
    <xacro:macro name="wheel_link" params="prefix">
        <link name="${prefix}_wheel">
            <visual>
                <geometry>
                    <cylinder radius="${wheel_radius}" length="${wheel_length}"/>
                </geometry>
                <origin xyz="0 0 0" rpy="${pi/2} 0 0"/>
                <material name="blue"/>
            </visual>
        </link>
    </xacro:macro>

    <!-- Khai báo Base -->
    <link name="base"/>
    
    <link name="base_link">
        <visual>
            <geometry>
                <box size="${base_length} ${base_width} ${base_height}"/>
            </geometry>
            <origin xyz="0 0 ${base_height/2}" rpy="0 0 0"/>
            <material name="green"/>
        </visual>
    </link>

    <!-- Gọi Macro sinh ra 2 bánh xe thay vì copy-paste -->
    <xacro:wheel_link prefix="right" />
    <xacro:wheel_link prefix="left" />

    <link name="caster_wheel">
        <visual>
            <geometry>
                <sphere radius="${wheel_radius/2}"/>
            </geometry>
            <origin xyz="0 0 0" rpy="0 0 0"/>
            <material name="blue"/>
        </visual>
    </link>

    <!-- Các Joints kết nối các Link -->
    <joint name="joint_base" type="fixed">
        <parent link="base"/>
        <child link="base_link"/>
        <origin xyz="0 0 0.1" rpy="0 0 0"/>
    </joint>

    <joint name="right_joint" type="continuous">
        <parent link="base_link"/>
        <child link="right_wheel"/>
        <origin xyz="-0.2 -0.225 0" rpy="0 0 0"/>
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

### File 3: `my_robot.urdf.xacro` (File chính)
Thay vì code cả ngàn dòng, file gốc của robot lúc này đóng vai trò như một bộ rễ gộp các thành phần lại với nhau bằng thẻ `<xacro:include>`.

```xml
<?xml version="1.0"?>
<robot name="my_robot" xmlns:xacro="http://www.ros.org/wiki/xacro">

    <!-- Nhúng file màu sắc -->
    <xacro:include filename="common_properties.xacro" />

    <!-- Nhúng file khung gầm -->
    <xacro:include filename="mobile_base.xacro" />

    <!-- Sau này có thể nhúng thêm Lidar, Camera, Arm... -->
    <!-- <xacro:include filename="lidar.xacro" /> -->

</robot>
```

Khi chạy Launch file (như ở [Tutorial 18](/posts/2026/11/ros2-tutorial-18/)), bộ parser `xacro` sẽ tự động đọc file `my_robot.urdf.xacro`, gộp `common_properties` và `mobile_base` lại, giải mã các biến `${}` thành số thực, và dịch ra thành một bản XML/URDF thuần túy để nạp vào `robot_state_publisher`.

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 16 - Viết bản thiết kế URDF đầu tiên](/posts/2026/11/ros2-tutorial-16/) | [Bài tiếp: Tutorial 18 - Hiển thị robot trên RViz2 ➡](/posts/2026/11/ros2-tutorial-18/)
