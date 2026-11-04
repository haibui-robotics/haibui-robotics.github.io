---
title: 'ROS 2 for Beginners: Hiểu Nền Tảng ROS 2'
date: 2026-11-03
permalink: /posts/2026/11/ros2-for-beginners/
tags:
  - ROS 2
  - ROS 2 Jazzy
  - Robotics
  - Tutorials
  - Beginners
---

Bài viết mở đầu chuỗi **ROS 2 for Beginners**, giúp bạn nắm vững bản chất của ROS 2, kiến trúc hệ thống và lộ trình 15 bài thực hành từ cơ bản đến làm chủ giao tiếp giữa các tiến trình trong robot.

---

## 1. Mục tiêu

Hiểu bản chất **ROS 2 (Robot Operating System)** là gì, kiến trúc hệ thống và cách các chương trình (tiến trình) trong một robot giao tiếp với nhau để tạo thành một hệ thống thống nhất.

---

## 2. ROS 2 là gì?

ROS **không phải** là một hệ điều hành truyền thống như Windows hay Ubuntu. Nó là một **Middleware (phần mềm trung gian)** cung cấp các thư viện, công cụ và quy chuẩn giao tiếp giúp bạn chế tạo phần mềm cho robot dễ dàng hơn.

ROS 2 sử dụng tiêu chuẩn **DDS (Data Distribution Service)** giúp các thành phần giao tiếp theo thời gian thực (*real-time*) và **không cần một Node trung tâm (Master Node)** như bản ROS 1.

---

## 3. Các khái niệm cốt lõi (Kiến thức cần học)

* **Workspace (Không gian làm việc):** Là thư mục gốc chứa toàn bộ mã nguồn, file cấu hình và các gói phần mềm (packages) của bạn. Giống như một khu công nghiệp chứa nhiều nhà máy.
* **Package (Gói):** Đơn vị tổ chức mã nguồn cơ bản nhất trong ROS. Một package có thể chứa mã nguồn (Python/C++), file cấu hình, file launch, v.v. Một robot thường được chia thành nhiều package (ví dụ: `navigation_pkg`, `vision_pkg`, `control_pkg`).
* **Node (Nút):** Một chương trình thực thi một nhiệm vụ duy nhất (ví dụ: đọc cảm biến LiDAR, điều khiển động cơ, xử lý ảnh). Các node hoạt động độc lập và giao tiếp với nhau.
* **Message (Tin nhắn):** Cấu trúc dữ liệu được định nghĩa sẵn để các Node gửi cho nhau (ví dụ: kiểu dữ liệu chứa tọa độ `x, y, z` hoặc mảng dữ liệu hình ảnh).
* **Topic (Chủ đề):** Kênh giao tiếp một chiều, truyền dữ liệu liên tục.
  * **Publisher:** Node phát dữ liệu lên một Topic.
  * **Subscriber:** Node đăng ký nhận dữ liệu từ Topic đó.
  * **Đặc điểm:** Giao tiếp bất đồng bộ, một Topic có thể có nhiều Publisher và nhiều Subscriber (như một đài phát thanh và các máy thu thanh).
* **Service (Dịch vụ):** Kênh giao tiếp hai chiều theo dạng yêu cầu/phản hồi (*Request/Response*).
  * **Đặc điểm:** Giao tiếp đồng bộ. Một Node (*Client*) gửi yêu cầu và chờ Node kia (*Server*) xử lý rồi trả về kết quả. Phù hợp cho các tác vụ nhanh, thực hiện một lần (ví dụ: Bật/tắt đèn, chụp một bức ảnh).
* **Action (Hành động):** Kênh giao tiếp dành cho các tác vụ mất nhiều thời gian (ví dụ: Robot di chuyển đến tọa độ A, hoặc Robot arm gắp vật thể). Nó giống như Service nhưng có thêm tính năng **Feedback** (cập nhật tiến độ liên tục) và cho phép **Cancel** (hủy) giữa chừng.
* **Parameter (Tham số):** Các biến cấu hình cục bộ của một Node. Cho phép bạn thay đổi cài đặt của Node (như tốc độ tối đa, cổng kết nối) mà không cần biên dịch lại code.
* **QoS (Quality of Service - Chất lượng dịch vụ):** Tính năng mới của ROS 2 giúp quản lý cách thức truyền dữ liệu (ví dụ: ưu tiên độ tin cậy để không mất gói tin, hoặc ưu tiên tốc độ cao cho dữ liệu cảm biến).

---

## 4. Sơ đồ minh họa kiến trúc ROS 2

Để dễ hình dung, hãy xem cách hệ thống thị giác và cánh tay robot giao tiếp:

```text
[ Camera Node ] 
      |
      | (Publisher: Gửi dữ liệu ảnh liên tục)
      v
   Topic: /camera/image_raw  <--- (Message: sensor_msgs/Image)
      |
      | (Subscriber: Nhận dữ liệu ảnh)
      v
[ Vision Node ] (Xử lý AI, phát hiện vật thể và tính toán tọa độ)
      |
      | (Service Client: Gửi yêu cầu tọa độ [x, y, z])
      v
   Service: /grasp_object  <--- (Request: Tọa độ / Response: Thành công/Thất bại)
      |
      | (Service Server: Nhận tọa độ và tính toán động học)
      v
[ Robot Arm Node ] (Điều khiển động cơ gắp vật)
```

---

## 5. Danh sách Thực hành (Tutorials)

*(Nhấn vào từng bài để xem hướng dẫn chi tiết, source code và video YouTube)*

1. [Tutorial 1: Cài đặt và thiết lập môi trường ROS 2 Jazzy](/posts/2026/11/ros2-tutorial-1/)
2. [Tutorial 2: Chạy thử Node ROS 2 đầu tiên (Sử dụng các node có sẵn để test)](/posts/2026/11/ros2-tutorial-2/)
3. [Tutorial 3: Tạo và cấu hình ROS 2 Workspace với colcon](/posts/2026/11/ros2-tutorial-3/)
4. [Tutorial 4: Tạo một ROS 2 Package bằng Python / C++](/posts/2026/11/ros2-tutorial-4/)
5. [Tutorial 5: Viết ROS 2 Node cơ bản bằng Python theo hướng đối tượng (OOP)](/posts/2026/11/ros2-tutorial-5/)
6. [Tutorial 6: Khám phá ROS 2 Topic qua Terminal](/posts/2026/11/ros2-tutorial-6/)
7. [Tutorial 7: Lập trình ROS 2 Publisher](/posts/2026/11/ros2-tutorial-7/)
8. [Tutorial 8: Lập trình ROS 2 Subscriber](/posts/2026/11/ros2-tutorial-8/)
9. [Tutorial 9: Tạo hệ thống vòng kín (Closed-loop) giao tiếp giữa Publisher và Subscriber](/posts/2026/11/ros2-tutorial-9/)
10. [Tutorial 10: Khái niệm và cách kiểm tra ROS 2 Service qua Terminal](/posts/2026/11/ros2-tutorial-10/)
11. [Tutorial 11: Lập trình ROS 2 Service Server & Client bằng Python](/posts/2026/11/ros2-tutorial-11/)
12. [Tutorial 12: Tạo cấu trúc dữ liệu tùy chỉnh (Custom Message & Service - msg, srv)](/posts/2026/11/ros2-tutorial-12/)
13. [Tutorial 13: Khai báo và sử dụng Parameters trong Node](/posts/2026/11/ros2-tutorial-13/)
14. [Tutorial 14: Gom cụm hệ thống với Launch File (Khởi chạy nhiều node cùng lúc)](/posts/2026/11/ros2-tutorial-14/)
