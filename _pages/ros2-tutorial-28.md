---
layout: single
title: "Tutorial 28: Tiêu chuẩn điều khiển /cmd_vel (Twist, TwistStamped) và Tự viết Node Teleop Bàn phím (Dự án mobileRobot_ws)"
permalink: /posts/2026/11/ros2-tutorial-28/
author_profile: true
---

[⬅ Quay lại bài: Động học, Odometry và Điều khiển Robot (ros2_control)](/posts/2026/11/dong-hoc-odometry-ros2-control/) | [Bài tiếp: Tutorial 30 - Giới thiệu ros2_control & Bộ điều khiển ➡](/posts/2026/11/ros2-tutorial-30/)

---

## 1. Lý thuyết cốt lõi: Topic `/cmd_vel` và 3 chuẩn gói tin điều khiển vận tốc

Trong hệ sinh thái ROS 2, để ra lệnh cho một robot di chuyển, các thuật toán điều hướng (như Nav2) hoặc bộ điều khiển tay cầm/bàn phím (*Teleop*) không gửi trực tiếp xung PWM tới động cơ mà gửi **yêu cầu vận tốc** qua một Topic tiêu chuẩn (thường đặt tên là `/cmd_vel`).

Tùy thuộc vào bộ điều khiển phía dưới mà bạn đang sử dụng, có **3 kiểu gói tin (Message Types)** điều khiển vận tốc mà bạn bắt buộc phải phân biệt rõ:

| Kiểu gói tin (Message Type) | Cấu trúc dữ liệu chính | Khi nào sử dụng? |
| :--- | :--- | :--- |
| **`geometry_msgs/msg/Twist`** | `linear: {x, y, z}` ($\text{m/s}$)<br>`angular: {x, y, z}` ($\text{rad/s}$) | Dùng cho plugin **`DiffDrive` thuần của Gazebo Sim** (như ở [Tutorial 21](/posts/2026/11/ros2-tutorial-21/)). |
| **`geometry_msgs/msg/TwistStamped`** | `header: {stamp, frame_id}`<br>`twist: {linear, angular}` | Dùng cho **`diff_drive_controller`** của `ros2_control` hoặc node tính động học nghịch `simple_controller` (chuẩn mới trên ROS 2 Jazzy, có thêm dấu thời gian `stamp` để tự động phanh xe nếu gói tin bị trễ quá `cmd_vel_timeout`). |
| **`std_msgs/msg/Float64MultiArray`** | `data: [left_vel, right_vel]` ($\text{rad/s}$) | Dùng khi điều khiển trực tiếp vận tốc góc từng bánh xe thông qua **`JointGroupVelocityController`** (`simple_velocity_controller` trong `ros2_control`). |

> 💡 **Lưu ý với xe tự hành 2 bánh vi sai (Differential Drive):**  
> Robot di chuyển trên mặt phẳng 2D và không thể trượt ngang, nên trong gói tin `Twist` / `TwistStamped`, ta chỉ sử dụng duy nhất 2 thành phần:
> * **`linear.x`** ($\text{m/s}$): Vận tốc tiến ($+$) hoặc lùi ($-$).
> * **`angular.z`** ($\text{rad/s}$): Vận tốc xoay quanh trục thẳng đứng $Z$ — theo quy tắc bàn tay phải, **quay trái là dương ($+$)**, **quay phải là âm ($-$)**.

---

## 2. Cách 1: Sử dụng Node `teleop_twist_keyboard` có sẵn của ROS 2

ROS 2 cung cấp sẵn gói công cụ `teleop_twist_keyboard` giúp bạn kiểm tra nhanh chuyển động của robot mà chưa cần viết code.

### 1. Cài đặt (nếu chưa có):
```bash
sudo apt install ros-jazzy-teleop-twist-keyboard
```

### 2. Chạy ở chế độ mặc định (Phát gói tin `Twist` lên `/cmd_vel`):
Phù hợp khi bạn đang chạy robot với plugin `DiffDrive` của Gazebo Sim:
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```
*(Sử dụng cụm phím `i`, `,`, `j`, `l` để di chuyển, `k` để dừng, `q`/`z` để tăng/giảm tốc độ).*

### 3. Mẹo chạy ở chế độ `TwistStamped` và đổi tên Topic (Remapping):
Nếu bộ điều khiển của bạn yêu cầu gói tin `TwistStamped` hoặc lắng nghe ở một topic khác (ví dụ `/mobile_controller/cmd_vel`), bạn có thể truyền tham số trực tiếp trên Terminal:
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -p stamped:=true -r cmd_vel:=/mobile_controller/cmd_vel
```

---

## 3. Cách 2 (Mở rộng từ `mobileRobot_ws`): Tự viết Node `teleop_keyboard.py` điều khiển trực tiếp vận tốc 2 bánh (`Float64MultiArray`)

Trong thực tế, công cụ `teleop_twist_keyboard` mặc định dùng cụm phím `i, j, l, ,` khá bất tiện so với cụm phím **`W - A - S - D`** quen thuộc, đồng thời không thể phát mảng `Float64MultiArray` cho bộ điều khiển `JointGroupVelocityController` (`simple_velocity_controller`).

Trong dự án **`mobileRobot_ws`**, file `mobile_controller/teleop_keyboard.py` được thiết kế để:
* Đọc phím **`W, A, S, D, SPACE`** trực tiếp từ Terminal ở chế độ không chặn (*non-blocking*) bằng thư viện `termios`, `tty` và `select` (không cần nhấn `Enter` sau mỗi lần bấm).
* Tự tính toán vận tốc góc (đơn vị $\text{rad/s}$) cho bánh trái (`self.left`) và bánh phải (`self.right`), đóng gói vào `std_msgs/msg/Float64MultiArray` (`data = [left, right]`) và gửi ngay lập tức khi bấm phím kết hợp với Timer định kỳ `10 Hz`.
* Tự động gửi liên tiếp 5 gói tin vận tốc `[0.0, 0.0]` (`publish_stop()`) trước khi thoát chương trình (`q` hoặc `Ctrl+C`) để đảm bảo robot dừng hẳn an toàn.

### Mã nguồn `mobile_controller/teleop_keyboard.py`:

```python
#!/usr/bin/env python3
# =====================================================================================
#  teleop_keyboard.py - Dieu khien TRUC TIEP van toc 2 banh cho
#  JointGroupVelocityController (velocity_controllers/JointGroupVelocityController).
#
#  KHAC VOI diff_drive_controller:
#    - diff_drive_controller nhan TwistStamped (linear.x, angular.z) roi TU tinh
#      dong hoc nghich ra van toc 2 banh, dong thoi phat /odom + TF.
#    - JointGroupVelocityController KHONG lam gi ca: no chi nhan 1 mang
#      std_msgs/Float64MultiArray roi gan THANG vao command_interface/velocity
#      cua tung khop theo DUNG THU TU khai bao trong tham so `joints`.
#
#  => Chinh NODE NAY phai tu tinh van toc banh trai/phai.
#     data[0] = van toc left_wheel_joint  (rad/s)
#     data[1] = van toc right_wheel_joint (rad/s)
#
#  CHAY:
#      ros2 launch my_mobilerobot gazebo.launch.xml
#      ros2 run my_mobilerobot teleop_keyboard
#
#  PHIM:
#      w : 2 banh cung +  (di thang toi)
#      s : 2 banh cung -  (lui)
#      a : trai - / phai + (quay tai cho sang trai)
#      d : trai + / phai - (quay tai cho sang phai)
#      SPACE : dung (ca 2 ve 0)
#      q / Ctrl+C : thoat (tu dung truoc khi thoat)
# =====================================================================================

import select
import sys
import termios
import tty

import rclpy
from rclpy.node import Node
from rclpy.parameter import Parameter
from std_msgs.msg import Float64MultiArray

# Topic ma JointGroupVelocityController lang nghe: <ten_controller>/commands
# CMD_TOPIC = '/simple_velocity_controller/commands'
CMD_TOPIC = '/cmd_vel'
HELP_TEXT = """
--------------------- TELEOP JOINT GROUP VELOCITY ---------------------
   w : di thang toi         s : lui
   a : quay tai cho trai    d : quay tai cho phai
   SPACE : DUNG NGAY        q : thoat
------------------------------------------------------------------------
"""


class TeleopKeyboard(Node):
    """Node publish Float64MultiArray vao topic lenh cua JointGroupVelocityController."""

    def __init__(self):
        # [NOTE] Khong hardcode use_sim_time=True de khi chay robot that (khong co Gazebo /clock)
        # thi timer va cac lenh bam phim van duoc publish binh thuong.
        super().__init__('teleop_keyboard')

        # ---- Tham so chinh duoc: --ros-args -p vel_step:=0.5 -p max_vel:=6.0 ----
        self.declare_parameter('vel_step', 0.5)     # rad/s moi lan bam
        self.declare_parameter('max_vel', 6.0)      # rad/s (cho phep chay nhanh den ~60 RPM)
        self.declare_parameter('publish_rate', 10.0)

        # Van toc 2 banh hien tai (rad/s). Thu tu PHAI khop `joints` trong controller.yaml:
        #   joints: [left_wheel_joint, right_wheel_joint]  =>  data = [left, right]
        self.left = 0.0
        self.right = 0.0

        self.publisher_ = self.create_publisher(Float64MultiArray, CMD_TOPIC, 10)
        self.create_timer(1.0 / self.get_parameter('publish_rate').value, self.publish_cmd)

    def publish_cmd(self):
        """Dong goi [left, right] thanh Float64MultiArray roi gui."""
        msg = Float64MultiArray()
        msg.data = [float(self.left), float(self.right)]
        self.publisher_.publish(msg)

    def _clamp(self, v):
        m = float(self.get_parameter('max_vel').value)
        return max(-m, min(m, v))

    def update_from_key(self, key):
        step = float(self.get_parameter('vel_step').value)
        if key == 'w':                       # di thang: 2 banh cung dau (+)
            self.left = self._clamp(self.left + step)
            self.right = self._clamp(self.right + step)
        elif key == 's':                     # lui: 2 banh cung dau (-)
            self.left = self._clamp(self.left - step)
            self.right = self._clamp(self.right - step)
        elif key == 'a':                     # quay trai: trai -, phai +
            self.left = self._clamp(self.left - step)
            self.right = self._clamp(self.right + step)
        elif key == 'd':                     # quay phai: trai +, phai -
            self.left = self._clamp(self.left + step)
            self.right = self._clamp(self.right - step)
        elif key == ' ':
            self.left = 0.0
            self.right = 0.0
        else:
            return False

        self.get_logger().info(
            f'left={self.left:+.2f} rad/s | right={self.right:+.2f} rad/s',
            throttle_duration_sec=0.2)
        # [NOTE] Gui lenh ngay khi bam phim (khong can cho timer)
        self.publish_cmd()
        return True

    def publish_stop(self):
        self.left = 0.0
        self.right = 0.0
        for _ in range(5):
            self.publish_cmd()
            rclpy.spin_once(self, timeout_sec=0.05)


def main(args=None):
    rclpy.init(args=args)
    node = TeleopKeyboard()

    if not sys.stdin.isatty():
        node.get_logger().error(
            'stdin khong phai terminal => khong doc duoc ban phim. '
            'Hay chay truc tiep trong terminal, hoac dung node demo.')
        node.destroy_node()
        rclpy.shutdown()
        return

    settings = termios.tcgetattr(sys.stdin)
    print(HELP_TEXT)

    try:
        tty.setcbreak(sys.stdin.fileno())
        running = True
        while running and rclpy.ok():
            rclpy.spin_once(node, timeout_sec=0.0)
            if select.select([sys.stdin], [], [], 0.05)[0]:
                key = sys.stdin.read(1)
                if key in ('q', '\x03'):
                    running = False
                else:
                    node.update_from_key(key)
    except KeyboardInterrupt:
        pass
    finally:
        node.get_logger().info('Dang dung robot va thoat...')
        node.publish_stop()
        termios.tcsetattr(sys.stdin, termios.TCSADRAIN, settings)
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

## 4. Cách 3 (Mở rộng từ `mobileRobot_ws`): Tự viết Node `teleop_twiststamped.py` phát gói tin `TwistStamped`

Khi bạn sử dụng bộ điều khiển động học (như `simple_controller.py` ở bài tiếp theo hoặc `DiffDriveController` của `ros2_control`), bộ điều khiển sẽ nhận vận tốc tổng hợp của xe $(v, \omega)$ dưới dạng **`geometry_msgs/msg/TwistStamped`** rồi tự giải phương trình động học nghịch ra tốc độ 2 bánh.

File `mobile_controller/teleop_twiststamped.py` nâng cấp từ bài toán trên bằng cách:
* Khai báo các tham số bước nhảy độc lập cho vận tốc thẳng (`linear_step = 0.05 m/s`, `max_linear_vel = 1.0 m/s`) và vận tốc góc (`angular_step = 0.1 rad/s`, `max_angular_vel = 2.0 rad/s`).
* Tự động gắn nhãn thời gian thực `msg.header.stamp = self.get_clock().now().to_msg()` và `msg.header.frame_id = 'base_link'` vào mỗi gói tin `TwistStamped`.

### Mã nguồn `mobile_controller/teleop_twiststamped.py`:

```python
#!/usr/bin/env python3
# =====================================================================================
#  teleop_keyboard.py - Dieu khien robot qua TwistStamped cho DiffDriveController / MobileController
#
#  Phat tin hieu TwistStamped (linear.x, angular.z) toi topic:
#      /mobile_controller/cmd_vel
#
#  PHIM:
#      w : Tang van toc toi (linear.x +)
#      s : Tang van toc lui (linear.x -)
#      a : Tang van toc quay trai (angular.z +)
#      d : Tang van toc quay phai (angular.z -)
#      SPACE : DUNG NGAY (dua ca linear va angular ve 0)
#      q / Ctrl+C : Thoat
# =====================================================================================

import select
import sys
import termios
import tty

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TwistStamped

# Topic xuat lenh van toc TwistStamped
# CMD_TOPIC = '/mobile_controller/cmd_vel'
CMD_TOPIC = '/cmd_vel'
HELP_TEXT = """
--------------------- TELEOP TWIST STAMPED ---------------------
   w : Tang van toc toi      s : Tang van toc lui
   a : Tang quay trai        d : Tang quay phai
   SPACE : DUNG NGAY         q : Thoat
----------------------------------------------------------------
"""


class TeleopKeyboard(Node):
    """Node publish TwistStamped vao topic cmd_vel cua mobile_controller."""

    def __init__(self):
        # [FIX] Khong hardcode use_sim_time=True nua: khi chay Gazebo hay truyen
        # `--ros-args -p use_sim_time:=true`; khi chay robot that/RViz de mac dinh False.
        super().__init__('teleop_keyboard')

        # Tham so buoc tang va gioi han van toc
        self.declare_parameter('linear_step', 0.05)    # m/s moi lan bam
        self.declare_parameter('angular_step', 0.1)    # rad/s moi lan bam
        self.declare_parameter('max_linear_vel', 1.0)  # m/s
        self.declare_parameter('max_angular_vel', 2.0) # rad/s
        self.declare_parameter('publish_rate', 10.0)   # Hz
        self.declare_parameter('frame_id', 'base_link')

        # Vận tốc hiện tại
        self.linear_x = 0.0
        self.angular_z = 0.0

        self.publisher_ = self.create_publisher(TwistStamped, CMD_TOPIC, 10)
        self.create_timer(1.0 / self.get_parameter('publish_rate').value, self.publish_cmd)

    def publish_cmd(self):
        """Dong goi gia tri thanh TwistStamped bao gom Header và Stamp."""
        msg = TwistStamped()
        
        # Gán thời gian hiện tại và frame ID cho header
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.header.frame_id = self.get_parameter('frame_id').value

        # Gán vận tốc tuyến tính và vận tốc góc
        msg.twist.linear.x = float(self.linear_x)
        msg.twist.linear.y = 0.0
        msg.twist.linear.z = 0.0

        msg.twist.angular.x = 0.0
        msg.twist.angular.y = 0.0
        msg.twist.angular.z = float(self.angular_z)

        self.publisher_.publish(msg)

    def _clamp(self, val, max_val):
        return max(-max_val, min(max_val, val))

    def update_from_key(self, key):
        lin_step = float(self.get_parameter('linear_step').value)
        ang_step = float(self.get_parameter('angular_step').value)
        max_lin = float(self.get_parameter('max_linear_vel').value)
        max_ang = float(self.get_parameter('max_angular_vel').value)

        if key == 'w':
            self.linear_x = self._clamp(self.linear_x + lin_step, max_lin)
        elif key == 's':
            self.linear_x = self._clamp(self.linear_x - lin_step, max_lin)
        elif key == 'a':
            self.angular_z = self._clamp(self.angular_z + ang_step, max_ang)
        elif key == 'd':
            self.angular_z = self._clamp(self.angular_z - ang_step, max_ang)
        elif key == ' ':
            self.linear_x = 0.0
            self.angular_z = 0.0
        else:
            return False

        self.get_logger().info(
            f'linear.x={self.linear_x:+.2f} m/s | angular.z={self.angular_z:+.2f} rad/s',
            throttle_duration_sec=0.2
        )
        # [FIX] Gui lenh ngay khi bam phim (khong phai cho timer)
        self.publish_cmd()
        return True

    def publish_stop(self):
        self.linear_x = 0.0
        self.angular_z = 0.0
        for _ in range(5):
            self.publish_cmd()
            rclpy.spin_once(self, timeout_sec=0.05)


def main(args=None):
    rclpy.init(args=args)
    node = TeleopKeyboard()

    if not sys.stdin.isatty():
        node.get_logger().error(
            'stdin khong phai terminal => khong doc duoc ban phim. '
            'Hay chay truc tiep trong terminal.')
        node.destroy_node()
        rclpy.shutdown()
        return

    settings = termios.tcgetattr(sys.stdin)
    print(HELP_TEXT)

    try:
        tty.setcbreak(sys.stdin.fileno())
        running = True
        while running and rclpy.ok():
            rclpy.spin_once(node, timeout_sec=0.0)
            if select.select([sys.stdin], [], [], 0.05)[0]:
                key = sys.stdin.read(1)
                if key in ('q', '\x03'):
                    running = False
                else:
                    node.update_from_key(key)
    except KeyboardInterrupt:
        pass
    finally:
        node.get_logger().info('Dang dung robot va thoat...')
        node.publish_stop()
        termios.tcsetattr(sys.stdin, termios.TCSADRAIN, settings)
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

[⬅ Quay lại bài: Động học, Odometry và Điều khiển Robot (ros2_control)](/posts/2026/11/dong-hoc-odometry-ros2-control/) | [Bài tiếp: Tutorial 30 - Giới thiệu ros2_control & Bộ điều khiển ➡](/posts/2026/11/ros2-tutorial-30/)
