---
layout: single
title: "Tutorial 30: Giới thiệu ros2_control và tích hợp Bộ điều khiển vào Gazebo (Dự án mobileRobot_ws)"
permalink: /posts/2026/11/ros2-tutorial-30/
author_profile: true
---

[⬅ Quay lại bài: Động học, Odometry và Điều khiển Robot (ros2_control)](/posts/2026/11/dong-hoc-odometry-ros2-control/)

---

## 1. Lý thuyết cốt lõi: Hai cách để làm xe di chuyển trên Gazebo

Khi lập trình robot di động trong ROS 2 và Gazebo Sim, bạn cần nắm thật rõ rằng có **2 cách tiếp cận hoàn toàn khác nhau** để điều khiển bánh xe lăn bánh:

| Tiêu chí so sánh | Cách 1: Gazebo Plugin + `ros_gz_bridge`<br>*(Đã học ở [Tutorial 21](/posts/2026/11/ros2-tutorial-21/))* | Cách 2: Hệ thống `ros2_control`<br>*(Nội dung bài học này)* |
| :--- | :--- | :--- |
| **Cơ chế hoạt động** | Gazebo tự tính động học bằng plugin `gz-sim-diff-drive-system` và `gz-sim-joint-state-publisher-system`, sau đó đẩy `/odom`, `/tf`, `/joint_states` qua cầu nối `ros_gz_bridge` sang ROS 2. | ROS 2 nắm quyền điều khiển trực tiếp từng khớp bánh xe thông qua `controller_manager` và plugin cầu nối phần cứng `gz_ros2_control`. |
| **Phạm vi sử dụng** | **Chỉ chạy được trên Mô phỏng Gazebo.** Không thể mang xuống chạy trên mạch điều khiển thực tế. | **Vừa chạy được trên Mô phỏng Gazebo (`use_gazebo:=true`), vừa chạy được trên Phần cứng thật (`use_gazebo:=false` – ESP32/STM32).** |
| **Cấu hình `gazebo_bridge.yaml`** | Phải bridge cả `/cmd_vel`, `/odom`, `/tf`, `/joint_states` và `/clock`. | **Không cần** bridge `/cmd_vel`, `/odom`, `/tf`, `/joint_states` nữa (vì `ros2_control` đã xử lý trực tiếp bên trong ROS 2). Chỉ giữ lại `/clock` và các cảm biến (`/scan`, `/camera`, `/imu`). |
| **Khi nào nên dùng?** | Khi mới làm quen với Gazebo hoặc muốn test nhanh cảm biến, SLAM, Nav2 thuần mô phỏng. | Khi xây dựng **dự án Robot thực tế hoàn chỉnh** cần đồng nhất 100% thuật toán giữa Mô phỏng (Sim) và Đời thực (Real). |

> ⚠️ **Nguyên tắc quan trọng:** Trong một thời điểm, bạn **chỉ sử dụng 1 trong 2 cách**. Khi chuyển sang dùng **`ros2_control` (Cách 2)**, bạn phải đóng chú thích (hoặc xóa) 2 plugin `DiffDrive` và `JointStatePublisher` của Cách 1 đi để tránh 2 bộ điều khiển cùng tranh giành bánh xe!

### Sơ đồ kiến trúc của Cách 2 (`ros2_control` – Chạy cả Mô phỏng lẫn Phần cứng thật):

```text
           [ Thuật toán Điều hướng (Nav2) / Teleop (/cmd_vel) ]
                                    |
                                    v
+-----------------------------------------------------------------------+
|                       CONTROLLER MANAGER (50 Hz)                      |
|  +-----------------------------+   +-------------------------------+  |
|  |   joint_state_broadcaster   |   |  diff_drive_controller        |  |
|  | (Xuất topic /joint_states)  |   |  hoặc simple_velocity_ctrl    |  |
|  +-----------------------------+   +-------------------------------+  |
+-----------------------------------|-----------------------------------+
                                    |  state_interface (position, velocity)
                                    |  command_interface (velocity)
                                    v
+-----------------------------------------------------------------------+
|                     HARDWARE INTERFACE (URDF/Xacro)                   |
|        +--------------------------+--------------------------+        |
|        | use_gazebo := true       | use_gazebo := false      |        |
|        v                          v                          |        |
|  [GazeboSimSystem]       [MobileRobotInterface (C++)]        |        |
|  (Mô phỏng Gazebo)       (Giao tiếp UART/Serial với ESP32)   |        |
+-----------------------------------------------------------------------+
```

### Ba thành phần trụ cột của `ros2_control`:
1. **Hardware Interface (`gazebo_ros2_control.xacro`):** Khai báo danh sách các khớp động cơ (`right_wheel_joint`, `left_wheel_joint`) cùng 2 kênh giao tiếp:
   * **`command_interface`:** Kênh nhận lệnh điều khiển từ phần mềm gửi xuống động cơ (ví dụ: `velocity` – vận tốc góc $\text{rad/s}$).
   * **`state_interface`:** Kênh đọc dữ liệu phản hồi từ Encoder gửi lên phần mềm (gồm `position` – góc quay $\text{rad}$ và `velocity` – vận tốc góc $\text{rad/s}$).
2. **Plugin kết nối môi trường (`gazebo_sim_control.xacro`):** Nhúng plugin `gz_ros2_control::GazeboSimROS2ControlPlugin` vào Gazebo để tự động tạo node `/controller_manager` và đọc file cấu hình YAML.
3. **Cấu hình Bộ điều khiển (`controller.yaml`):** Khai báo tần số vòng lặp điều khiển (ví dụ `50 Hz`) và các bộ điều khiển cần nạp (`joint_state_broadcaster`, `simple_velocity_controller` hoặc `diff_drive_controller`).

---

## 2. Bước 1: Khai báo Giao diện Phần cứng trong `urdf/gazebo_ros2_control.xacro`

Trong dự án `mobileRobot_ws`, file `urdf/gazebo_ros2_control.xacro` sử dụng điều kiện `<xacro:if>` và `<xacro:unless>` theo tham số `use_gazebo` để tự động chuyển đổi giữa **Mô phỏng Gazebo** (`gz_ros2_control/GazeboSimSystem`) và **Robot thực tế dùng ESP32** (`mobile_robot_hardware/MobileRobotInterface`):

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

  <ros2_control name="MobileBaseHardwareInterface" type="system">

    <!-- Mô phỏng Gazebo: dùng GazeboSimSystem -->
    <xacro:if value="$(arg use_gazebo)">
      <hardware>
        <plugin>gz_ros2_control/GazeboSimSystem</plugin>
      </hardware>
    </xacro:if>

    <!-- Robot thật: dùng hardware interface đọc/ghi qua cổng serial (ESP32) -->
    <xacro:unless value="$(arg use_gazebo)">
      <hardware>
        <plugin>mobile_robot_hardware/MobileRobotInterface</plugin>
        <param name="serial_port">$(arg serial_port)</param>
        <param name="encoder_cpr">1234.3</param>
        <param name="use_pwm">false</param>
      </hardware>
    </xacro:unless>

    <!-- Khớp bánh phải: nhận lệnh vận tốc (-10 đến 10 rad/s), trả về vị trí & vận tốc -->
    <joint name="right_wheel_joint">
      <command_interface name="velocity">
        <param name="min">-10.0</param>
        <param name="max">10.0</param>
      </command_interface>
      <state_interface name="position" />
      <state_interface name="velocity" />
    </joint>

    <!-- Khớp bánh trái: nhận lệnh vận tốc (-10 đến 10 rad/s), trả về vị trí & vận tốc -->
    <joint name="left_wheel_joint">
      <command_interface name="velocity">
        <param name="min">-10.0</param>
        <param name="max">10.0</param>
      </command_interface>
      <state_interface name="position" />
      <state_interface name="velocity" />
    </joint>
  </ros2_control>

</robot>
```

---

## 3. Bước 2: Gắn Plugin `gz_ros2_control` trong `urdf/gazebo_sim_control.xacro`

Thay vì dùng plugin `DiffDrive` của Gazebo như ở Giai đoạn 2, file `urdf/gazebo_sim_control.xacro` cấu hình các thông số ma sát/độ cứng tiếp xúc cho các bánh xe (`mu1`, `mu2`, `kp`, `kd`) và nạp plugin cấp hệ thống **`gz_ros2_control::GazeboSimROS2ControlPlugin`** kèm đường dẫn tới file `controller.yaml`:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

  <!-- Cấu hình ma sát và độ cứng tiếp xúc cho bánh trái -->
  <gazebo reference="left_wheel">
    <mu1>1.0</mu1>
    <mu2>1.0</mu2>
    <kp>1000000.0</kp>
    <kd>10.0</kd>
    <minDepth>0.001</minDepth>
    <maxVel>0.1</maxVel>
    <fdir1>1 0 0</fdir1>
  </gazebo>

  <!-- Cấu hình ma sát và độ cứng tiếp xúc cho bánh phải -->
  <gazebo reference="right_wheel">
    <mu1>1.0</mu1>
    <mu2>1.0</mu2>
    <kp>1000000.0</kp>
    <kd>10.0</kd>
    <minDepth>0.001</minDepth>
    <maxVel>0.1</maxVel>
    <fdir1>1 0 0</fdir1>
  </gazebo>

  <!-- Giảm ma sát cho bánh tự lựa (caster_wheel) -->
  <gazebo reference="caster_wheel">
    <mu1>0.1</mu1>
    <mu2>0.1</mu2>
    <kp>1000000.0</kp>
    <kd>100.0</kd>
    <minDepth>0.001</minDepth>
    <maxVel>1.0</maxVel>
  </gazebo>

  <!-- Plugin tích hợp ros2_control vào môi trường Gazebo Sim -->
  <gazebo>
    <plugin filename="gz_ros2_control-system" name="gz_ros2_control::GazeboSimROS2ControlPlugin">
      <!-- Plugin cấp hệ thống của Gazebo, cho biết tích hợp ros2_control vào môi trường.
           Khi Gazebo nạp plugin này sẽ tự động tạo /controller_manager và đọc file yaml để biết cần quản lý những gì -->
      <parameters>$(find mobile_controller)/config/controller.yaml</parameters>
    </plugin>
  </gazebo>

</robot>
```

*(Trong file gốc `my_robot.urdf.xacro`, cả 2 file trên đều được nhúng vào thông qua `<xacro:include filename="gazebo_sim_control.xacro"/>` và `<xacro:include filename="gazebo_ros2_control.xacro"/>`).*

---

## 4. Bước 3: Cấu hình Bộ điều khiển trong `config/controller.yaml`

Trong gói `mobile_controller`, file `config/controller.yaml` định nghĩa cấu hình cho `controller_manager`. Tại đây bạn có **2 lựa chọn bộ điều khiển**:
* **Cách 1 (`simple_velocity_controller` – `JointGroupVelocityController`):** Nhận trực tiếp mảng vận tốc góc $[\omega_L, \omega_R]$ cho 2 bánh xe (rất phù hợp khi bạn muốn tự viết Node tính toán phương trình Động học thuận/nghịch và Odometry bằng Python/C++ để hiểu sâu bản chất toán học).
* **Cách 2 (`diff_drive_controller` – `DiffDriveController`):** Bộ điều khiển xe vi sai trọn gói của ROS 2 (tự động nhận `/cmd_vel`, tự tính động học sang 2 bánh xe và tự phát `/odom` + TF).

Mã nguồn đầy đủ file `mobile_controller/config/controller.yaml`:

```yaml
# controller_manager là "trái tim" của hệ thống ros2_control, chịu trách nhiệm quản lý vòng đời (khởi tạo, load, activate, deactivate) của tất cả các bộ điều khiển.
controller_manager:
  ros__parameters:
    update_rate: 50  # Tần số vòng lặp điều khiển (đơn vị: Hz). Dòng này bắt controller_manager chạy ở tần số 50 Hz (tức mỗi 20ms nó sẽ gọi hàm read() từ phần cứng và hàm update() của các controller một lần).

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster
# Chức năng: Đọc tất cả trạng thái khớp (position, velocity) từ state_interface của phần cứng và publish ra topic /joint_states.
# Tại sao phải dùng?: Các node như robot_state_publisher cần topic /joint_states này để tính toán các phép biến đổi TF giữa các bộ phận của robot. Không có nó, robot trong RViz2 sẽ bị mất liên kết hình học.

    # diff_drive_controller:
    #   type: diff_drive_controller/DiffDriveController
# Chức năng: Controller chuyên dụng cho xe di chuyển 2 bánh vi sai (Differential Drive).
# Cơ chế: Nó lắng nghe topic nhận lệnh /cmd_vel (geometry_msgs/msg/Twist), tính toán ra vận tốc góc cần thiết cho từng bánh xe và gửi lệnh vận tốc xuống command_interface của bánh trái và bánh phải. Đồng thời nó cũng đọc encoder để tính vị trí xe (Odometry).

    simple_velocity_controller:
      type: velocity_controllers/JointGroupVelocityController

simple_velocity_controller:
  ros__parameters:
    joints:
      - left_wheel_joint   # Tên khớp bánh xe trái (phải trùng với tên trong file URDF)
      - right_wheel_joint

# diff_drive_controller:
#   ros__parameters:
#     left_wheel_names: ["left_wheel_joint"]    # Bắt buộc phải trùng tên trong URDF
#     right_wheel_names: ["right_wheel_joint"]
#
#     wheel_separation: 0.45 # Khoảng cách giữa tâm 2 bánh xe 
#     wheel_radius: 0.1      # Bán kính bánh xe 
#     # Hai thông số này phải chuẩn xác tuyệt đối theo thiết kế mô hình CAD/URDF. Controller dùng 2 số này để chuyển đổi từ vận tốc dài/góc của robot (v, omega) sang vận tốc quay của từng bánh xe (omega_L, omega_R) và ngược lại. Khai báo sai sẽ dẫn đến xe chạy bị lệch hoặc tính toán Odometry bị sai lệch nghiêm trọng.
#     wheel_separation_multiplier: 1.0
#     left_wheel_radius_multiplier: 1.0
#     right_wheel_radius_multiplier: 1.0
#
#     publish_limited_velocity: true
#
#     odom_frame_id: "odom"
#     base_frame_id: "base_footprint"
#
#     pose_covariance_diagonal: [0.001, 0.001, 0.001, 0.001, 0.001, 0.01]
#     twist_covariance_diagonal: [0.001, 0.001, 0.001, 0.001, 0.001, 0.01]
#
#     open_loop: true
#     cmd_vel_timeout: 0.5
#     enable_odom_tf: true
#     publish_rate: 50.0
#     # odom_frame_id: "odom": Tên frame tọa độ vị trí bắt đầu của robot.
#     # base_frame_id: "base_footprint": Tên frame gốc gắn liền với trọng tâm robot dưới mặt đất.
#     # enable_odom_tf: true: Cho phép controller phát xuất (publish) phép biến đổi tọa độ odom -> base_footprint lên cây TF.
#     # publish_rate: 50.0: Tần số publish dữ liệu Odometry (topic /odom và cây TF) là 50 Hz.
#     # Ma trận hiệp phương sai (covariance): Định nghĩa độ sai số/nhiễu ước tính của dữ liệu Vị trí (pose) và Vận tốc (twist) thu được từ bánh xe. Dạng mảng 6 phần tử tương ứng với đường chéo ma trận 6x6 cho 6 bậc tự do: (x, y, z, roll, pitch, yaw). Thường được gói robot_localization (EKF node) sử dụng để dung hợp dữ liệu Odometry bánh xe với IMU/LiDAR.
#     linear.x.max_velocity: 1.0
#     linear.x.min_velocity: -1.0
#     linear.x.max_acceleration: 1.0
#     angular.z.max_velocity: 1.0
#     angular.z.min_velocity: -1.0
#     angular.z.max_acceleration: 1.0
#     angular.z.min_acceleration: -1.0
```

---

## 5. Bước 4: Kích hoạt (Spawn) các Bộ điều khiển bằng Launch File

Khi Gazebo khởi động plugin `gz_ros2_control`, node `/controller_manager` mới chỉ đọc file `controller.yaml` vào bộ nhớ chứ chưa tự động kích hoạt các controller. Chúng ta dùng công cụ `spawner` của gói `controller_manager` trong file `mobile_controller/launch/controller.launch.xml` để nạp và bật (`activate`) chúng:

```xml
<launch>
    <arg name="use_gazebo" default="true"/>
    <arg name="use_sim_time" default="true"/>
    <arg name="launch_rviz" default="true"/>
    <!-- <arg name="serial_port" default="/dev/ttyUSB0"/> -->

    <set_env name="GTK_PATH" value=""/>
    <set_env name="GTK_EXE_PREFIX" value=""/>
    <set_env name="GTK_IM_MODULE_FILE" value=""/>
    <set_env name="GIO_MODULE_DIR" value=""/>
    <set_env name="LOCPATH" value=""/>

    <!-- 1. Kích hoạt bộ phát trạng thái khớp (/joint_states) -->
    <node pkg="controller_manager" exec="spawner" args="joint_state_broadcaster"/>

    <!-- 2. Kích hoạt bộ điều khiển vận tốc bánh xe -->
    <!-- <node pkg="controller_manager" exec="spawner" args="diff_drive_controller"/> -->
    <node pkg="controller_manager" exec="spawner" args="simple_velocity_controller"/>
</launch>
```

### Các lệnh Terminal kiểm tra trạng thái `ros2_control`:
Sau khi khởi chạy mô phỏng, bạn có thể mở Terminal kiểm tra xem phần cứng và các bộ điều khiển đã ở trạng thái `active` và `claimed` hay chưa:

```bash
# 1. Xem danh sách các giao diện phần cứng (command_interface & state_interface)
ros2 control list_hardware_interfaces

# 2. Xem trạng thái các bộ điều khiển đang chạy (active / inactive)
ros2 control list_controllers

# 3. Phát thử lệnh vận tốc góc (rad/s) trực tiếp cho 2 bánh xe qua simple_velocity_controller
ros2 topic pub /simple_velocity_controller/commands std_msgs/msg/Float64MultiArray "data: [5.0, 5.0]"
```

---

[⬅ Quay lại bài: Động học, Odometry và Điều khiển Robot (ros2_control)](/posts/2026/11/dong-hoc-odometry-ros2-control/)
