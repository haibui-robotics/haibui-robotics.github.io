---
layout: single
title: "Tutorial 27: Thiết kế & Tích hợp Thế giới ảo (.world / .sdf) trong Gazebo Sim (Dự án amr_ws)"
permalink: /posts/2026/11/ros2-tutorial-27/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 25 - Gắn cảm biến IMU vào Gazebo](/posts/2026/11/ros2-tutorial-25/)

---

## 1. Lý thuyết cốt lõi: File `.world` / `.sdf` là gì?

Khi làm việc với ROS 2 và Gazebo, bạn sẽ thấy có 2 định dạng file XML khác nhau:
* **URDF / Xacro (`.urdf`, `.xacro`):** Chỉ dùng để mô tả **một con robot đơn lẻ** (các khớp, các khâu, cảm biến gắn trên thân xe).
* **SDF – Simulation Description Format (`.world`, `.sdf`):** Dùng để mô tả **toàn bộ thế giới ảo** bao gồm: gia tốc trọng trường, bước nhảy thời gian vật lý, ánh sáng mặt trời, mặt đất, nhà kho, kệ hàng, bàn ghế và cả các plugin hệ thống của môi trường.

### Giải phẫu 5 thành phần bắt buộc của một file World trong Gazebo Sim (Harmonic)

1. **Cấu hình Vật lý (`<physics>` & `<gravity>`):**
   Quy định bộ giải vật lý (`ode`, `dart`...), bước thời gian mô phỏng `max_step_size` (thường là `0.001s` tương ứng tần số tính toán va chạm `1000 Hz`) và tốc độ thời gian thực `real_time_factor = 1`.
2. **Các System Plugins cấp độ World (Cực kỳ quan trọng!):**
   > ⚠️ **Lưu ý sống còn khi chuyển các file `.world` cũ sang Gazebo Sim (Jazzy / Harmonic):**  
   > Nếu trong file `.world` định nghĩa sẵn các plugin `<plugin>` nhưng thiếu `gz-sim-sensors-system` hoặc `gz-sim-imu-system`, thì dù file URDF của robot có khai báo LiDAR, Camera hay IMU đúng 100%, **Gazebo cũng sẽ không phát ra bất kỳ dữ liệu cảm biến nào!**
3. **Ánh sáng (`<light>`):** Thường là nguồn sáng định hướng mặt trời (`type="directional" name="sun"`).
4. **Mặt sàn (`<model name="ground_plane">`):** Một mặt phẳng tĩnh (`<static>true</static>`) có thẻ `<collision>` để đỡ robot không bị rơi tự do xuống vực.
5. **Các mô hình vật thể (`<model>` / `<include>`):** Tường, kệ hàng, bàn ghế... có thể viết trực tiếp (*inline*) hoặc nhúng qua đường dẫn `model://` và `https://fuel.gazebosim.org/...`.

---

## 2. Phân tích 4 World thực tế trong dự án `amr_ws`

Trong thư mục `amr_ws/src/amr_description/worlds/`, dự án được trang bị 4 thế giới ảo từ đơn giản đến phức tạp phục vụ từng mục đích kiểm thử khác nhau:

```text
amr_ws/src/amr_description/
├── models/                         # Chứa tài nguyên 3D của các mô hình nhà kho & căn hộ
│   ├── aws_robomaker_warehouse_ShelfE_01/
│   ├── aws_robomaker_warehouse_WallB_01/
│   ├── aws_robomaker_residential_Bed_01/
│   └── ...
└── worlds/
    ├── empty.world                 # [1] Thế giới trống tối giản (test nhanh động học)
    ├── test_world.sdf              # [2] Thế giới tự dựng bằng khối cơ bản + Gazebo Fuel
    ├── small_warehouse.world       # [3] Nhà kho công nghiệp AWS (test SLAM & AMR)
    └── small_house.world           # [4] Căn hộ gia đình nhiều phòng (test điều hướng hẹp)
```

---

### Cấp độ 1: Thế giới trống tối giản (`worlds/empty.world`)
Khi bạn mới thử nghiệm thuật toán điều khiển động cơ hoặc kiểm tra chiều quay bánh xe, việc mở một nhà kho nặng hàng trăm MB là không cần thiết. File `empty.world` chỉ gồm đúng **Ánh sáng mặt trời** và **Mặt sàn $100 \times 100\text{m}$**, giúp Gazebo khởi động tức thì:

```xml
<?xml version="1.0" ?>
<sdf version="1.7">
    <world name="empty_world">
        <physics type="ode">
            <max_step_size>0.01</max_step_size>
        </physics>

        <!-- Nguồn sáng mặt trời -->
        <light type="directional" name="sun">
            <cast_shadows>true</cast_shadows>
            <pose>0 0 10 0 0 0</pose>
            <diffuse>0.8 0.8 0.8 1</diffuse>
            <specular>0.2 0.2 0.2 1</specular>
            <direction>-0.5 0.1 -0.9</direction>
        </light>
        
        <!-- Mặt phẳng sàn 100m x 100m -->
        <model name="ground_plane">
            <static>true</static>
            <link name="link">
                <collision name="collision">
                    <geometry>
                        <plane><normal>0 0 1</normal></plane>
                    </geometry>
                </collision>
                <visual name="visual">
                    <geometry>
                        <plane>
                            <normal>0 0 1</normal>
                            <size>100 100</size>
                        </plane>
                    </geometry>
                </visual>
            </link>
        </model>
    </world>
</sdf>
```

---

### Cấp độ 2: Tự dựng World & Nhúng model từ đám mây (`worlds/test_world.sdf`)
Trong `test_world.sdf`, chúng ta kết hợp 2 kỹ thuật:
1. **Tự tạo vật cản tĩnh bằng khối hình học (`<box>`):** Đặt thẻ `<static>true</static>` để cố định bàn, ghế, tường tại tọa độ `<pose>x y z roll pitch yaw</pose>`.
2. **Nhúng trực tiếp mô hình từ đám mây Gazebo Fuel (`<include>`):** Không cần tải sẵn về máy, chỉ cần gắn link URI từ `https://fuel.gazebosim.org/...`.

```xml
<sdf version='1.10'>
  <world name='empty'>
    <!-- 5 System Plugins cốt lõi của Gazebo Sim -->
    <plugin name='gz::sim::systems::Physics' filename='gz-sim-physics-system'/>
    <plugin name='gz::sim::systems::UserCommands' filename='gz-sim-user-commands-system'/>
    <plugin name='gz::sim::systems::SceneBroadcaster' filename='gz-sim-scene-broadcaster-system'/>
    <plugin name='gz::sim::systems::Contact' filename='gz-sim-contact-system'/>
    <plugin name='gz::sim::systems::Sensors' filename='gz-sim-sensors-system'>
      <render_engine>ogre2</render_engine>
    </plugin>

    <!-- Cách 1: Tự định nghĩa vật cản cố định bằng khối hình hộp (Inline Model) -->
    <model name='DiningTable'>
      <static>true</static>
      <pose>1.789 -2.261 0 0 0 1.556</pose>
      <link name='link'>
        <pose>0 0 0.375 0 0 0</pose>
        <visual name='visual'>
          <geometry><box><size>1.6 0.9 0.75</size></box></geometry>
        </visual>
        <collision name='collision'>
          <geometry><box><size>1.6 0.9 0.75</size></box></geometry>
        </collision>
      </link>
    </model>

    <!-- Cách 2: Nhúng trực tiếp mô hình từ đám mây Gazebo Fuel -->
    <include>
      <uri>https://fuel.gazebosim.org/1.0/OpenRobotics/models/Grey Wall</uri>
      <name>grey_wall</name>
      <pose>8.363 -1.409 0 0 0 1.543</pose>
    </include>
  </world>
</sdf>
```

> 💡 **Kinh nghiệm xương máu khi lưu World từ Gazebo GUI:**  
> Khi bạn kéo thả model trong Gazebo rồi bấm *Save World*, Gazebo đôi khi lưu đường dẫn tuyệt đối theo tên máy tính của người tạo (ví dụ: `file:///home/ed/.gz/fuel/...`). Khi mang file `.sdf` đó sang máy bạn (`/home/pc/...`), Gazebo sẽ báo lỗi **`Failed to load a world`**. Giải pháp là luôn đổi đường dẫn về chuẩn tương đối `model://Ten_Model` hoặc link online `https://fuel.gazebosim.org/...` như trong `test_world.sdf`!

---

### Cấp độ 3: Bản đồ Nhà kho Công nghiệp (`worlds/small_warehouse.world`)
Đây là bản đồ chính đang được sử dụng trong `amr_ws`. Để bản đồ nhà kho (vốn được thiết kế từ thời Gazebo Classic) hoạt động hoàn hảo với mọi cảm biến (`LiDAR`, `Camera`, `IMU`) trên **Gazebo Sim (ROS 2 Jazzy)**, ở đầu file `small_warehouse.world` ta bổ sung đầy đủ **5 System Plugins** và gọi các mô hình kệ hàng qua giao thức `model://`:

```xml
<?xml version='1.0' encoding='utf-8'?>
<sdf version="1.6">
  <world name="default">
    <gravity>0 0 -9.8</gravity>
    <physics default="0" name="default_physics" type="ode">
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1</real_time_factor>
      <real_time_update_rate>1000</real_time_update_rate>
    </physics>

    <!-- 5 Plugin hệ thống của Gazebo Sim cho vật lý, giao diện và cảm biến LiDAR, Camera, IMU -->
    <plugin filename="gz-sim-physics-system" name="gz::sim::systems::Physics"/>
    <plugin filename="gz-sim-user-commands-system" name="gz::sim::systems::UserCommands"/>
    <plugin filename="gz-sim-scene-broadcaster-system" name="gz::sim::systems::SceneBroadcaster"/>
    <plugin filename="gz-sim-sensors-system" name="gz::sim::systems::Sensors">
      <render_engine>ogre2</render_engine>
    </plugin>
    <plugin filename="gz-sim-imu-system" name="gz::sim::systems::Imu"/>

    <!-- Nhúng các mô hình kệ hàng, tường nhà kho từ thư mục amr_description/models/ -->
    <model name="aws_robomaker_warehouse_ShelfF_01_001">
      <include>
        <uri>model://aws_robomaker_warehouse_ShelfF_01</uri>
      </include>
      <pose frame="">-5.795143 -0.956635 0 0 0 0</pose>
    </model>

    <model name="aws_robomaker_warehouse_WallB_01_001">
      <include>
        <uri>model://aws_robomaker_warehouse_WallB_01</uri>
      </include>
      <pose frame="">0.0 0.0 0 0 0 0</pose>
    </model>
    <!-- ... (các kệ hàng ShelfD, ShelfE, thùng hàng, pallet...) ... -->
  </world>
</sdf>
```

Khi kết hợp với dòng cấu hình trong `launch/gazebo.launch.xml`:
```xml
<set_env name="GZ_SIM_RESOURCE_PATH" value="/home/pc/amr_ws/src/amr_description/models:/home/pc/amr_ws/src"/>
```
Gazebo sẽ tự động tìm vào thư mục `amr_description/models/` để lấy toàn bộ hình ảnh 3D và kết cấu bề mặt (*textures*) của nhà kho!

---

## 3. Tổng hợp các Nguồn tải World & 3D Models đẹp nhất (Miễn phí)

Thay vì phải tự vẽ từng cái bàn, kệ hàng từ đầu, bạn có thể tải các bản đồ tiêu chuẩn quốc tế tại các kho mã nguồn mở sau:

### 1. Kho đám mây chính thức Gazebo Fuel (Open Robotics)
* 🔗 **Link:** [https://app.gazebosim.org/fuel](https://app.gazebosim.org/fuel)
* **Đặc điểm:** Hàng ngàn mô hình 3D (xe nâng, người đi bộ, kệ kho, tòa nhà, địa hình đồi núi) và Worlds hoàn chỉnh. Bạn có thể bấm nút **Copy SDF** trên web để dán thẳng thẻ `<include><uri>https://fuel.gazebosim.org/...</uri></include>` vào file `.world` của mình.

### 2. Bộ tứ Thế giới thực tế của AWS RoboMaker (Chuẩn công nghiệp cho AMR)
Do đội ngũ Amazon Web Services Robotics thiết kế với độ chi tiết cực cao, rất lý tưởng để test SLAM và Navigation:
* 🏭 **Nhà kho công nghiệp (Small Warehouse World):**  
  [https://github.com/aws-robotics/aws-robomaker-small-warehouse-world](https://github.com/aws-robotics/aws-robomaker-small-warehouse-world) *(Đang tích hợp trong `amr_ws`)*
* 🏠 **Căn hộ gia đình (Small House World):**  
  [https://github.com/aws-robotics/aws-robomaker-small-house-world](https://github.com/aws-robotics/aws-robomaker-small-house-world) *(Đang tích hợp trong `amr_ws`)*
* 🏥 **Bệnh viện nhiều phòng (Hospital World):**  
  [https://github.com/aws-robotics/aws-robomaker-hospital-world](https://github.com/aws-robotics/aws-robomaker-hospital-world)
* 📚 **Cửa hàng sách / Siêu thị mini (Bookstore World):**  
  [https://github.com/aws-robotics/aws-robomaker-bookstore-world](https://github.com/aws-robotics/aws-robomaker-bookstore-world)

### 3. Bộ Worlds của Clearpath Robotics & ROBOTIS TurtleBot3
* 🚧 **Clearpath Simulation Worlds** (Văn phòng, công trường xây dựng, nhà kho, địa hình nông nghiệp):  
  [https://github.com/clearpathrobotics/clearpath_simulator](https://github.com/clearpathrobotics/clearpath_simulator)
* 🤖 **TurtleBot3 Worlds** (Bản đồ lục giác kinh điển để test SLAM nhanh và nhẹ):  
  [https://github.com/ROBOTIS-GIT/turtlebot3_simulations](https://github.com/ROBOTIS-GIT/turtlebot3_simulations)

---

## 4. Cách đổi nhanh giữa các World trong `amr_ws`

Trong file `amr_description/launch/gazebo.launch.xml`, bạn chỉ cần thay tên file `.world` ở thẻ `<arg name="gz_args" .../>` (hoặc tạo một biến `world_name` truyền từ Terminal):

```xml
    <!-- Đổi small_warehouse.world thành small_house.world, test_world.sdf hoặc empty.world -->
    <include file="$(find-pkg-share ros_gz_sim)/launch/gz_sim.launch.py">
        <arg name="gz_args" value="$(find-pkg-share amr_description)/worlds/small_warehouse.world -r"/>
    </include>
```

*(Lưu ý: Nếu chuyển sang `small_house.world` hoặc `empty.world` mà muốn sử dụng đầy đủ cảm biến LiDAR/IMU trên Gazebo Sim, đừng quên copy khối 5 thẻ `<plugin>` từ đầu file `small_warehouse.world` sang đầu file world đó nhé!)*

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/) | [⬅ Bài trước: Tutorial 25 - Gắn cảm biến IMU vào Gazebo](/posts/2026/11/ros2-tutorial-25/)
