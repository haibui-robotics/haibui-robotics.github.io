---
layout: single
title: "Tutorial 19: Chuẩn bị cho Gazebo (Khối lượng - Inertia & Khung va chạm - Collision)"
permalink: /posts/2026/11/ros2-tutorial-19/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 18 - Hiển thị robot trên RViz2](/posts/2026/11/ros2-tutorial-18/)

---

## 1. Sự khác biệt giữa RViz2 và Gazebo

Như chúng ta đã biết ở bài trước, **RViz2** chỉ là công cụ để hiển thị (*visualization*). RViz2 không có khái niệm về thế giới vật lý: không có trọng lực, không có ma sát, và các vật thể có thể đi xuyên qua nhau.

Ngược lại, **Gazebo** là một môi trường mô phỏng vật lý (*simulation*). Khi bạn thả một robot vào Gazebo, nó sẽ chịu tác động của trọng lượng, lực cản và sự va chạm. Để robot tồn tại được trong Gazebo mà không bị rơi tự do xuyên qua mặt đất hay bay lơ lửng, file Xacro/URDF của chúng ta bắt buộc phải có thêm 2 thuộc tính vật lý cho mỗi bộ phận (`<link>`):

* **`<collision>` (Khung va chạm):** Định nghĩa ranh giới vật lý của vật thể để Gazebo tính toán va chạm. Nếu không có `<collision>`, Gazebo coi vật thể này là *"bóng ma"*, robot của bạn sẽ rớt thẳng xuống lòng đất!
* **`<inertial>` (Khối lượng và Quán tính):** Khai báo khối lượng (`<mass>`) và moment quán tính (`<inertia>`) của vật thể. Điều này giúp Gazebo biết vật thể nặng bao nhiêu và lực cần thiết để làm nó xoay hay dịch chuyển.

---

## 2. Định nghĩa Moment Quán tính (Inertia)

**Moment quán tính (Inertia Matrix)** là một ma trận $3 \times 3$ đặc trưng cho mức độ chống lại sự thay đổi vận tốc góc của một vật thể quay quanh một trục. Tùy thuộc vào hình học của vật thể (hình hộp, hình trụ, hình cầu), công thức tính ma trận này sẽ khác nhau (bạn có thể tra cứu trên Wikipedia: *List of moments of inertia*).

Vì công thức tính khá phức tạp và dài dòng, **Xacro** lại phát huy sức mạnh! Chúng ta sẽ viết các **Macro** (hàm) để tính toán quán tính tự động dựa trên khối lượng và kích thước.

---

## 3. Thực hành: Nâng cấp mã Xacro cho vật lý

### Bước 1: Thêm hàm tính Inertia vào `common_properties.xacro`
Mở file `common_properties.xacro` và thêm hàm `box_inertia` để tính ma trận quán tính cho khối hình hộp (dùng cho thân xe).

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
    
    <material name="green">
        <color rgba="0 0.5 0.2 1"/>
    </material>
    <material name="blue">
        <color rgba="0 0 0.5 1"/>
    </material>

    <!-- Macro tính moment quán tính cho hình hộp chữ nhật -->
    <xacro:macro name="box_inertia" params="m x y z o_xyz o_rpy">
        <inertial>
            <mass value="${m}"/>
            <origin xyz="${o_xyz}" rpy="${o_rpy}"/>
            <inertia ixx="${(m/12)*(y*y + z*z)}" ixy="0" ixz="0"
                     iyy="${(m/12)*(x*x + z*z)}" iyz="0"
                     izz="${(m/12)*(x*x + y*y)}"/>
        </inertial>
    </xacro:macro>
    
    <!-- Bạn có thể tạo thêm cylinder_inertia và sphere_inertia tương tự... -->

</robot>
```

### Bước 2: Bổ sung Collision và Inertia vào `mobile_base.xacro`
Bây giờ, chúng ta áp dụng `<collision>` và gọi hàm `box_inertia` cho thân xe (`base_link`).

```xml
<!-- ... (các biến kích thước) ... -->

<link name="base"/>

<link name="base_link">
    <visual>
        <geometry>
            <box size="0.6 0.4 0.2"/>
        </geometry>
        <origin xyz="0 0 0.1" rpy="0 0 0"/>
        <material name="green"/>
    </visual>
    
    <!-- 1. Thêm khung va chạm (Giống hệt visual) -->
    <collision>
        <geometry>
            <box size="0.6 0.4 0.2"/>
        </geometry>
        <origin xyz="0 0 0.1" rpy="0 0 0"/>
    </collision>
    
    <!-- 2. Gọi Macro tính khối lượng và quán tính -->
    <xacro:box_inertia m="5.0" 
                       x="${base_length}" 
                       y="${base_width}" 
                       z="${base_height}" 
                       o_xyz="0 0 ${base_height/2.0}" 
                       o_rpy="0 0 0" />
</link>
```

Bạn sẽ lặp lại quá trình này: Thêm thẻ `<collision>` và gọi Macro quán tính tương ứng cho các bánh xe (`right_wheel`, `left_wheel`, `caster_wheel`). Khi mọi bộ phận (*Link*) đều đã có đủ khối lượng và khung va chạm, robot của bạn đã sẵn sàng để được đưa vào **Gazebo** ở bài học tiếp theo!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 18 - Hiển thị robot trên RViz2](/posts/2026/11/ros2-tutorial-18/)
