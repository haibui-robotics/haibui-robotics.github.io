---
layout: single
title: "Tutorial 6: Lập trình ROS 2 Subscriber"
permalink: /posts/2026/11/ros2-tutorial-6/
author_profile: true
---

[⬅ Quay lại chuỗi bài: ROS 2 for Beginners](/posts/2026/11/ros2-for-beginners/) | [Bài tiếp: Tutorial 7 - Lập trình ROS 2 Publisher ➡](/posts/2026/11/ros2-tutorial-7/)

---

## 1. Cốt lõi của một Subscriber

Ngược lại với Publisher, **Subscriber** là một Node thụ động. Nó không cần Timer để hoạt động định kỳ, mà sẽ "ngồi chờ" dữ liệu gửi đến. Để tạo Subscriber, ta cần 2 bước chính:

### Bước 1: Khởi tạo (`create_subscription`)
Khai báo kiểu dữ liệu, tên Topic cần nghe, hàm xử lý khi có dữ liệu đến (*Callback*) và kích thước hàng đợi.

```python
self.subscriber_ = self.create_subscription(MessageType, 'topic_name', self.callback, queue_size)
```

### Bước 2: Viết hàm Callback xử lý dữ liệu
Hàm này sẽ tự động được gọi mỗi khi có một tin nhắn mới xuất hiện trên Topic. Hàm phải nhận vào một tham số là gói tin (`msg`).

```python
def callback(self, msg: MessageType):
    # Lấy dữ liệu ra từ msg và xử lý (ví dụ: in ra màn hình)
    print(msg.data)
```

---

## 2. Ví dụ thực hành

Node `first_subcriber` dưới đây sẽ đăng ký nhận dữ liệu từ Topic `/chatter` (Topic mà chúng ta đã tạo ở [Tutorial 7](/posts/2026/11/ros2-tutorial-7/)) và in nội dung ra màn hình mỗi khi nhận được tin.

**Mã nguồn (`first_subcriber.py`):**

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class first_subcriber(Node):       
    def __init__(self):           
        super().__init__('first_subcriber') # Đặt tên node
        
        # 1. Khởi tạo Subscriber: Kiểu String, Topic '/chatter', gọi hàm self.callback, Queue size 10
        self.subscriber_ = self.create_subscription(String, '/chatter', self.callback, 10)

    # 2. Hàm Callback được tự động gọi khi có tin nhắn mới tới
    def callback(self, msg: String):
        # Trích xuất dữ liệu (msg.data) và in ra terminal
        self.get_logger().info(f'Received: {msg.data}')

def main(args=None):
    rclpy.init(args=args)
    node = first_subcriber()        
    
    # Quan trọng: Giữ Node chạy liên tục để luôn lắng nghe dữ liệu từ Topic
    rclpy.spin(node)
    
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

[⬅ Quay lại chuỗi bài: ROS 2 for Beginners](/posts/2026/11/ros2-for-beginners/) | [Bài tiếp: Tutorial 7 - Lập trình ROS 2 Publisher ➡](/posts/2026/11/ros2-tutorial-7/)
