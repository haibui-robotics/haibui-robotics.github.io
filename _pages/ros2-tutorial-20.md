---
layout: single
title: "Tutorial 20: Cài đặt Gazebo Sim, World và 'Spawn' robot vào thế giới ảo (Dự án amr_ws)"
permalink: /posts/2026/11/ros2-tutorial-20/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 19 - Chuẩn bị cho Gazebo (Inertia & Collision)](/posts/2026/11/ros2-tutorial-19/) | [Bài tiếp: Tutorial 21 - Làm cho robot di chuyển trong Gazebo ➡](/posts/2026/11/ros2-tutorial-21/)

---

## 1. Mục tiêu và Khái niệm cốt lõi

Ở [Tutorial 19](/posts/2026/11/ros2-tutorial-19/), mô hình robot của chúng ta trong file `urdf/amr.urdf.xacro` đã có đầy đủ các thuộc tính vật lý: khối lượng (`<mass>`), ma trận quán tính (`<inertia>`), hình dáng hiển thị (`<visual>`) và khung va chạm (`<collision>`).

Trong bài này, chúng ta sẽ lấy trực tiếp gói **`amr_description`** thuộc dự án **[`amr_ws`](/portfolio/amr-ws/)** để đưa mô hình robot 3D xuất hiện trong thế giới ảo **Gazebo Sim** (*giai đoạn này chỉ tập trung thả robot vào thế giới ảo, chưa gắn các Plugin điều khiển hay cầu nối Bridge*).

Để làm được điều này, bạn cần nắm vững 3 khái niệm:
* **World (Thế giới ảo):** Gazebo lưu trữ môi trường mô phỏng (nhà kho, ngôi nhà, mặt phẳng trống) trong các file `.world` hoặc `.sdf` (ví dụ: `worlds/small_warehouse.world`).
* **Spawn (Thả robot vào Gazebo):** Trong ROS 2, ta sử dụng node `create` thuộc gói `ros_gz_sim` để đọc cấu trúc XML của robot từ Topic `/robot_description` (do `robot_state_publisher` phát ra) và thả robot xuống tọa độ mong muốn trong Gazebo.
* **Đồng bộ thời gian (`use_sim_time`):** Thời gian vật lý trong Gazebo có thể chạy nhanh hoặc chậm hơn đồng hồ thực của máy tính. Do đó, mọi Node (`robot_state_publisher`, `rviz2`) đều phải bật `use_sim_time = true` để đồng bộ theo đồng hồ của Gazebo.

---

## 2. Cấu trúc Package `amr_description` và Cài đặt Thư mục

Trong dự án `amr_ws`, gói `amr_description` chứa toàn bộ file thiết kế URDF/Xacro, file lưới 3D CAD (`meshes/*.STL`), các mô hình vật thể nhà kho (`models/`) và bản đồ thế giới ảo (`worlds/`):

```text
amr_ws/src/amr_description/
├── CMakeLists.txt
├── package.xml
├── urdf/
│   └── amr.urdf.xacro          # File mô tả cấu trúc 3D & vật lý của robot AMR
├── meshes/
│   ├── base_link.STL           # Khung gầm chính
│   ├── right_wheel_link.STL    # Bánh chủ động phải
│   ├── left_wheel_link.STL     # Bánh chủ động trái
│   ├── caster1_wheel_link.STL  # Bánh tự lựa trước
│   ├── caster2_wheel_link.STL  # Bánh tự lựa sau
│   ├── lidar_link.STL          # Cụm cảm biến LiDAR
│   ├── cam_link.STL            # Cụm Camera
│   └── imu_link.STL            # Chip IMU
├── models/                     # Chứa các mô hình kệ hàng, bàn ghế của nhà kho
├── worlds/
│   ├── empty.world
│   ├── small_house.world
│   └── small_warehouse.world   # Thế giới ảo nhà kho dùng để test AMR
├── rviz/
│   └── config.rviz
└── launch/
    ├── display.launch.xml
    └── gazebo.launch.xml       # Launch file khởi chạy Gazebo & Spawn robot
```

### Khai báo cài đặt thư mục trong `CMakeLists.txt`
Để ROS 2 tìm thấy các file `.STL`, `.world` và `.xacro` sau khi build, trong `CMakeLists.txt` ta cần copy toàn bộ các thư mục này vào `share/${PROJECT_NAME}`:

```cmake
cmake_minimum_required(VERSION 3.8)
project(amr_description)

find_package(ament_cmake REQUIRED)
find_package(rclpy REQUIRED)

install(DIRECTORY
  urdf
  launch
  meshes
  models
  worlds
  rviz
  photos
  config
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

### Khai báo phụ thuộc trong `package.xml`
Đảm bảo trong `package.xml` đã có các gói thực thi cho Gazebo Sim và URDF:

```xml
<exec_depend>robot_state_publisher</exec_depend>
<exec_depend>xacro</exec_depend>
<exec_depend>rviz2</exec_depend>
<exec_depend>ros_gz_sim</exec_depend>
<exec_depend>ros2launch</exec_depend>
```

---

## 3. Hai thiết lập môi trường cực kỳ quan trọng trong Launch File

Khi mở Gazebo Sim cùng mô hình CAD `.STL` trên Ubuntu, có 2 lỗi kinh điển mà người mới rất hay gặp:

### A. Lỗi Gazebo không hiển thị được file 3D `.STL` hoặc đồ vật trong World (`GZ_SIM_RESOURCE_PATH`)
Trong file `urdf/amr.urdf.xacro`, đường dẫn tới các file lưới 3D được viết theo chuẩn ROS:
```xml
<mesh filename="package://amr_description/meshes/base_link.STL" />
```
Để Gazebo Sim hiểu được `package://amr_description/...` nằm ở đâu trên ổ cứng, đồng thời tìm thấy các kệ hàng trong `small_warehouse.world`, ta cấu hình biến môi trường `GZ_SIM_RESOURCE_PATH` trỏ tới **cả 2 đường dẫn** (ngăn cách nhau bởi dấu hai chấm `:`):
1. `/home/pc/amr_ws/src/amr_description/models`: Để Gazebo tải các mô hình đồ vật trong nhà kho.
2. `/home/pc/amr_ws/src`: Để Gazebo ánh xạ `package://amr_description/meshes/...` vào đúng `/home/pc/amr_ws/src/amr_description/meshes/...`.

### B. Lỗi xung đột thư viện đồ họa GTK trên VS Code bản SNAP
Nếu bạn mở Terminal tích hợp bên trong VS Code (cài qua Snap trên Ubuntu) để chạy Gazebo/RViz2, VS Code sẽ tiêm các biến môi trường `GTK_PATH`, `LOCPATH`... gây văng ứng dụng (*symbol lookup error*). Ta dùng thẻ `<set_env>` đặt rỗng các biến này ngay đầu file Launch.

---

## 4. Mã nguồn `launch/gazebo.launch.xml` (Commit V1 – Giai đoạn Spawn Robot)

Dưới đây là toàn bộ nội dung file `amr_description/launch/gazebo.launch.xml` trong dự án `amr_ws` (commit `V1-rviz+spawn gazebo`) ở giai đoạn **Spawn mô hình vào Gazebo** (chưa sử dụng plugin điều khiển hay `ros_gz_bridge`):

```xml
<launch>
    <arg name="use_gazebo" default="true"/>
    <arg name="use_rviz" default="true"/>
    <arg name="use_sim_time" default="true"/>

    <!-- Trỏ đường dẫn tài nguyên -->
    <set_env name="GZ_SIM_RESOURCE_PATH" value="/home/pc/amr_ws/src/amr_description/models:/home/pc/amr_ws/src"/>

    <let name="urdf_path" value="$(find-pkg-share amr_description)/urdf/amr.urdf.xacro"/>
    <let name="rviz_config_path" value="$(find-pkg-share amr_description)/rviz/config.rviz"/>

    <node pkg="robot_state_publisher" exec="robot_state_publisher">
        <param name="robot_description" value="$(command 'xacro $(var urdf_path)')"/>
        <param name="use_sim_time" value="$(var use_sim_time)"/>
    </node>

    <!-- Khởi động Gazebo -->
    <include file="$(find-pkg-share ros_gz_sim)/launch/gz_sim.launch.py">
        <arg name="gz_args" value="$(find-pkg-share amr_description)/worlds/small_warehouse.world -r"/>
    </include>
    <!-- Spawn robot -->
    <node pkg="ros_gz_sim" exec="create" output="screen" args="-topic robot_description -name amr_description -z 0.1">
        <param name="use_sim_time" value="$(var use_sim_time)"/>
    </node>

    <node pkg="rviz2" exec="rviz2" name="rviz2" output="screen" args="-d $(var rviz_config_path)" if="$(var use_rviz)">
        <param name="use_sim_time" value="$(var use_sim_time)"/>   
    </node>

</launch>
<!-- 
Include gazebo + Chay 3 node:
+)robot_state_publisher -> truyen tham so robot_description tu duong dan toi file urdf (node nhan chuoi xml de mo ta,tinh toan vi tri, phat ra TF
+)ros_gz_sim -> spawn robot vao gazebo (node nhan chuoi xml de mo ta, spawn robot vao gazebo)
+)rviz2 -> hien thi robot   
-->
```

---

## 5. Thực hành Build, Khởi chạy & Mẹo xử lý lỗi

### Bước 1: Build và chạy thử mô phỏng
Mở Terminal tại thư mục gốc `~/amr_ws` và thực hiện:

```bash
colcon build --packages-select amr_description
source install/setup.bash
ros2 launch amr_description gazebo.launch.xml
```

### Bước 2: Quan sát kết quả
1. Cửa sổ **Gazebo Sim** mở lên với không gian nhà kho (`small_warehouse.world`).
2. Node `create` đọc cấu trúc từ `/robot_description` và thả chiếc xe tự hành **`amr_description`** (với đầy đủ khung xe, 2 bánh chủ động, 2 bánh caster, LiDAR, Camera, IMU từ file `.STL`) rơi nhẹ từ độ cao `z = 0.1m` xuống mặt sàn nhà kho.
3. Nhờ đã khai báo đầy đủ `<collision>` và `<inertial>` trong `amr.urdf.xacro`, robot đứng vững trên mặt sàn dưới tác dụng của trọng lực mà không bị xuyên qua đất hay lật nghiêng.

> 💡 **Mẹo thực tế (Pro-Tip):** Đôi khi bạn bấm `Ctrl + C` trên Terminal nhưng tiến trình máy chủ vật lý của Gazebo vẫn chạy ngầm khiến lần mở tiếp theo bị đen màn hình hoặc báo trùng tên robot. Hãy chạy lệnh sau để dọn sạch tiến trình Gazebo ngầm:
> ```bash
> pkill -f 'gz sim'
> ```

Ở bài tiếp theo ([Tutorial 21](/posts/2026/11/ros2-tutorial-21/)), chúng ta sẽ bắt đầu mở khóa các Plugin truyền động và cầu nối `ros_gz_bridge` để điều khiển chiếc xe AMR này lăn bánh trong nhà kho!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 19 - Chuẩn bị cho Gazebo (Inertia & Collision)](/posts/2026/11/ros2-tutorial-19/) | [Bài tiếp: Tutorial 21 - Làm cho robot di chuyển trong Gazebo ➡](/posts/2026/11/ros2-tutorial-21/)
