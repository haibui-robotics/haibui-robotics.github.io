---
layout: single
title: "Tutorial 18: Hiển thị robot trên RViz2 (robot_state_publisher)"
permalink: /posts/2026/11/ros2-tutorial-18/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 17 - Viết code thông minh hơn với Xacro](/posts/2026/11/ros2-tutorial-17/)

---

## 1. Xây dựng cấu trúc Package mô phỏng

Để chạy được mô phỏng và quản lý các file cấu hình, file launch, bạn cần tạo một package ROS 2 (ví dụ: `my_robot_description`) với cấu trúc thư mục chuẩn mực như sau. Hãy di chuyển file `my_robot.urdf` vừa tạo ở bài trước vào thư mục `urdf`.

```text
my_robot_description/
├── CMakeLists.txt / package.xml
├── urdf/
│   └── my_robot.urdf         # File mô tả cấu trúc robot
├── meshes/                   # (Tùy chọn) Chứa file 3D CAD (.stl, .dae) nếu có
├── launch/                   # Nơi chứa các file khởi chạy
└── rviz/                     # Nơi lưu cấu hình giao diện RViz2
```

---

## 2. Tại sao không thể ném thẳng file URDF vào RViz2?

**RViz2** là một phần mềm trực quan hóa dữ liệu. Nó không tự hiểu được cấu trúc cha-con (*Link-Joint*) tĩnh trong file URDF. Để RViz2 (hoặc bất kỳ thuật toán điều hướng nào) thấy được robot, hệ thống cần các hệ trục tọa độ **TF** liên tục phát trên mạng ROS.

Chúng ta cần 2 Node trung gian đóng vai trò biên dịch:

* **`robot_state_publisher`**: Đây là Node lõi do ROS 2 cung cấp. Nhiệm vụ của nó là đọc nội dung file URDF, tính toán **Động học thuận (Forward Kinematics)** và phát liên tục vị trí 3D chính xác của từng bộ phận lên cây tọa độ TF (`/tf` và `/tf_static`).
* **`joint_state_publisher_gui`**: Cung cấp một bảng điều khiển nhỏ có các thanh trượt (*slider*). Khi bạn kéo thanh trượt, Node này giả lập tín hiệu quay của động cơ và phát lên Topic `/joint_states`. `robot_state_publisher` nghe topic này để cập nhật chuyển động cho TF.

---

## 3. Viết Launch File để tự động hóa

Tạo một file `launch/display.launch.xml` để chạy cả `robot_state_publisher`, bảng GUI và RViz2 cùng một lúc, thay vì phải gõ từng lệnh trên Terminal.

```xml
<launch>
    <!-- 1. Tạo các biến đường dẫn tạm thời -->
    <!-- Hàm find-pkg-share giúp tự động tìm đúng đường dẫn tới thư mục cài đặt của package -->
    <let name="urdf_path" value="$(find-pkg-share my_robot_description)/urdf/my_robot.urdf" />
    <let name="rviz_config_path" value="$(find-pkg-share my_robot_description)/rviz/urdf_config.rviz"/>

    <!-- 2. Khởi chạy Node robot_state_publisher -->
    <node pkg="robot_state_publisher" exec="robot_state_publisher">
        <!-- Tham số robot_description BẮT BUỘC phải có. Lệnh 'xacro' sẽ đọc file URDF/Xacro và nhúng trực tiếp chuỗi XML vào tham số này -->
        <param name="robot_description" value="$(command 'xacro $(var urdf_path)')" />
    </node>

    <!-- 3. Khởi chạy Node joint_state_publisher_gui (Bảng điều khiển khớp xoay) -->
    <node pkg="joint_state_publisher_gui" exec="joint_state_publisher_gui"/>

    <!-- 4. Mở giao diện RViz2 -->
    <!-- args="-d ...": Yêu cầu RViz2 mở luôn file cấu hình đã lưu sẵn, đỡ mất công add lại RobotModel mỗi lần chạy -->
    <node pkg="rviz2" exec="rviz2" name="rviz2" output="screen" args="-d $(var rviz_config_path)"/>
</launch>
```

---

## 4. Thực hành kiểm tra

Sau khi đã có file URDF và Launch file, bạn cần cập nhật `CMakeLists.txt` (hoặc `setup.py`) để hệ thống copy thư mục `launch`, `urdf`, `rviz` vào không gian `install`. Sau đó làm theo các bước:

1. **Build lại package:**
   ```bash
   colcon build --packages-select my_robot_description
   ```
2. **Source môi trường:**
   ```bash
   source install/setup.bash
   ```
3. **Chạy launch file:**
   ```bash
   ros2 launch my_robot_description display.launch.xml
   ```

Lúc này, phần mềm **RViz2** sẽ hiện lên với mô hình 3D của robot. Kế bên là một bảng điều khiển, bạn có thể thử kéo các thanh trượt của `right_joint` và `left_joint` để thấy bánh xe quay trực tiếp trên màn hình.

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 17 - Viết code thông minh hơn với Xacro](/posts/2026/11/ros2-tutorial-17/)
