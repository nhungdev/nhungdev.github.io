---
title: "Phân tích giao thức SOME/IP trong Automotive"
date: 2025-07-10 10:00:00 +0700
categories: [Automotive, Protocols]
tags: [someip, automotive, vsomeip, autosar, middleware, embedded]
description: Bài viết giới thiệu tổng quan về giao thức SOME/IP và triển khai vsomeip trong hệ thống ô tô hiện đại.
---

## 1. Giới thiệu

SOME/IP là viết tắt của "**Scalable service-Oriented MiddlewarE over IP**" (Phần mềm trung gian hướng dịch vụ có thể mở rộng qua IP). Middleware này được thiết kế dành riêng cho các trường hợp sử dụng điển hình trong ngành ô tô, với mục tiêu tương thích với **AUTOSAR** (ít nhất ở mức định dạng trên dây - wire-format level).

**Bổ sung:** SOME/IP được phát triển để đáp ứng nhu cầu giao tiếp hiệu quả giữa các đơn vị điều khiển điện tử (ECU) trong xe hơi hiện đại, hỗ trợ các tính năng như lái xe tự hành, kết nối IoT và hệ thống giải trí thông minh. Tính mở rộng (scalability) của nó giúp xử lý số lượng ECU ngày càng tăng trong các phương tiện.

### 1.1. Ba phần chính của đặc tả SOME/IP

- **Định dạng trên dây (On-wire Format)**  
- **Giao thức (Protocol)**  
- **Khám phá dịch vụ (Service Discovery)**  

---

## 2. Định dạng trên dây của SOME/IP

Giao tiếp SOME/IP cơ bản bao gồm các thông điệp được gửi giữa các thiết bị hoặc người đăng ký qua mạng IP. Hãy xem xét hình minh họa sau:

**[Mô tả hình: Sơ đồ cho thấy hai thiết bị A và B trao đổi thông điệp SOME/IP.]**
![SomeIP Communication Protocol](/assets/img/SomeIP/SOMEIPOnWireFormat.jpg)

Trong hình, bạn thấy hai thiết bị (A và B). Thiết bị A gửi một thông điệp SOME/IP đến B và nhận lại một thông điệp phản hồi. Giao thức vận chuyển cơ bản có thể là TCP hoặc UDP; đối với chính thông điệp, điều này không tạo ra sự khác biệt. Giả sử trên thiết bị B đang chạy một dịch vụ cung cấp một hàm, hàm này được thiết bị A gọi thông qua thông điệp, và thông điệp phản hồi là kết quả trả về.

### 2.1. Cấu trúc thông điệp SOME/IP

Thông điệp SOME/IP gồm hai phần chính: **tiêu đề (header)** và **tải trọng (payload)**.

#### **2.1.1. Tiêu đề (Header)**

Tiêu đề chứa các định danh quan trọng, bao gồm:

- **Service ID**: Định danh duy nhất cho mỗi dịch vụ.  
- **Method ID**: 0-32767 cho phương thức, 32768-65535 cho sự kiện.  
- **Length**: Độ dài của payload tính bằng byte (bao gồm thêm 8 byte cho các ID tiếp theo).  
- **Client ID**: Định danh duy nhất cho client gọi bên trong ECU, phải duy nhất trong toàn xe.  
- **Session ID**: Định danh quản lý phiên, tăng dần cho mỗi lần gọi.  
- **Protocol Version**: Hiện tại là 0x01.  
- **Interface Version**: Phiên bản chính của giao diện dịch vụ.  
- **Message Type**: Loại thông điệp, bao gồm:  
  - REQUEST (0x00): Yêu cầu mong đợi phản hồi (kể cả void hoặc không có dữ liệu).  
  - REQUEST_NO_RETURN (0x01): Yêu cầu "gửi và quên" (fire & forget).  
  - NOTIFICATION (0x02): Yêu cầu thông báo/sự kiện không mong đợi phản hồi.  
  - RESPONSE (0x80): Thông điệp phản hồi.  
  - REQUEST_ACK (0x40)  
  - NOTIFICATION_ACK (0x42)  
  - ERROR (0x81)  
  - RESPONSE_ACK (0xC0)  
  - ERROR_ACK (0xC1)  
  - UNKNOWN (0xFF)  
- **Return Code**: Mã trả về, ví dụ:  
  - E_OK (0x00): Không có lỗi.  
  - E_NOT_OK (0x01): Lỗi không xác định.  
  - E_WRONG_INTERFACE_VERSION (0x08): Phiên bản giao diện không khớp.  
  - E_MALFORMED_MESSAGE (0x09): Lỗi giải tuần tự hóa, payload không thể giải mã.  
  - E_WRONG_MESSAGE_TYPE (0x0A): Loại thông điệp không đúng (ví dụ: REQUEST_NO_RETURN cho phương thức định nghĩa là REQUEST).  
  - E_UNKNOWN_SERVICE (0x02)  
  - E_UNKNOWN_METHOD (0x03)  
  - E_NOT_READY (0x04)  
  - E_NOT_REACHABLE (0x05)  
  - E_TIMEOUT (0x06)  
  - E_WRONG_PROTOCOL_VERSION (0x07)  
  - E_UNKNOWN (0xFF)  

Có các loại "REQUESTs" và "RESPONSEs" cho các gọi hàm thông thường, và thông điệp thông báo (NOTIFICATION) cho các sự kiện mà client đã đăng ký. Lỗi được báo cáo như phản hồi hoặc thông báo thông thường nhưng với mã trả về phù hợp.

#### **2.1.2. Tải trọng (Payload)**

Payload chứa dữ liệu đã được tuần tự hóa. Trong trường hợp đơn giản, nếu dữ liệu là một cấu trúc lồng nhau chỉ với các kiểu dữ liệu cơ bản, các phần tử của cấu trúc sẽ được "làm phẳng" và ghi lần lượt vào payload.

---

## 3. Giao thức SOME/IP

Phần này tập trung vào hai điểm chính:

1. **Ràng buộc vận chuyển (Transport Bindings)**: UDP và TCP.  
2. **Mẫu giao tiếp cơ bản**: Publish/Subscribe và Request/Response.  

### 3.1. Ràng buộc vận chuyển

Giao thức vận chuyển cơ bản có thể là UDP hoặc TCP:  
- **UDP**: Thông điệp SOME/IP không bị phân mảnh; một gói UDP có thể chứa nhiều thông điệp, nhưng thông điệp không được dài quá 1400 byte.  
- **TCP**: Dùng cho thông điệp lớn hơn, tận dụng các tính năng mạnh mẽ của TCP. Nếu xảy ra lỗi đồng bộ trong luồng TCP, SOME/IP sử dụng "magic cookies" để tìm lại điểm bắt đầu của thông điệp tiếp theo.  

**Lưu ý:** Giao diện dịch vụ cần được khởi tạo, và mỗi phiên bản giao diện được xác định bằng số cổng của giao thức vận chuyển (instance ID không nằm trong tiêu đề thông điệp). Do đó, không thể có nhiều phiên bản của cùng một giao diện trên cùng một cổng.

### 3.2. Mẫu giao tiếp

**[Mô tả hình: Sơ đồ minh họa các mẫu giao tiếp Request/Response và Publish/Subscribe.]**
![SomeIP Communication Protocol](/assets/img/SomeIP/SOMEIPProtocol.jpg)

Ngoài cơ chế REQUEST/RESPONSE tiêu chuẩn cho gọi thủ tục từ xa, SOME/IP còn hỗ trợ mẫu PUBLISH/SUBSCRIBE cho các sự kiện. Sự kiện trong SOME/IP luôn được nhóm thành **event group**, nên client chỉ có thể đăng ký vào nhóm sự kiện, không phải từng sự kiện riêng lẻ. Ngoài ra, SOME/IP hỗ trợ "fields" với setter/getter theo mẫu REQUEST/RESPONSE và thông báo thay đổi qua sự kiện. Việc đăng ký được thực hiện qua khám phá dịch vụ SOME/IP.

---

## 4. Khám phá dịch vụ SOME/IP

Khám phá dịch vụ (Service Discovery) dùng để định vị các phiên bản dịch vụ, kiểm tra xem chúng có đang chạy không, và xử lý mẫu Publish/Subscribe. Điều này chủ yếu được thực hiện qua:  
- **Offer messages**: Thiết bị phát sóng (multicast) các thông điệp chứa danh sách dịch vụ mà nó cung cấp.  
- **Find messages**: Nếu client cần dịch vụ chưa được cung cấp, nó có thể gửi thông điệp tìm kiếm.  

Thông điệp SD của SOME/IP được gửi qua UDP và hỗ trợ cả việc publish/subscribe nhóm sự kiện.

**[Mô tả hình: Sơ đồ cấu trúc thông điệp SD SOME/IP.]**
![SomeIP Communication Protocol](/assets/img/SomeIP/SOMEIPServiceDiscovery.jpg)

<!-- <p style="text-align: center;"><em>Hình 5: Cấu trúc push-pull và open-drain</em></p> -->

Phần này đủ để bắt đầu. Chi tiết hơn sẽ được thảo luận trong các ví dụ hoặc trong đặc tả chính thức.

---

## 5. Tổng quan về vsomeip

**vsomeip** là triển khai mã nguồn mở của SOME/IP do **COVESA** phát triển, hỗ trợ cả giao tiếp giữa các thiết bị (external communication) và giao tiếp liên tiến trình nội bộ (internal inter-process communication).  

**[Mô tả hình: Sơ đồ kiến trúc vsomeip.]**
![SomeIP Communication Protocol](/assets/img/SomeIP/vsomeipOverview.jpg)

### 5.1. Các thành phần chính

- **Communication Endpoints**: Xác định giao thức (TCP/UDP) và các thông số như số cổng, được cấu hình qua file JSON.  
- **Local Endpoints**: Dùng unix domain sockets (thông qua Boost.Asio) cho giao tiếp nội bộ nhanh chóng.  
- **Routing Manager**: Quản lý định tuyến thông điệp, chỉ có một trên mỗi thiết bị. Nếu không cấu hình, ứng dụng vsomeip đầu tiên sẽ khởi động nó.  

**Lưu ý:** vsomeip không triển khai tuần tự hóa dữ liệu (được xử lý bởi CommonAPI). Nó chỉ hỗ trợ giao thức SOME/IP và khám phá dịch vụ.

---

## 6. SOME/IP trong ngành ô tô

### 6.1. Tổng quan

SOME/IP đã trở thành giao thức quan trọng trong ngành ô tô hiện đại, đặc biệt với sự phát triển của xe điện và xe tự lái. Giao thức này hỗ trợ kiến trúc hướng dịch vụ (SOA) cho phép các ECU giao tiếp linh hoạt và hiệu quả.

### 6.2. Ứng dụng cụ thể

- **Hệ thống ADAS**: SOME/IP truyền dữ liệu từ cảm biến radar, camera đến bộ xử lý trung tâm.
- **Hệ thống giải trí**: Kết nối màn hình cảm ứng, hệ thống âm thanh với bộ điều khiển chính.
- **Hệ thống điều khiển động cơ**: Giao tiếp giữa các ECU điều khiển động cơ, hộp số.
- **Hệ thống kết nối**: Kết nối xe với mạng di động và các dịch vụ đám mây.

### 6.3. Lợi ích

- **Khả năng mở rộng**: Hỗ trợ số lượng lớn ECU và dịch vụ.
- **Tương thích AUTOSAR**: Tuân thủ tiêu chuẩn ngành ô tô.
- **Hiệu suất cao**: Tối ưu hóa cho môi trường thời gian thực.
- **Linh hoạt**: Hỗ trợ cả TCP và UDP tùy theo yêu cầu ứng dụng.

---

## 7. Kết luận

SOME/IP là một giải pháp quan trọng cho giao tiếp trong ô tô hiện đại, với khả năng mở rộng và hỗ trợ kiến trúc hướng dịch vụ (SOA). Triển khai vsomeip cung cấp nền tảng mã nguồn mở mạnh mẽ cho việc phát triển các ứng dụng SOME/IP. Giao thức này tiếp tục đóng vai trò quan trọng trong việc kết nối các hệ thống phức tạp trong xe hơi hiện đại.

---

## 8. Tài liệu tham khảo

- [SOME/IP Official Specification](https://some-ip.com/)
- [vsomeip Documentation](https://github.com/COVESA/vsomeip)
- [AUTOSAR SOME/IP Service Discovery](https://www.autosar.org/fileadmin/user_upload/standards/classic/4-3/AUTOSAR_SWS_ServiceDiscovery.pdf)
- [Automotive Ethernet and SOME/IP](https://www.automotive-ethernet.com/some-ip/)