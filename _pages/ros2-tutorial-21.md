---
layout: single
title: "Tutorial 21: Làm cho robot di chuyển trong Gazebo (Plugins & ros_gz_bridge – Dự án amr_ws)"
permalink: /posts/2026/11/ros2-tutorial-21/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 20 - Cài đặt Gazebo Sim & Spawn robot](/posts/2026/11/ros2-tutorial-20/) | [Bài tiếp: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo ➡](/posts/2026/11/ros2-tutorial-22/)

---

## 1. Mục tiêu và Khái niệm cốt lõi

Ở [Tutorial 20](/posts/2026/11/ros2-tutorial-20/) *(commit `V1-rviz+spawn gazebo`)*, chúng ta đã thả thành công mô hình xe tự hành **`amr_description`** vào bản đồ nhà kho trong Gazebo Sim. Tuy nhiên, lúc này robot mới chỉ đứng yên như một khối vật lý tĩnh.

Trong bài này *(tương ứng với commit **`V2-bridge+plugin`** trong dự án **[`amr_ws`](/portfolio/amr-ws/)***)*, chúng ta chỉ tập trung vào bài toán **làm cho robot di chuyển và đồng bộ chuyển động lên RViz2** qua 3 bước:

1. **Chuẩn hóa Khớp bánh xe (`amr.urdf.xacro`):** Sửa lỗi giới hạn vận tốc `velocity="0"` và chiều trục quay do công cụ xuất CAD (SolidWorks to URDF) sinh ra, đồng thời nhúng file plugin điều khiển.
2. **Gắn Gazebo System Plugins (`gazebo_sim_controll.xacro`):** Tích hợp bộ truyền động vi sai `DiffDrive`, bộ đọc góc khớp bánh xe `JointStatePublisher` và giảm ma sát cho 2 bánh tự lựa (*caster wheels*).
3. **Thiết lập Cầu nối `ros_gz_bridge` (`gazebo_bridge.yaml` & `gazebo.launch.xml`):** Chuyển đổi 5 kênh dữ liệu cốt lõi (`/clock`, `/cmd_vel`, `/odom`, `/tf`, `/joint_states`) giữa Gazebo Sim và ROS 2.

> 💡 **Ghi nhớ quan trọng – 2 Cách điều khiển xe trên Gazebo:**  
> Để làm robot di chuyển trong Gazebo, chúng ta có **2 cách tiếp cận**:
> * **Cách 1 (Bài học này): Dùng Gazebo Plugin (`DiffDrive`) + `ros_gz_bridge`** — Gọn nhẹ, dễ hiểu, giúp xe chạy ngay trên môi trường ảo nhưng **chỉ dùng được trong mô phỏng**.
> * **Cách 2 ([Tutorial 30](/posts/2026/11/ros2-tutorial-30/)): Dùng hệ thống `ros2_control`** — Chuẩn công nghiệp giúp bạn dùng chung 1 bộ code điều khiển cho **cả Mô phỏng Gazebo lẫn Phần cứng Robot thật (ESP32/STM32)**.

---

## 2. Bước 1: Chuẩn hóa khớp bánh xe trong `urdf/amr.urdf.xacro`

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

Đồng thời, nhúng file `gazebo_sim_controll.xacro` ở ngay đầu file `urdf/amr.urdf.xacro`:

```xml
<robot name="amr_description" xmlns:xacro="http://www.ros.org/wiki/xacro">
  <xacro:include filename="$(find amr_description)/urdf/gazebo_sim_controll.xacro"/>
```

---

## 3. Bước 2: Viết file `urdf/gazebo_sim_controll.xacro` (Commit V2)

File này chứa cấu hình vật lý và 2 **System Plugins** điều khiển chuyển động cho robot trong Gazebo Sim:

* **Khoảng cách 2 bánh (`wheel_separation`):** Trong URDF, bánh trái đặt ở $y = +0.073\text{m}$ và bánh phải ở $y = -0.073\text{m}$ $\Rightarrow$ Khoảng cách tâm 2 bánh là $0.073 \times 2 = 0.146\text{m}$, bán kính bánh xe `wheel_radius` là $0.03\text{m}$.
* **Khai báo Topic phía Gazebo (`/plugin/cmd_vel`, `/plugin/joint_states`, `/odom`, `/tf`):** Đặt tên rõ ràng cho từng kênh bên trong Gazebo để lát nữa kết nối qua `gazebo_bridge.yaml`.

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

    <gazebo>
        <plugin filename="gz-sim-diff-drive-system"
                name="gz::sim::systems::DiffDrive">
            <left_joint>left_wheel_joint</left_joint>
            <right_joint>right_wheel_joint</right_joint>
            <wheel_separation>0.146</wheel_separation>
            <wheel_radius>0.03</wheel_radius>
            <topic>/plugin/cmd_vel</topic>
            <odom_topic>/odom</odom_topic>
            <tf_topic>/tf</tf_topic>

            <frame_id>odom</frame_id>
            <child_frame_id>base_footprint</child_frame_id>
        </plugin>
    </gazebo>
    <gazebo>
        <plugin filename="gz-sim-joint-state-publisher-system"
                name="gz::sim::systems::JointStatePublisher">
            <topic>/plugin/joint_states</topic>
            <joint_name>left_wheel_joint</joint_name>
            <joint_name>right_wheel_joint</joint_name>
        </plugin>
    </gazebo>

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

* **Plugin diff-drive:** [https://github.com/gazebosim/gz-sim/blob/main/src/systems/diff_drive/DiffDrive.hh](https://github.com/gazebosim/gz-sim/blob/main/src/systems/diff_drive/DiffDrive.hh)
* **Plugin joint-state-publisher:** [https://github.com/gazebosim/gz-sim/blob/main/src/systems/joint_state_publisher/JointStatePublisher.hh](https://github.com/gazebosim/gz-sim/blob/main/src/systems/joint_state_publisher/JointStatePublisher.hh)

---

## 4. Bước 3: Cấu hình Cầu nối `config/gazebo_bridge.yaml` và `launch/gazebo.launch.xml` (Commit V2)

### A. File `config/gazebo_bridge.yaml`
Ở giai đoạn điều khiển chuyển động này, chúng ta chỉ cần khai báo đúng **5 cầu nối** giữa ROS 2 và Gazebo Sim:
* `/clock`: Đồng bộ thời gian mô phỏng.
* `/cmd_vel` (ROS 2) $\rightarrow$ `/plugin/cmd_vel` (Gazebo): Truyền lệnh điều khiển vận tốc xuống plugin `DiffDrive`.
* `/odom` & `/tf`: Truyền vị trí và biến đổi tọa độ `odom` $\rightarrow$ `base_footprint` từ Gazebo về ROS 2.
* `/plugin/joint_states` (Gazebo) $\rightarrow$ `/joint_states` (ROS 2): Truyền góc quay thực tế của 2 bánh xe về cho `robot_state_publisher` tính TF bánh xe.

```yaml
- ros_topic_name: "/clock"
  gz_topic_name: "/clock"
  ros_type_name: "rosgraph_msgs/msg/Clock"
  gz_type_name: "gz.msgs.Clock"
  direction: GZ_TO_ROS

- ros_topic_name: "/cmd_vel"
  gz_topic_name: "/plugin/cmd_vel"
  ros_type_name: "geometry_msgs/msg/Twist"
  gz_type_name: "gz.msgs.Twist"
  direction: ROS_TO_GZ

- ros_topic_name: "/odom"
  gz_topic_name: "/odom"
  ros_type_name: "nav_msgs/msg/Odometry"
  gz_type_name: "gz.msgs.Odometry"
  direction: GZ_TO_ROS

- ros_topic_name: "/tf"
  gz_topic_name: "/tf"
  ros_type_name: "tf2_msgs/msg/TFMessage"
  gz_type_name: "gz.msgs.Pose_V"
  direction: GZ_TO_ROS

- ros_topic_name: "/joint_states"
  gz_topic_name: "/plugin/joint_states"
  ros_type_name: "sensor_msgs/msg/JointState"
  gz_type_name: "gz.msgs.Model"
  direction: GZ_TO_ROS
```

* **Cách tra cứu tên Topic và kiểu dữ liệu bằng lệnh Terminal:**
```bash
ros2 topic list
ros2 topic info /clock              # ros2 topic info + tên topic
gz topic -l
gz topic --info --topic /clock      # gz topic --info --topic + tên topic
```

### B. Thêm `ros_gz_bridge` vào `launch/gazebo.launch.xml`
Trong file `launch/gazebo.launch.xml` (của commit `V2-bridge+plugin`), ta khai báo thêm biến `gazebo_config_path` và khởi chạy node `parameter_bridge`:

```xml
<launch>
    <arg name="use_gazebo" default="true"/>
    <arg name="use_rviz" default="true"/>
    <arg name="use_sim_time" default="true"/>

    <!-- Trỏ đường dẫn tài nguyên -->
    <set_env name="GZ_SIM_RESOURCE_PATH" value="/home/pc/amr_ws/src/amr_description/models:/home/pc/amr_ws/src"/>

    <let name="urdf_path" value="$(find-pkg-share amr_description)/urdf/amr.urdf.xacro"/>
    <let name="rviz_config_path" value="$(find-pkg-share amr_description)/rviz/config.rviz"/>
    <let name="gazebo_config_path" value="$(find-pkg-share amr_description)/config/gazebo_bridge.yaml"/>

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

    <!-- Cầu nối ROS 2 <-> Gazebo -->
    <node pkg="ros_gz_bridge" exec="parameter_bridge" output="screen">
        <param name="use_sim_time" value="$(var use_sim_time)"/>
        <param name="config_file" value="$(var gazebo_config_path)"/>
    </node>

    <node pkg="rviz2" exec="rviz2" name="rviz2" output="screen" args="-d $(var rviz_config_path)" if="$(var use_rviz)">
        <param name="use_sim_time" value="$(var use_sim_time)"/>   
    </node>
</launch>
```

---

## 5. Nguồn Tra cứu Chính thức: Tìm Tên Plugins & Kiểu Tin nhắn ở đâu?

Khi phát triển các loại cơ cấu truyền động khác nhau (xe 2 bánh vi sai, xe 4 bánh Mecanum, xe lái Ackermann, cánh tay robot...), bạn có thể tra cứu trực tiếp tại các nguồn tài liệu chính thức sau của Open Robotics / Gazebo:

### 1. Danh sách toàn bộ System Plugins của Gazebo Sim (`gz-sim`)
* 🔗 **Kho mã nguồn tất cả Plugins (`src/systems`):**  
  [https://github.com/gazebosim/gz-sim/tree/gz-sim8/src/systems](https://github.com/gazebosim/gz-sim/tree/gz-sim8/src/systems)  
  *(Mẹo: Khi bấm vào thư mục của bất kỳ plugin nào — ví dụ `diff_drive`, `mecanum_drive`, `ackermann_steering`, `joint_state_publisher`, `odometry_publisher`... — hãy mở file header `.hh` của nó. Ngay ở phần đầu file luôn có hướng dẫn chi tiết tất cả các thẻ XML `<plugin>` mà plugin đó hỗ trợ!)*
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

### 2. Bảng tra cứu kiểu dữ liệu `ros_gz_bridge` (ROS 2 ↔ Gazebo)
* 🔗 **Tài liệu chính thức `ros_gz_bridge`:**  
  [https://github.com/gazebosim/ros_gz/tree/ros2/ros_gz_bridge](https://github.com/gazebosim/ros_gz/tree/ros2/ros_gz_bridge)  
  *(Chứa bảng đối chiếu toàn bộ `ros_type_name` và `gz_type_name` được hỗ trợ).*

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
   Nhấn các phím `i` (tiến), `,` (lùi), `j` (quay trái), `l` (quay phải) — lệnh vận tốc phát lên `/cmd_vel` sẽ đi qua `ros_gz_bridge` sang `/plugin/cmd_vel` của Gazebo, điều khiển chiếc xe AMR lăn bánh mượt mà trong nhà kho và đồng bộ vị trí trực tiếp trên RViz2!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 20 - Cài đặt Gazebo Sim & Spawn robot](/posts/2026/11/ros2-tutorial-20/) | [Bài tiếp: Tutorial 22 - Gắn cảm biến LiDAR 2D vào Gazebo ➡](/posts/2026/11/ros2-tutorial-22/)
