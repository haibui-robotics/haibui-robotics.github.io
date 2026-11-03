---
layout: single
title: "Tutorial 7: Lập trình ROS 2 Publisher"
permalink: /posts/2026/11/ros2-tutorial-7/
author_profile: true
---

[⬅ Quay lại chuỗi bài: ROS 2 for Beginners](/posts/2026/11/ros2-for-beginners/)

---

## 1. Mục tiêu

Trong bài này, chúng ta sẽ học cách tạo ra một **Publisher** – một thành phần liên tục phát dữ liệu (*message*) lên một kênh giao tiếp (*Topic*) để các Node khác có thể đọc được.

---

## 2. Nội dung cốt lõi: Publisher hoạt động như thế nào?

Để tạo một Publisher trong ROS 2 bằng Python (sử dụng `rclpy` và lập trình hướng đối tượng OOP), chúng ta cần thực hiện 3 bước cốt lõi sau bên trong Class của Node:

### Bước 1: Khởi tạo Publisher (`create_publisher`)
Bạn không thể cứ thế đẩy dữ liệu đi. Bạn phải khai báo cho ROS 2 biết bạn muốn gửi loại dữ liệu gì, vào Topic nào.

```python
self.publisher_ = self.create_publisher(MessageType, 'topic_name', queue_size)
```

* **`MessageType`**: Kiểu dữ liệu của tin nhắn (ví dụ: `String`, `Int32`, `Image`...). Bạn phải import kiểu dữ liệu này từ các thư viện tin nhắn (như `std_msgs`).
* **`'topic_name'`**: Tên của kênh giao tiếp (ví dụ: `'/chatter'`).
* **`queue_size` (QoS)**: Kích thước hàng đợi. Ví dụ để là `10`, nghĩa là nếu mạng bị chậm hoặc Subscriber nhận không kịp, hệ thống sẽ lưu tạm tối đa 10 tin nhắn gần nhất để không bị mất dữ liệu.

### Bước 2: Tạo Timer và Hàm Callback để phát dữ liệu định kỳ
Trong ROS 2, chúng ta không dùng vòng lặp `while True` có chứa `sleep()` để gửi dữ liệu liên tục, vì nó sẽ chặn luồng thực thi của Node. Thay vào đó, ta dùng **Timer**.

```python
self.create_timer(period, self.callback_function)
```

* **`period`**: Chu kỳ gọi hàm (tính bằng giây). Ví dụ `1.0` là cứ 1 giây gọi hàm 1 lần.
* **`self.callback_function`**: Tên hàm sẽ được thực thi mỗi khi hết 1 chu kỳ.

### Bước 3: Định nghĩa hàm Callback và Gửi dữ liệu (`publish`)
Đây là nơi bạn thực sự tạo ra gói tin và phát nó đi.

```python
def callback_function(self):
    msg = MessageType()           # Tạo một đối tượng tin nhắn rỗng
    msg.data = "Dữ liệu của bạn"  # Điền dữ liệu vào tin nhắn
    self.publisher_.publish(msg)  # Đẩy tin nhắn lên Topic
```

---

## 3. Ví dụ cụ thể: Viết Node `first_publisher.py`

Bây giờ, chúng ta sẽ áp dụng 3 bước trên để viết một Node có tên là `first_publisher`. Node này sẽ gửi đi một chuỗi văn bản `"Hello <số đếm>"` lên topic có tên là `/chatter` mỗi giây 1 lần.

**Mã nguồn hoàn chỉnh (`first_publisher.py`):**

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class first_publisher(Node):       
    def __init__(self):           
        # Khởi tạo Node với tên 'first_publisher'
        super().__init__('first_publisher') 
        
        self.cnt = 0
        
        # 1. Tạo Publisher: Kiểu String, Topic '/chatter', Queue size 10
        self.publisher_ = self.create_publisher(String, '/chatter', 10)
        
        # 2. Tạo Timer: Cứ 1.0 giây sẽ gọi hàm self.callback một lần
        self.create_timer(1.0, self.callback)
        
        self.get_logger().info("Sending...")

    # 3. Hàm Callback xử lý việc tạo thông điệp và gửi đi
    def callback(self):
        msg = String()
        msg.data = f'Hello {self.cnt}'
        
        # Đẩy dữ liệu lên Topic
        self.publisher_.publish(msg)
        
        self.cnt += 1
        
        # In log ra màn hình terminal để dễ quan sát
        self.get_logger().info(f'Sended: {msg.data}')

def main(args=None):
    rclpy.init(args=args)            # Khởi tạo thư viện rclpy
    node = first_publisher()         # Khởi tạo đối tượng node
    rclpy.spin(node)                 # Giữ cho node chạy liên tục để chờ các sự kiện (như Timer)
    rclpy.shutdown()                 # Dọn dẹp tài nguyên khi kết thúc

if __name__ == '__main__':
    main()
```

---

## 4. Test

📺 **Video hướng dẫn & chạy thử trên YouTube:**  
👉 [Xem video trên kênh YouTube Hải Bùi](https://www.youtube.com/@HaiBui-tl7gq)

---

[⬅ Quay lại chuỗi bài: ROS 2 for Beginners](/posts/2026/11/ros2-for-beginners/)
