---
layout: single
title: "Tutorial 15: Khái niệm TF2 và Cây tọa độ (TF Tree)"
permalink: /posts/2026/11/ros2-tutorial-15/
author_profile: true
---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/)

---

## 1. TF (Transform) là gì? Tại sao Robot cần TF?

Hãy tưởng tượng bạn là một con robot. Mắt (**Camera**) của bạn nhìn thấy một quả táo cách 1 mét. Bây giờ bạn muốn dùng tay (**Cánh tay robot**) để nhặt quả táo đó lên.

* **Vấn đề:** Camera biết quả táo cách nó 1m. Nhưng cánh tay robot lại nằm ở vị trí khác (thấp hơn, lùi về sau). Làm sao cánh tay biết quả táo cách nó bao xa để với tới?
* **Giải pháp:** Chúng ta cần biết chính xác khoảng cách và góc xoay từ Camera đến Cánh tay.

Trong ROS 2, mọi bộ phận của robot (tâm xe, bánh xe, LiDAR, Camera) đều có một **Hệ trục tọa độ (Coordinate Frame)** riêng với gốc $(0,0,0)$ nằm tại chính nó.

**TF2 (Transform)** là hệ thống giúp ROS 2 liên tục theo dõi và tính toán khoảng cách (**Translation:** $x, y, z$) cùng góc xoay (**Rotation:** $\text{roll}, \text{pitch}, \text{yaw}$) giữa tất cả các hệ trục tọa độ này theo thời gian thực. Nhờ TF, LiDAR quét được tường ở đâu, ROS 2 có thể tự động quy đổi khoảng cách đó về tâm của robot để né tránh.

---

## 2. Bản chất của TF Tree (Cây tọa độ)

Hệ thống TF quản lý các hệ trục tọa độ theo cấu trúc hình cây (**Tree**). Gồm có:
* **Parent Frame (Tọa độ cha):** Trục gốc để tham chiếu.
* **Child Frame (Tọa độ con):** Trục được gắn liền và xác định vị trí dựa trên trục cha.

**Ví dụ một nhánh TF cơ bản trên xe tự hành:**

`odom` *(Gốc không gian)* → `base_link` *(Tâm robot)* → `laser_link` *(Cảm biến LiDAR)*

* `laser_link` là con của `base_link`: Cho biết LiDAR được lắp ở đâu trên thân xe.
* `base_link` là con của `odom`: Cho biết robot đang di chuyển đến tọa độ nào trong môi trường.

> **Quy tắc tối thượng của TF Tree:** Một **Parent** có thể có nhiều **Child**, nhưng một **Child** chỉ được phép có **duy nhất một Parent**. Tuyệt đối không có vòng lặp khép kín.

---

## 3. Hai loại TF bạn cần nắm rõ

Khi thiết kế robot, bạn sẽ gặp 2 loại TF phát trên hệ thống:

1. **Static TF (Biến đổi tĩnh):** Dành cho các bộ phận được bắt ốc vít cố định, không bao giờ thay đổi vị trí.
   * *Ví dụ:* Khoảng cách từ tâm thân xe (`base_link`) đến cảm biến LiDAR (`laser_link`).
2. **Dynamic TF (Biến đổi động):** Dành cho các hệ trục liên tục thay đổi vị trí theo thời gian.
   * *Ví dụ:* Khoảng cách từ tâm môi trường (`odom`) đến tâm thân xe di chuyển (`base_link`), hoặc các khớp xoay của cánh tay robot.

---

## 4. Thực hành: Trực quan hóa TF Tree trên Robot thực tế

*(Để hiểu rõ nhất cách TF Tree hoạt động, hãy xem video minh họa dưới đây)*

Trong video này, tôi sẽ sử dụng một mô hình xe robot đã được tạo sẵn:

📺 **Video hướng dẫn & minh họa trực quan trên YouTube:**  
👉 [Xem video trên kênh YouTube Hải Bùi](https://www.youtube.com/@HaiBui-tl7gq)

---

[⬅ Quay lại bài: Mô phỏng Robot với ROS 2](/posts/2026/11/mo-phong-robot-ros2/)
