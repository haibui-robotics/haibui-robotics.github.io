---
layout: single
title: "Tutorial 21: Làm cho robot di chuyển trong Gazebo (Plugins & ros_gz_bridge)"
permalink: /posts/2026/11/ros2-tutorial-21/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 20 - Cài đặt Gazebo Sim & Spawn robot](/posts/2026/11/ros2-tutorial-20/)

---

## 1. Mục tiêu và Khái niệm cốt lõi

Tính đến [Tutorial 20](/posts/2026/11/ros2-tutorial-20/), mô hình robot của chúng ta trong Gazebo mới chỉ là một "bức tượng" có trọng lượng. Để robot có thể chạy được khi ta bấm phím điều khiển, chúng ta cần giải quyết 2 vấn đề:

* **Vấn đề 1 (Động cơ ảo):** Cần gắn thêm các module giả lập động cơ và cảm biến góc quay vào bánh xe. Trong Gazebo, chúng được gọi là các **Plugin**.
* **Vấn đề 2 (Khác biệt ngôn ngữ):** Gazebo Sim và ROS 2 là 2 phần mềm độc lập. ROS 2 dùng tiêu chuẩn tin nhắn riêng (ví dụ: `geometry_msgs/msg/Twist`), trong khi Gazebo dùng chuẩn Google Protobuf (ví dụ: `gz.msg.Twist`). Để điều khiển robot từ ROS 2, ta cần một "thông dịch viên" gọi là **`ros_gz_bridge`**.

---

## 2. Phần 1: Khai báo Gazebo Plugins bằng Xacro

Tạo một file mới tên là `mobile_base.gazebo.xacro` (để tách biệt code vật lý với code hình học). File này sẽ cấu hình ma sát cho bánh xe tự lựa và gắn 2 Plugin quan trọng: `DiffDrive` (Điều khiển xe 2 bánh) và `JointStatePublisher` (Đọc góc xoay của bánh xe).

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
    
    <!-- 1. Cấu hình ma sát cho bánh xe tự lựa (Caster Wheel) -->
    <!-- Giảm ma sát (mu) gần bằng 0 để bánh xe trượt mượt mà, không bị kẹt -->
    <gazebo reference="caster_wheel">     
        <mu1 value="0.1" />    
        <mu2 value="0.1" />
    </gazebo>

    <!-- 2. Plugin Điều khiển xe 2 bánh (Differential Drive) -->
    <gazebo>
        <plugin filename="gz-sim-diff-drive-system" name="gz::sim::system::DiffDrive">
            <!-- Tên khớp nối bánh xe trái và phải (phải khớp với tên định nghĩa trong thẻ joint) -->
            <left_joint>left_joint</left_joint>
            <right_joint>right_joint</right_joint>
            <!-- Khoảng cách giữa 2 bánh xe và bán kính bánh xe -->
            <wheel_separation>0.45</wheel_separation>
            <wheel_radius>0.1</wheel_radius>
            <!-- Khai báo hệ tọa độ TF -->
            <frame_id>odom</frame_id>
            <child_frame_id>base_link</child_frame_id>
        </plugin>
    </gazebo>

    <!-- 3. Plugin phát dữ liệu góc quay của các khớp -->
    <gazebo>
        <plugin filename="gz-sim-joint-state-publisher-system" name="gz::sim::system::JointStatePublisher">
            <joint_name>left_joint</joint_name>
            <joint_name>right_joint</joint_name>
        </plugin>
    </gazebo>

</robot>
```

*(Đừng quên thêm thẻ `<xacro:include filename="mobile_base.gazebo.xacro"/>` vào file `my_robot.urdf.xacro` gốc của bạn nhé).*

---

## 3. Phần 2: Cấu hình Cầu nối (`ros_gz_bridge`)

File cấu hình YAML này đóng vai trò là "bảng từ điển dịch thuật", chỉ định chính xác: Topic nào ở Gazebo sẽ kết nối với Topic nào ở ROS 2, dùng kiểu dữ liệu gì và dữ liệu sẽ chảy theo chiều nào.

Tạo thư mục `config` (cùng cấp với thư mục `launch`) và tạo file `gazebo_bridge.yaml`:

```yaml
# Đồng bộ thời gian mô phỏng giữa Gazebo và ROS 2
- ros_topic_name: "/clock"
  gz_topic_name: "/clock"
  ros_type_name: "rosgraph_msgs/msg/Clock"
  gz_type_name: "gz.msg.Clock"
  direction: GZ_TO_ROS 

# Truyền góc xoay bánh xe từ Gazebo về ROS 2
- ros_topic_name: "/joint_states"
  gz_topic_name: "/world/empty/model/my_robot/joint_state"
  ros_type_name: "sensor_msgs/msg/JointState"
  gz_type_name: "gz.msg.Model"
  direction: GZ_TO_ROS 

# Truyền hệ trục tọa độ chuyển động (Odometry) từ Gazebo về ROS 2
- ros_topic_name: "/tf"
  gz_topic_name: "/model/my_robot/tf"
  ros_type_name: "tf2_msgs/msg/TFMessage"
  gz_type_name: "gz.msg.Pose_V"
  direction: GZ_TO_ROS

# Nhận lệnh điều khiển từ bàn phím/ROS 2 gửi vào Gazebo
- ros_topic_name: "/cmd_vel"
  gz_topic_name: "/model/my_robot/cmd_vel"
  ros_type_name: "geometry_msgs/msg/Twist"
  gz_type_name: "gz.msg.Twist"
  direction: ROS_TO_GZ 
```

**Mỗi block bắt buộc phải có 5 thông số:**

1. `ros_topic_name`: Tên topic hiển thị phía ROS 2.
2. `gz_topic_name`: Tên topic thực tế phát ra bên trong Gazebo.
3. `ros_type_name`: Kiểu tin nhắn chuẩn của ROS 2.
4. `gz_type_name`: Kiểu tin nhắn của Gazebo (Protobuf).
5. `direction`: Hướng truyền dữ liệu (`GZ_TO_ROS`, `ROS_TO_GZ`, hoặc `BIDIRECTIONAL`).

---

## 4. Mẹo (Pro-Tip): Làm sao để tự tìm tên và kiểu tin nhắn?

Khi làm việc với các cảm biến mới (LiDAR, Camera), bạn sẽ phải tự cấu hình Bridge. Dưới đây là tuyệt chiêu tra cứu bằng Terminal:

### A. Tra cứu phía Gazebo:
* **Tìm tên Topic (`gz_topic_name`):** Mở Gazebo, mở Terminal và gõ:
  ```bash
  gz topic -l
  ```
  *(Sẽ ra danh sách thực tế ví dụ: `/model/my_robot/cmd_vel`)*.
* **Tìm kiểu dữ liệu (`gz_type_name`):** Gõ lệnh info kèm tên topic vừa lấy được:
  ```bash
  gz topic -i -t /model/my_robot/cmd_vel
  ```
  *(Sẽ ra `Type: gz.msgs.Twist`)*.

### B. Tra cứu phía ROS 2:
* **Tìm tên Topic (`ros_topic_name`):** Gõ lệnh:
  ```bash
  ros2 topic list
  ```
* **Tìm kiểu dữ liệu (`ros_type_name`):** Dùng lệnh info:
  ```bash
  ros2 topic info /cmd_vel
  ```
  *(Sẽ ra `Type: geometry_msgs/msg/Twist`)*.

---

## 5. Cập nhật Launch File và Khởi chạy

**Bước 1:** Mở `package.xml` và thêm dependency để package hiểu được cầu nối:
```xml
<exec_depend>ros_gz_bridge</exec_depend>
```

**Bước 2:** Mở file launch (ví dụ `gazebo.launch.xml`), thêm biến dẫn tới file config và khởi chạy node `parameter_bridge`:

```xml
<launch>
    <!-- ... (các node khởi chạy Gazebo và spawn robot của bạn) ... -->

    <!-- Gọi biến trỏ tới file YAML -->
    <let name="gazebo_config_path" value="$(find-pkg-share my_robot_description)/config/gazebo_bridge.yaml"/>

    <!-- Khởi chạy Node Bridge -->
    <node pkg="ros_gz_bridge" exec="parameter_bridge">
        <param name="config_file" value="$(var gazebo_config_path)"/>
    </node>
</launch>
```

**Hoàn tất!** Giờ đây, khi bạn chạy Launch file, bạn có thể mở một Terminal khác, gõ lệnh:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

và nhấn các phím mũi tên để xem con robot của bạn chính thức lăn bánh trong môi trường vật lý ảo Gazebo!

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 20 - Cài đặt Gazebo Sim & Spawn robot](/posts/2026/11/ros2-tutorial-20/)
