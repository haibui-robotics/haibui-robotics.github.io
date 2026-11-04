---
layout: single
title: "Tutorial 20: Cài đặt Gazebo Sim, World và 'Spawn' robot vào thế giới ảo"
permalink: /posts/2026/11/ros2-tutorial-20/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 19 - Chuẩn bị cho Gazebo (Inertia & Collision)](/posts/2026/11/ros2-tutorial-19/)

---

## 1. Mục tiêu và Khái niệm quan trọng

Ở bài trước, robot của chúng ta đã có đủ "thể xác" (khối lượng, khung va chạm). Bây giờ, chúng ta sẽ đưa nó vào Gazebo. Để làm được điều này thông qua Launch file, bạn cần hiểu 3 khái niệm cốt lõi:

* **World (Thế giới ảo):** Gazebo lưu trữ môi trường ảo (nhà kho, sa mạc, thành phố) trong các file có đuôi `.world` (hoặc `.sdf`).
* **Spawn (Thả robot):** Trong ROS 2, chúng ta dùng một Node có tên là `create` (thuộc gói `ros_gz_sim`) để "bắt" cấu trúc robot từ Topic `robot_description` (do `robot_state_publisher` phát ra) và thả nó vào tọa độ mong muốn trong Gazebo.
* **Đồng bộ thời gian (`use_sim_time`):** Thế giới vật lý trong Gazebo có thể chạy nhanh hoặc chậm hơn thời gian thực của máy tính. Bắt buộc phải cài đặt `use_sim_time = true` cho **TẤT CẢ** các Node (RViz, State Publisher, Bridge) để chúng sử dụng đồng hồ của Gazebo. Nếu không, hệ thống TF (tọa độ) sẽ bị lệch nhịp và báo lỗi.

---

## 2. Xử lý các lỗi môi trường phổ biến (Environment Variables)

Khi khởi chạy Gazebo và RViz2 cùng lúc, đặc biệt là nếu bạn code bằng VS Code bản SNAP trên Ubuntu, bạn rất dễ gặp lỗi đồ họa (GTK) hoặc lỗi Gazebo không load được file 3D (`.stl`, `.dae`).

Để khắc phục, chúng ta sử dụng thẻ `<set_env>` ngay đầu file Launch để cấu hình lại môi trường:

* Xóa các biến GTK xung đột.
* Trỏ biến `GZ_SIM_RESOURCE_PATH` về đúng thư mục chứa `models`/`meshes` của bạn để Gazebo có thể render hình ảnh.

---

## 3. Viết Launch File tổng hợp (Mô phỏng hoàn chỉnh)

Dưới đây là một file `gazebo.launch.xml` hoàn chỉnh. Nó sẽ kết nối toàn bộ kiến thức từ [Tutorial 16](/posts/2026/11/ros2-tutorial-16/) đến giờ: Mở thế giới ảo nhà kho, dựng URDF, thả robot, bật cầu nối Bridge và mở RViz2.

```xml
<launch>
    <!-- 1. Khai báo các tham số đầu vào (Arguments) -->
    <arg name="launch_rviz" default="true"/>
    <arg name="use_sim_time" default="true"/>

    <!-- 2. Loại bỏ biến môi trường do VS Code bản SNAP tiêm vào (Tránh lỗi văng GTK khi chạy RViz/Gazebo) -->
    <set_env name="GTK_PATH" value=""/>
    <set_env name="GTK_EXE_PREFIX" value=""/>
    <set_env name="GTK_IM_MODULE_FILE" value=""/>
    <set_env name="GIO_MODULE_DIR" value=""/>
    <set_env name="LOCPATH" value=""/>

    <!-- 3. Cấu hình đường dẫn tài nguyên cho Gazebo (Để Gazebo tìm thấy file 3D mesh của robot) -->
    <!-- Lưu ý: Thay đổi đường dẫn này cho khớp với máy của bạn -->
    <set_env name="GZ_SIM_RESOURCE_PATH" value="/home/pc/mobileRobot_ws/src/mobile_description/my_mobilerobot/models"/>
    
    <!-- Tạo các biến đường dẫn file -->
    <let name="urdf_path" value="$(find-pkg-share my_mobilerobot)/urdf/my_robot.urdf.xacro"/>
    <let name="rviz_config_path" value="$(find-pkg-share my_mobilerobot)/rviz/config.rviz"/>
    <let name="gazebo_config_path" value="$(find-pkg-share my_mobilerobot)/config/gazebo_bridge.yaml"/>

    <!-- 4. robot_state_publisher: Biên dịch URDF sang TF -->
    <node pkg="robot_state_publisher" exec="robot_state_publisher">
        <!-- Truyền thêm biến use_gazebo:=true vào xacro nếu bạn có dùng điều kiện if-else trong URDF -->
        <param name="robot_description" value="$(command 'xacro $(var urdf_path) use_gazebo:=true')"/>
        <param name="use_sim_time" value="$(var use_sim_time)"/>
    </node>

    <!-- 5. Khởi chạy Gazebo Sim với một World có sẵn (small_warehouse) -->
    <include file="$(find-pkg-share ros_gz_sim)/launch/gz_sim.launch.py">
        <!-- Thêm cờ '-r' để Gazebo tự động "Play" (chạy mô phỏng ngay) khi mở -->
        <arg name="gz_args" value="$(find-pkg-share my_mobilerobot)/worlds/small_warehouse.world -r"/>
    </include>

    <!-- 6. Spawn robot vào thế giới Gazebo -->
    <!-- Lấy mô hình từ topic, đặt tên là my_mobilerobot, thả ở độ cao Z = 0.1 mét (để không bị lún xuống sàn) -->
    <node pkg="ros_gz_sim" exec="create" args="-topic robot_description -name my_mobilerobot -z 0.1"/>

    <!-- 7. Cầu nối ROS 2 <-> Gazebo (ros_gz_bridge) -->
    <node pkg="ros_gz_bridge" exec="parameter_bridge" output="screen">
        <param name="config_file" value="$(var gazebo_config_path)"/>
        <param name="use_sim_time" value="$(var use_sim_time)"/>
    </node>

    <!-- 8. Nạp ros2_control (Sẽ tìm hiểu sâu ở Giai đoạn 3) -->
    <include file="$(find-pkg-share mobile_controller)/launch/controller.launch.xml">
        <arg name="use_gazebo" value="true"/>
        <arg name="use_sim_time" value="$(var use_sim_time)"/>
        <arg name="launch_rviz" value="false"/> <!-- Không bật rviz ở đây vì đã bật ở dưới -->
    </include>

    <!-- 9. Khởi chạy RViz2 -->
    <!-- Dùng thuộc tính 'if' để kiểm tra arg launch_rviz, nếu true mới bật -->
    <node pkg="rviz2" exec="rviz2" output="screen" args="-d $(var rviz_config_path)" if="$(var launch_rviz)">
        <param name="use_sim_time" value="$(var use_sim_time)"/>
    </node>
</launch>
```

---

## 4. Phân tích luồng hoạt động

Khi bạn chạy lệnh `ros2 launch my_mobilerobot gazebo.launch.xml`, những việc sau sẽ diễn ra:

1. Máy tính dọn dẹp biến môi trường để chống lỗi đồ họa.
2. `robot_state_publisher` đọc Xacro, tính toán ra TF và phát cấu trúc lên Topic.
3. Gazebo Sim mở lên tải bản đồ nhà kho (`small_warehouse.world`) và tự động bấm nút Play (`-r`).
4. Lệnh `create` chộp lấy cấu trúc robot trên Topic, thả cái *"Bịch!"* xuống độ cao `0.1m` trong nhà kho.
5. Cầu nối `parameter_bridge` mở ra, bắt đầu truyền dữ liệu qua lại giữa hai bên.
6. RViz2 mở lên, hiển thị tọa độ và hình ảnh mô phỏng. *(Bởi vì ta để `use_sim_time="true"`, RViz2 và Gazebo sẽ "đập chung một nhịp tim" thời gian).*

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 19 - Chuẩn bị cho Gazebo (Inertia & Collision)](/posts/2026/11/ros2-tutorial-19/)
