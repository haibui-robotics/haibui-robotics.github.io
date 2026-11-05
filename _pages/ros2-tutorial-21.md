---
layout: single
title: "Tutorial 21: Làm cho robot di chuyển trong Gazebo (Plugins & ros_gz_bridge – Dự án amr_ws)"
permalink: /posts/2026/11/ros2-tutorial-21/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 20 - Cài đặt Gazebo Sim & Spawn robot](/posts/2026/11/ros2-tutorial-20/) | [Bài tiếp: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo ➡](/posts/2026/11/ros2-tutorial-22/)

---

## 1. Mục tiêu và Khái niệm cốt lõi

Ở [Tutorial 20](/posts/2026/11/ros2-tutorial-20/), chúng ta đã thả thành công mô hình xe tự hành **`amr_description`** vào bản đồ nhà kho trong Gazebo Sim. Tuy nhiên, lúc này robot mới chỉ đứng yên như một khối vật lý tĩnh. Để điều khiển xe chạy bằng bàn phím hoặc thuật toán tự hành, ta cần giải quyết 3 bước trong dự án **[`amr_ws`](/portfolio/amr-ws/)**:

1. **Chuẩn hóa Khớp bánh xe (`amr.urdf.xacro`):** Sửa lỗi giới hạn vận tốc `velocity="0"` và chiều trục quay do công cụ xuất CAD (SolidWorks to URDF) sinh ra.
2. **Gắn Gazebo System Plugins (`gazebo_sim_controll.xacro`):** Tích hợp bộ truyền động vi sai `DiffDrive`, bộ đọc góc khớp bánh xe `JointStatePublisher` và giảm ma sát cho 2 bánh tự lựa (*caster wheels*).
3. **Mở Cầu nối `ros_gz_bridge` (`gazebo_bridge.yaml` & `gazebo.launch.xml`):** Chuyển đổi các gói tin điều khiển (`/cmd_vel`), vị trí (`/odom`, `/tf`, `/joint_states`) và cảm biến giữa Gazebo Sim và ROS 2.

---

## 2. Bước 1: Hai chỉnh sửa bắt buộc trong `amr.urdf.xacro` (Khi xuất từ SolidWorks)

Khi xuất mô hình từ SolidWorks sang URDF, các khớp xoay liên tục (`continuous`) của 2 bánh xe thường mắc **2 lỗi ngầm** khiến robot không thể chạy đúng trong Gazebo:

1. **Khóa cứng vận tốc (`velocity="0"`):** Mặc định SolidWorks xuất ra `<limit effort="10" velocity="0" />`. Trong Gazebo Sim, `velocity="0"` có nghĩa là **vận tốc tối đa bằng 0 rad/s** (bánh xe bị phanh cứng hoàn toàn!). Ta bắt buộc phải tăng lên `velocity="100"`.
2. **Ngược chiều trục quay (`<axis>`):** Trong bản xuất gốc, `right_wheel_joint` có `<axis xyz="0 -1 0" />` trong khi `left_wheel_joint` là `<axis xyz="0 1 0" />`. Khi plugin `DiffDrive` cấp tốc độ dương cho cả 2 bánh để đi thẳng, 2 bánh sẽ quay ngược chiều nhau làm xe tự xoay tròn tại chỗ. Ta cần đồng bộ cả 2 bánh về `<axis xyz="0 1 0" />`.

Đoạn cấu hình khớp bánh xe đã chuẩn hóa trong `urdf/amr.urdf.xacro`:

```xml
  <!-- Khớp bánh phải -->
  <joint name="right_wheel_joint" type="continuous">
    <origin xyz="-0.01801 -0.073 0.025476" rpy="0 0 0" />
    <parent link="base_link" />
    <child link="right_wheel_link" />
    <!-- [SỬA 1] Đồng bộ trục quay (0 1 0) cùng chiều với bánh trái -->
    <axis xyz="0 1 0" />
    <!-- [SỬA 2] Tăng giới hạn velocity từ 0 lên 100 rad/s để mở khóa bánh xe -->
    <limit effort="10" velocity="100" />
    <dynamics damping="0.1" friction="0.1" />
  </joint>

  <!-- Khớp bánh trái -->
  <joint name="left_wheel_joint" type="continuous">
    <origin xyz="-0.018017 0.073 0.02548" rpy="0 0 0" />
    <parent link="base_link" />
    <child link="left_wheel_link" />
    <axis xyz="0 1 0" />
    <limit effort="10" velocity="100" />
    <dynamics damping="0.1" friction="0.1" />
  </joint>
```

Đồng thời, mở khóa 3 dòng `<xacro:include>` ở ngay đầu file `urdf/amr.urdf.xacro`:

```xml
<robot name="amr_description" xmlns:xacro="http://www.ros.org/wiki/xacro">
  <xacro:include filename="$(find amr_description)/urdf/lidar.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/imu.xacro"/>
  <xacro:include filename="$(find amr_description)/urdf/gazebo_sim_controll.xacro"/>
```

---

## 3. Bước 2: Viết file `urdf/gazebo_sim_controll.xacro`

File này chứa cấu hình vật lý và các **System Plugins** điều khiển chuyển động cho robot trong Gazebo Sim:

* **Khoảng cách 2 bánh (`wheel_separation`):** Trong URDF, bánh trái đặt ở $y = +0.073\text{m}$ và bánh phải ở $y = -0.073\text{m}$ $\Rightarrow$ Khoảng cách tâm 2 bánh là $0.073 \times 2 = 0.146\text{m}$.
* **Chỉ định rõ tên Topic (`/cmd_vel`, `/odom`, `/tf`, `/joint_states`):** Giúp cấu hình `gazebo_bridge.yaml` ngắn gọn, không bị phụ thuộc vào tên model dài dòng của Gazebo.

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <!-- 1. Plugin Điều khiển xe 2 bánh vi sai (Differential Drive) -->
    <gazebo>
        <plugin filename="gz-sim-diff-drive-system"
                name="gz::sim::systems::DiffDrive">
            <left_joint>left_wheel_joint</left_joint>
            <right_joint>right_wheel_joint</right_joint>
            <!-- Khoảng cách 2 bánh theo URDF (y = +-0.073m => 0.146m) và bán kính bánh xe (~0.03m) -->
            <wheel_separation>0.146</wheel_separation>
            <wheel_radius>0.03</wheel_radius>
            <topic>/cmd_vel</topic>
            <odom_topic>/odom</odom_topic>
            <tf_topic>/tf</tf_topic>

            <frame_id>odom</frame_id>
            <child_frame_id>base_footprint</child_frame_id>
        </plugin>
    </gazebo>

    <!-- 2. Plugin phát góc quay thực tế của 2 bánh xe lên /joint_states -->
    <gazebo>
        <plugin filename="gz-sim-joint-state-publisher-system"
                name="gz::sim::systems::JointStatePublisher">
            <!-- Chỉ định tên topic cố định là /joint_states để ros_gz_bridge chuyển về cho robot_state_publisher -->
            <topic>/joint_states</topic>
            <joint_name>left_wheel_joint</joint_name>
            <joint_name>right_wheel_joint</joint_name>
        </plugin>
    </gazebo>

    <!-- 3. Giảm ma sát cho 2 bánh tự lựa (caster) vì joint đang để là fixed, giúp robot trượt mượt trên sàn -->
    <gazebo reference="caster1_wheel_link">
        <mu1>0.001</mu1>
        <mu2>0.001</mu2>
    </gazebo>
    <gazebo reference="caster2_wheel_link">
        <mu1>0.001</mu1>
        <mu2>0.001</mu2>
    </gazebo>

</robot>
```

---

## 4. Bước 3: Cấu hình Cầu nối `config/gazebo_bridge.yaml` và `gazebo.launch.xml`

### A. File `config/gazebo_bridge.yaml`
Khai báo bảng ánh xạ giữa các Topic của ROS 2 và Gazebo Sim (lưu ý tiền tố kiểu dữ liệu Protobuf trong Gazebo Sim là `gz.msgs.*`):

```yaml
# 1. Đồng bộ thời gian mô phỏng
- ros_topic_name: "/clock"
  gz_topic_name: "/clock"
  ros_type_name: "rosgraph_msgs/msg/Clock"
  gz_type_name: "gz.msgs.Clock"
  direction: GZ_TO_ROS

# 2. Ảnh từ Camera sensor
- ros_topic_name: "/camera/image_raw"
  gz_topic_name: "/camera/image_raw"
  ros_type_name: "sensor_msgs/msg/Image"
  gz_type_name: "gz.msgs.Image"
  direction: GZ_TO_ROS

# 3. Thông số nội tại Camera
- ros_topic_name: "/camera/camera_info"
  gz_topic_name: "/camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"
  direction: GZ_TO_ROS

# 4. Dữ liệu quét LiDAR 2D
- ros_topic_name: "/scan"
  gz_topic_name: "/scan"
  ros_type_name: "sensor_msgs/msg/LaserScan"
  gz_type_name: "gz.msgs.LaserScan"
  direction: GZ_TO_ROS

# 5. Dữ liệu cảm biến quán tính IMU
- ros_topic_name: "/imu/out"
  gz_topic_name: "/imu"
  ros_type_name: "sensor_msgs/msg/Imu"
  gz_type_name: "gz.msgs.IMU"
  direction: GZ_TO_ROS

# 6. Truyền lệnh điều khiển vận tốc từ ROS 2 sang plugin DiffDrive trong Gazebo
- ros_topic_name: "/cmd_vel"
  gz_topic_name: "/cmd_vel"
  ros_type_name: "geometry_msgs/msg/Twist"
  gz_type_name: "gz.msgs.Twist"
  direction: ROS_TO_GZ

# 7. Truyền dữ liệu Odometry từ Gazebo sang ROS 2
- ros_topic_name: "/odom"
  gz_topic_name: "/odom"
  ros_type_name: "nav_msgs/msg/Odometry"
  gz_type_name: "gz.msgs.Odometry"
  direction: GZ_TO_ROS

# 8. Truyền biến đổi tọa độ TF (odom -> base_footprint) từ Gazebo sang ROS 2
- ros_topic_name: "/tf"
  gz_topic_name: "/tf"
  ros_type_name: "tf2_msgs/msg/TFMessage"
  gz_type_name: "gz.msgs.Pose_V"
  direction: GZ_TO_ROS

# 9. Truyền góc quay 2 bánh xe về ROS 2 để robot_state_publisher tính TF bánh xe (hết lỗi TF trên RViz)
- ros_topic_name: "/joint_states"
  gz_topic_name: "/joint_states"
  ros_type_name: "sensor_msgs/msg/JointState"
  gz_type_name: "gz.msgs.Model"
  direction: GZ_TO_ROS
```

### B. Mở khóa `ros_gz_bridge` trong `launch/gazebo.launch.xml`
Trong file `launch/gazebo.launch.xml` (đã tạo ở Tutorial 20), ta bỏ chú thích cho biến `gazebo_config_path` và node `parameter_bridge`:

```xml
    <let name="gazebo_config_path" value="$(find-pkg-share amr_description)/config/gazebo_bridge.yaml"/>
    ...
    <!-- Cầu nối ROS 2 <-> Gazebo -->
    <node pkg="ros_gz_bridge" exec="parameter_bridge" output="screen">
        <param name="config_file" value="$(var gazebo_config_path)"/>
        <param name="use_sim_time" value="true"/>
    </node>
```

---

## 5. Nguồn Tra cứu Chính thức: Tìm Tên Plugins & Kiểu Tin nhắn ở đâu?

Khi phát triển các loại robot khác nhau (xe 4 bánh Mecanum, xe lái Ackermann, cánh tay robot, drone...) hoặc gắn thêm cảm biến mới, bạn có thể tra cứu trực tiếp tại **3 nguồn tài liệu chính thức** sau của Open Robotics / Gazebo:

### 1. Danh sách toàn bộ System Plugins của Gazebo Sim (`gz-sim`)
* 🔗 **Kho mã nguồn tất cả Plugins (`src/systems`):**  
  [https://github.com/gazebosim/gz-sim/tree/gz-sim8/src/systems](https://github.com/gazebosim/gz-sim/tree/gz-sim8/src/systems)  
  *(Mẹo: Khi bấm vào thư mục của bất kỳ plugin nào — ví dụ `diff_drive`, `mecanum_drive`, `ackermann_steering`, `joint_state_publisher`, `odometry_publisher`, `imu`... — hãy mở file header `.hh` của nó. Ngay ở phần đầu file luôn có hướng dẫn chi tiết tất cả các thẻ XML `<plugin>` mà plugin đó hỗ trợ!)*
* 🔗 **Trang tra cứu API Gazebo Sim Systems:**  
  [https://gazebosim.org/api/sim/8/namespacegz_1_1sim_1_1systems.html](https://gazebosim.org/api/sim/8/namespacegz_1_1sim_1_1systems.html)

| Chức năng | `filename` | `name` |
| :--- | :--- | :--- |
| Điều khiển xe 2 bánh vi sai | `gz-sim-diff-drive-system` | `gz::sim::systems::DiffDrive` |
| Điều khiển xe 4 bánh Mecanum | `gz-sim-mecanum-drive-system` | `gz::sim::systems::MecanumDrive` |
| Điều khiển xe kiểu ô tô (Ackermann) | `gz-sim-ackermann-steering-system` | `gz::sim::systems::AckermannSteering` |
| Xuất góc khớp (`/joint_states`) | `gz-sim-joint-state-publisher-system` | `gz::sim::systems::JointStatePublisher` |
| Điều khiển vị trí khớp (Arm/Servo) | `gz-sim-joint-position-controller-system` | `gz::sim::systems::JointPositionController` |
| Điều khiển vận tốc khớp | `gz-sim-joint-controller-system` | `gz::sim::systems::JointController` |
| Hệ thống Cảm biến (LiDAR/Camera) | `gz-sim-sensors-system` | `gz::sim::systems::Sensors` |
| Hệ thống Cảm biến quán tính IMU | `gz-sim-imu-system` | `gz::sim::systems::Imu` |

### 2. Đặc tả các thẻ `<sensor>` chuẩn SDFormat
* 🔗 **SDFormat `<sensor>` Specification:**  
  [http://sdformat.org/spec?ver=1.11&elem=sensor](http://sdformat.org/spec?ver=1.11&elem=sensor)  
  *(Tra cứu cấu trúc các thẻ con của `gpu_lidar`, `camera`, `rgbd_camera`, `imu`, `navsat`...)*

### 3. Bảng tra cứu kiểu dữ liệu `ros_gz_bridge` (ROS 2 ↔ Gazebo)
* 🔗 **Tài liệu chính thức `ros_gz_bridge`:**  
  [https://github.com/gazebosim/ros_gz/tree/ros2/ros_gz_bridge](https://github.com/gazebosim/ros_gz/tree/ros2/ros_gz_bridge)  
  *(Chứa bảng đối chiếu toàn bộ `ros_type_name` và `gz_type_name` được hỗ trợ).*

### 4. Tra cứu nhanh bằng lệnh Terminal khi đang chạy mô phỏng
* **Phía Gazebo:**
  ```bash
  gz topic -l                        # Liệt kê danh sách các topic trong Gazebo
  gz topic -i -t /cmd_vel            # Xem kiểu dữ liệu (gz_type_name) của một topic
  ```
* **Phía ROS 2:**
  ```bash
  ros2 topic list                    # Liệt kê danh sách các topic bên ROS 2
  ros2 topic info /cmd_vel           # Xem kiểu dữ liệu (ros_type_name) của một topic
  ```

---

## 6. Chạy thử nghiệm điều khiển Robot trong Gazebo

1. **Build và khởi chạy mô phỏng:**
   ```bash
   cd ~/amr_ws
   colcon build --packages-select amr_description
   source install/setup.bash
   ros2 launch amr_description gazebo.launch.xml
   ```
2. **Mở một Terminal thứ 2 để lái robot bằng bàn phím:**
   ```bash
   ros2 run teleop_twist_keyboard teleop_twist_keyboard
   ```
   Nhấn các phím `i` (tiến), `,` (lùi), `j` (quay trái), `l` (quay phải) — bạn sẽ thấy chiếc xe AMR di chuyển mượt mà trong nhà kho Gazebo và mô hình trên RViz2 cũng lăn bánh đồng bộ theo thời gian thực!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 20 - Cài đặt Gazebo Sim & Spawn robot](/posts/2026/11/ros2-tutorial-20/) | [Bài tiếp: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo ➡](/posts/2026/11/ros2-tutorial-22/)
