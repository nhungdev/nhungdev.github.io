---
title: "Phân tích giao thức CAN trong Automotive"
date: 2025-07-06 10:00:00 +0700
categories: [Automotive, Protocols]
tags: [can, automotive]
description: Bài viết phân tích chi tiết giao thức CAN trong ngành công nghiệp Automotive.
---

# Phân tích chi tiết về giao thức CAN trong ngành công nghiệp Automotive

Giao thức **Controller Area Network (CAN)** là một trong những giao thức giao tiếp quan trọng nhất trong ngành công nghiệp ô tô, được sử dụng để kết nối các đơn vị điều khiển điện tử (ECUs - Electronic Control Units) trong xe. Được phát triển bởi Bosch vào những năm 1980, CAN đã trở thành tiêu chuẩn quốc tế (ISO 11898) và được áp dụng rộng rãi không chỉ trong ô tô mà còn trong các lĩnh vực như tự động hóa công nghiệp, thiết bị y tế, và hàng không vũ trụ. Bài viết này cung cấp một phân tích chi tiết về giao thức CAN, bao gồm các khía cạnh từ tổng quan, cấu trúc frame, đến bảo mật, theo yêu cầu của người dùng.

## 1. Tổng quan về CAN

**CAN (Controller Area Network)** là một giao thức giao tiếp đa bậc thầy (multi-master) được thiết kế để cho phép các thiết bị điện tử trong xe hơi giao tiếp với nhau mà không cần máy chủ trung tâm. Trước khi CAN ra đời, các nhà sản xuất ô tô sử dụng hệ thống dây điện điểm-điểm, dẫn đến dây cáp cồng kềnh, nặng và tốn kém. CAN đã thay thế hệ thống này bằng một mạng bus nối tiếp, giảm đáng kể số lượng dây điện, chi phí và trọng lượng.

### Đặc điểm chính của CAN:
- **Đa bậc thầy**: Nhiều node (ECUs) có thể gửi tin nhắn trên cùng một bus mà không cần phối hợp trước.
- **Khả năng chống nhiễu**: CAN sử dụng tín hiệu vi phân (differential signaling) với hai dây (CAN_H và CAN_L), giúp chống lại nhiễu điện từ, rất phù hợp cho môi trường ô tô khắc nghiệt.
- **Tốc độ truyền**: CAN tiêu chuẩn hỗ trợ tốc độ lên đến 1 Mbit/s, trong khi CAN FD (Flexible Data-Rate) đạt tới 8 Mbit/s.
- **Khả năng phát hiện lỗi**: CAN có cơ chế phát hiện và xử lý lỗi mạnh mẽ, đảm bảo tính toàn vẹn dữ liệu.
- **Ứng dụng rộng rãi**: Ngoài ô tô, CAN được sử dụng trong tự động hóa công nghiệp, thiết bị y tế, và hàng không vũ trụ.

Theo báo cáo thị trường, thị trường thiết bị CAN Bus được định giá 10,3 tỷ USD vào năm 2023 và dự kiến đạt 15,5 tỷ USD vào năm 2030 với tốc độ tăng trưởng hàng năm (CAGR) là 12,6% ([Logic Fruit](https://www.logic-fruit.com/blog/can/controller-area-network-can-guide/)).

![CAN Protocol Overview](/assets/img/CAN/can-protocol-overview.png){: width="800" height="400" .shadow }
_CAN Protocol Overview - Hệ thống giao tiếp CAN trong ô tô hiện đại_

## 2. Cấu trúc frame trong CAN

Frame CAN là đơn vị cơ bản để truyền dữ liệu trên bus. Có hai loại frame chính: **Base Frame (CAN 2.0A)** với Identifier 11 bit và **Extended Frame (CAN 2.0B)** với Identifier 29 bit. Dưới đây là cấu trúc chi tiết của một frame CAN:

### Cấu trúc Base Frame (CAN 2.0A):

| **Trường**                | **Độ dài (bit)** | **Mô tả**                                                                 |
|---------------------------|------------------|---------------------------------------------------------------------------|
| Start of Frame (SOF)      | 1                | Bit dominant (0) báo hiệu bắt đầu frame.                                  |
| Arbitration Field         | 12               | Bao gồm Identifier (11 bit) và RTR (1 bit, dominant cho Data Frame).      |
| Control Field             | 6                | Bao gồm IDE (1 bit, 0 cho Base Frame), r0 (1 bit), DLC (4 bit).           |
| Data Field                | 0–64 (0–8 byte)  | Chứa dữ liệu thực tế, tối đa 8 byte.                                      |
| CRC Field                 | 15               | Cyclic Redundancy Check để phát hiện lỗi.                                 |
| ACK Field                 | 2                | ACK Slot (1 bit, recessive) và ACK Delimiter (1 bit, recessive).          |
| End of Frame (EOF)        | 7                | 7 bit recessive báo hiệu kết thúc frame.                                  |
| Interframe Space (IFS)    | 3                | Khoảng trống giữa các frame, 3 bit recessive.                            |

### Cấu trúc Extended Frame (CAN 2.0B):
- Tương tự Base Frame, nhưng Arbitration Field dài 32 bit:
  - Identifier A: 11 bit
  - Substitute Remote Request (SRR): 1 bit (recessive)
  - Identifier Extension (IDE): 1 bit (recessive)
  - Identifier B: 18 bit
  - RTR: 1 bit
  - Reserved Bits (r1, r0): 2 bit
- Độ dài tối đa: 157 bit sau bit stuffing.

### Lưu ý:
- **Bit stuffing**: Sau 5 bit liên tiếp giống nhau, một bit ngược lại được chèn để đảm bảo đồng bộ. Điều này làm tăng độ dài frame tối đa lên 132 bit (Base Frame) hoặc 157 bit (Extended Frame).
- **Remote Frame**: Tương tự Data Frame nhưng không có Data Field, dùng để yêu cầu dữ liệu từ node khác.

Cấu trúc frame đảm bảo truyền dữ liệu đáng tin cậy và hỗ trợ phát hiện lỗi hiệu quả ([Wikipedia](https://en.wikipedia.org/wiki/CAN_bus)).

## 3. CAN Bus Arbitration

Arbitration là quá trình giải quyết xung đột khi nhiều node muốn truyền dữ liệu cùng lúc. CAN sử dụng phương pháp **bitwise arbitration không phá hủy**, đảm bảo tin nhắn có độ ưu tiên cao được truyền mà không mất dữ liệu.

### Quy trình arbitration:
- Mỗi node bắt đầu gửi frame từ SOF.
- Trong Arbitration Field, các node so sánh Identifier của mình với tín hiệu trên bus.
- Bit **dominant (0)** thắng bit **recessive (1)**. Node có Identifier nhỏ hơn (nhiều bit 0 hơn ở vị trí đầu tiên khác nhau) sẽ thắng và tiếp tục truyền.
- Node thua arbitration dừng truyền và chờ đến lượt tiếp theo (sau 6 bit clock).

### Ví dụ:
- Node A: Identifier 00000001111 (15).
- Node B: Identifier 00000010000 (16).
- Tại bit thứ 4, Node A gửi 0 (dominant), Node B gửi 1 (recessive). Node A thắng, Node B dừng truyền.

Arbitration đảm bảo tin nhắn ưu tiên cao (như phanh ABS) được truyền trước, rất quan trọng trong các hệ thống thời gian thực ([Wikipedia](https://en.wikipedia.org/wiki/CAN_bus)).

## 4. CAN Bit Timing & Baudrate

Bit timing và baudrate là các yếu tố quan trọng để đảm bảo đồng bộ giữa các node trên bus CAN.

### Bit Timing:
Mỗi bit được chia thành các **time quanta** (thường 8–25 quanta), bao gồm:
- **Synchronization Segment (Sync_Seg)**: 1 quantum, dùng để đồng bộ hóa.
- **Propagation Time Segment (Prop_Seg)**: 1–8 quanta, bù đắp độ trễ truyền tín hiệu.
- **Phase Buffer Segment 1 (Phase_Seg1)**: 1–8 quanta, điều chỉnh pha.
- **Phase Buffer Segment 2 (Phase_Seg2)**: 1–8 quanta, điều chỉnh pha bổ sung.

**Đồng bộ hóa**:
- **Hard synchronization**: Xảy ra tại SOF (recessive sang dominant).
- **Resynchronization**: Xảy ra tại mỗi chuyển tiếp recessive sang dominant trong frame.

### Baudrate:
- **CAN 2.0**: Tốc độ từ 10 kbit/s đến 1 Mbit/s, phù hợp với chiều dài bus từ 40 m (1 Mbit/s) đến 500 m (125 kbit/s).
- **CAN FD**: Lên đến 8 Mbit/s, với tốc độ biến đổi trong frame (arbitration chậm, dữ liệu nhanh).
- **CAN XL**: Lên đến 20 Mbit/s, hỗ trợ payload 2,048 byte ([Wikipedia](https://en.wikipedia.org/wiki/CAN_bus)).

Tất cả các node phải có cùng baudrate để giao tiếp chính xác.

## 5. CAN Error Handling

CAN có cơ chế phát hiện và xử lý lỗi mạnh mẽ để đảm bảo tính toàn vẹn dữ liệu.

### Các loại lỗi:

| **Loại lỗi**            | **Mô tả**                                                                 |
|-------------------------|---------------------------------------------------------------------------|
| Bit Error               | Bit truyền không khớp với bit nhận.                                       |
| Stuff Error             | 6 bit liên tiếp giống nhau (vi phạm bit stuffing).                        |
| CRC Error               | Sai lệch trong Cyclic Redundancy Check.                                   |
| Form Error              | Sai lệch trong cấu trúc frame (ví dụ: EOF không đúng).                    |
| Acknowledgment Error    | Không nhận được tín hiệu xác nhận từ node khác.                           |

### Xử lý lỗi:
- Khi phát hiện lỗi, node gửi **Error Frame** (6–12 bit dominant/recessive, tiếp theo là 8 bit recessive).
- **Trạng thái lỗi**:
  - **Error-Active** (TEC/REC < 128): Gửi Error Frame chủ động.
  - **Error-Passive** (TEC/REC 128–255): Gửi Error Frame thụ động.
  - **Bus-Off** (TEC > 255): Node ngừng truyền/nhận.
- **Bộ đếm lỗi**:
  - **TEC (Transmit Error Counter)**: Tăng khi gửi lỗi.
  - **REC (Receive Error Counter)**: Tăng khi nhận lỗi.

Cơ chế này đảm bảo mạng CAN hoạt động ổn định ngay cả khi có lỗi ([Wikipedia](https://en.wikipedia.org/wiki/CAN_bus)).

## 6. CAN Driver & Stack

- **CAN Driver**: Là phần cứng (CAN transceiver) chuyển đổi tín hiệu logic từ microcontroller thành tín hiệu vi phân trên bus CAN. Ví dụ: TJA1041/TJA1043 từ NXP.
- **CAN Stack**: Là phần mềm triển khai giao thức CAN, bao gồm:
  - Quản lý frame (gửi/nhận).
  - Xử lý arbitration và lỗi.
  - Hỗ trợ các giao thức cấp cao hơn.
- Ví dụ stack:
  - **NI-XNET**: Hỗ trợ CAN, LIN, FlexRay, giảm độ trễ với DMA ([NI](https://www.ni.com/en/shop/seamlessly-connect-to-third-party-devices-and-supervisory-system/ni-xnet-can--lin--and-flexray-platform-overview.html)).
  - **NI-CAN**: Dành cho giao diện CAN cũ, hỗ trợ Frame API và Channel API.

## 7. Protocols chạy trên CAN

CAN là nền tảng cho nhiều giao thức cấp cao, phục vụ các ứng dụng cụ thể:

| **Giao thức** | **Ứng dụng**                          | **Tổ chức/Thông tin**                     |
|---------------|---------------------------------------|-------------------------------------------|
| CANopen       | Tự động hóa công nghiệp, lift control | CAN in Automation ([CANopen Lift](https://en.canopen-lift.org/wiki/)) |
| DeviceNet     | Tự động hóa công nghiệp               | ODVA                                      |
| J1939         | Xe nặng, hàng hải                    | SAE                                       |
| ISO 15765-2   | Chẩn đoán ô tô (UDS)                 | ISO                                       |
| CANaerospace  | Hàng không                           | Stock                                     |
| UAVCAN        | Robot, hàng không vũ trụ             | UAVCAN                                    |

Các giao thức này bổ sung chức năng cấp cao, như quản lý thiết bị hoặc chẩn đoán ([Wikipedia](https://en.wikipedia.org/wiki/CAN_bus)).

## 8. CAN Tools

Các công cụ hỗ trợ phát triển, phân tích và kiểm tra CAN bao gồm:
- **Phần mềm**:
  - **CanKing**: Công cụ giám sát CAN miễn phí từ Kvaser ([Kvaser](https://kvaser.com/canking/)).
  - **CANalyzer**: Phân tích CAN từ Vector.
  - **asammdf**: Hiển thị và phân tích dữ liệu CAN ([CSS Electronics](https://www.csselectronics.com/pages/can-bus-simple-intro-tutorial)).
- **Phần cứng**:
  - **CANable**: Adapter USB-CAN mã nguồn mở ([CANable](https://canable.io/)).
  - **CAN Bus Analyzer**: Thiết bị đo tín hiệu CAN từ Microchip ([Microchip](https://www.microchip.com/en-us/development-tool/apgdt002)).
- **Công cụ mã nguồn mở**:
  - **Scapy**: Thư viện Python hỗ trợ CAN/ISOTP/UDS ([GitHub](https://github.com/iDoka/awesome-canbus)).
  - **CANalyzat0r**: Phân tích bảo mật CAN.

## 9. Ứng dụng thực tế

CAN được sử dụng rộng rãi trong nhiều lĩnh vực:
- **Ô tô**: Kết nối ECUs cho động cơ, phanh ABS, túi khí, hệ thống giải trí ([AutoPi](https://www.autopi.io/blog/can-bus-explained/)).
- **Tự động hóa công nghiệp**: Điều khiển máy móc và robot.
- **Y tế**: Thiết bị giám sát và chẩn đoán.
- **Hàng không vũ trụ**: Điều khiển CubeSat và hệ thống hàng không ([SpringerLink](https://link.springer.com/chapter/10.1007/0-8176-4404-0_32)).

Ví dụ: Trong ô tô, khi nhấn bàn đạp phanh, CAN gửi tín hiệu từ cảm biến bàn đạp đến hệ thống ABS và đèn phanh, đảm bảo phản hồi đồng thời và an toàn.

## 10. CAN Security

CAN không có tính năng bảo mật tích hợp như mã hóa hay xác thực, khiến nó dễ bị tấn công, đặc biệt trong các phương tiện kết nối internet.

### Rủi ro bảo mật:
- **Injection Attack**: Gửi tin nhắn giả để điều khiển ECUs ([Medium](https://medium.com/@chaincom/understanding-can-bus-vulnerabilities-and-how-blockchain-can-amplify-security-a58388bf1fb4)).
- **Replay Attack**: Lặp lại tin nhắn hợp lệ để gây rối.
- **Bus-Off Attack**: Gây lỗi để đưa node vào trạng thái Bus-Off ([VicOne](https://vicone.com/blog/how-to-get-away-with-car-theft-unveiling-the-dark-side-of-the-can-bus)).

### Giải pháp:
- **Phân đoạn mạng**: Chia bus thành các subnetwork để giới hạn phạm vi tấn công ([PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC7219335/)).
- **Intrusion Detection Systems (IDS)**: Sử dụng thuật toán học máy để phát hiện hành vi bất thường ([Wikipedia](https://en.wikipedia.org/wiki/CAN_bus)).
- **Mã hóa nhẹ**: Nghiên cứu mã hóa không ảnh hưởng hiệu suất.
- **Cổng an toàn**: Lọc và xác thực tin nhắn giữa các mạng.
- **Zero-Trust Architecture**: Xác minh mọi giao tiếp ([Wikipedia](https://en.wikipedia.org/wiki/CAN_bus)).
- **Tiêu chuẩn hóa**: Áp dụng SAE J3061 và ISO/SAE 21434 để tăng cường bảo mật.

### Ví dụ thực tế:
Năm 2022, một nhà nghiên cứu bảo mật (Ian Tabor) phát hiện xe của mình bị đánh cắp thông qua tấn công CAN, cho thấy sự cần thiết của các biện pháp bảo mật ([VicOne](https://vicone.com/blog/how-to-get-away-with-car-theft-unveiling-the-dark-side-of-the-can-bus)).

## Kết luận

Giao thức CAN là một nền tảng quan trọng trong ngành ô tô, cung cấp giao tiếp đáng tin cậy, hiệu quả và tiết kiệm chi phí. Tuy nhiên, với sự gia tăng của các phương tiện kết nối, bảo mật CAN đang trở thành một thách thức lớn. Các giải pháp như phân đoạn mạng, IDS, và mã hóa nhẹ đang được phát triển để giải quyết vấn đề này. Với các công cụ như CanKing, CANalyzer, và các giao thức cấp cao như CANopen, CAN tiếp tục là một phần không thể thiếu trong các hệ thống hiện đại.