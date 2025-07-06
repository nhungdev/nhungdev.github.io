---
title: "Phân tích giao thức UDS trong Automotive"
date: 2025-07-06 12:00:00 +0700
categories: [Automotive, Protocols]
tags: [uds, automotive, diagnostic, can]
description: Bài viết phân tích chi tiết giao thức UDS (Unified Diagnostic Services) trong ngành công nghiệp Automotive.
---

# Phân tích chi tiết về giao thức UDS trong ngành công nghiệp Automotive

Giao thức **Unified Diagnostic Services (UDS)** là một giao thức chẩn đoán quan trọng trong ngành công nghiệp ô tô, được chuẩn hóa theo **ISO 14229**. Khi được triển khai trên **Controller Area Network (CAN)**, được gọi là **UDS on CAN**, nó sử dụng giao thức truyền tải **ISO-TP (ISO 15765-2)** để hỗ trợ các tác vụ chẩn đoán, cập nhật phần mềm, và kiểm tra hệ thống. Bài viết này cung cấp một phân tích chi tiết về UDS on CAN, bao gồm các khía cạnh từ tổng quan, cấu trúc thông điệp, đến bảo mật và ứng dụng thực tế.

## 1. UDS là gì?

**Unified Diagnostic Services (UDS)** là một giao thức giao tiếp được sử dụng trong các đơn vị điều khiển điện tử (ECUs) của ô tô để hỗ trợ chẩn đoán, cập nhật phần mềm, kiểm tra thói quen, và nhiều tác vụ khác. Được chuẩn hóa theo **ISO 14229**, UDS là một tiêu chuẩn quốc tế, không phụ thuộc vào nhà sản xuất, được phát triển từ các giao thức cũ hơn như **KWP2000 (ISO 14230-3)** và **DoCAN (ISO 15765-3)**. UDS hoạt động ở tầng phiên (Session Layer) và tầng ứng dụng (Application Layer) của mô hình OSI, khác với CAN chỉ sử dụng tầng vật lý và tầng liên kết dữ liệu.

UDS cho phép các công cụ chẩn đoán (tester) giao tiếp với ECUs để thực hiện các tác vụ như:
- Đọc và xóa mã lỗi chẩn đoán (Diagnostic Trouble Codes - DTC).
- Truy xuất thông số xe như số nhận dạng xe (VIN) hoặc trạng thái sạc pin (SoC).
- Cập nhật firmware ECU (flashing).
- Kiểm soát các phiên chẩn đoán và thực hiện các thói quen kiểm tra.

UDS được sử dụng rộng rãi bởi các nhà sản xuất thiết bị gốc (OEM) và nhà cung cấp Tier-1, tích hợp trong các tiêu chuẩn như AUTOSAR ([Wikipedia](https://en.wikipedia.org/wiki/Unified_Diagnostic_Services)).

Theo báo cáo thị trường, thị trường chẩn đoán ô tô được định giá 18,2 tỷ USD vào năm 2023 và dự kiến đạt 32,5 tỷ USD vào năm 2030 với tốc độ tăng trưởng hàng năm (CAGR) là 8,5% ([MarketsandMarkets](https://www.marketsandmarkets.com/Market-Reports/automotive-diagnostic-scan-tools-market-1532.html)).

![UDS Protocol Overview](/assets/img/UDS/uds-protocol-overview.png){: width="800" height="400" .shadow }
_UDS Protocol Overview - Hệ thống chẩn đoán UDS trong ô tô hiện đại_

## 2. Cấu trúc của UDS Message

Thông điệp UDS được thiết kế để truyền qua các giao thức tầng thấp như CAN, sử dụng **ISO-TP** để xử lý dữ liệu lớn. Một thông điệp UDS thường bao gồm các thành phần sau:

| **Thành phần**       | **Mô tả**                                                                 |
|----------------------|---------------------------------------------------------------------------|
| **Service ID (SID)** | Mã 1 byte xác định loại dịch vụ (ví dụ: 0x22 cho ReadDataByIdentifier).   |
| **Sub-function**     | Byte tùy chọn, chỉ định chi tiết yêu cầu (ví dụ: loại báo cáo DTC).       |
| **Data**             | Dữ liệu bổ sung, như Data Identifier (DID) hoặc dữ liệu truyền tải.       |

### Ví dụ:
- Yêu cầu đọc VIN (DID 0xF190): `[0x22] [0xF1 0x90]`
- Phản hồi: `[0x62] [0xF1 0x90] [VIN data]`

Thông điệp UDS được truyền qua ISO-TP, cho phép chia nhỏ dữ liệu lớn thành các khung CAN (8 byte cho CAN tiêu chuẩn, 64 byte cho CAN FD) ([CSS Electronics](https://www.csselectronics.com/pages/uds-protocol-tutorial-unified-diagnostic-services)).

## 3. Các nhóm Service chính

UDS cung cấp nhiều dịch vụ, được nhóm theo chức năng. Dưới đây là các nhóm dịch vụ chính:

| **Nhóm dịch vụ**                     | **SID** | **Mô tả**                                                                 |
|-------------------------------------|---------|---------------------------------------------------------------------------|
| **Diagnostic Session Control**      | 0x10    | Chuyển đổi giữa các phiên chẩn đoán (mặc định, lập trình, mở rộng).       |
| **ECU Reset**                       | 0x11    | Khởi động lại ECU (hard reset, soft reset).                              |
| **ReadDataByIdentifier**            | 0x22    | Đọc dữ liệu theo DID (ví dụ: VIN, SoC).                                   |
| **ReadDiagnosticInformation**       | 0x19    | Đọc mã lỗi chẩn đoán (DTC).                                              |
| **ClearDiagnosticInformation**      | 0x14    | Xóa DTC và dữ liệu liên quan.                                            |
| **RoutineControl**                  | 0x31    | Khởi động, dừng, hoặc kiểm tra kết quả của các thói quen (routines).      |
| **TransferData**                    | 0x36    | Chuyển dữ liệu, thường dùng để flash firmware.                           |
| **RequestTransferExit**             | 0x37    | Kết thúc quá trình truyền dữ liệu.                                       |
| **SecurityAccess**                  | 0x27    | Xác thực để truy cập các dịch vụ bảo mật.                                |

Lưu ý: Các SID từ 0x00 đến 0x0F được dành riêng cho các dịch vụ OBD theo quy định pháp luật ([CSS Electronics](https://www.csselectronics.com/cdn/shop/files/Unified-Diagnostic-Services-UDS-overview-0x22-0x19.png)).

## 4. Tầng truyền tải - ISO-TP (ISO 15765-2)

**ISO-TP (ISO 15765-2)** là giao thức truyền tải được sử dụng để truyền các thông điệp UDS trên CAN, cho phép xử lý dữ liệu lớn (lên đến 4095 byte) bằng cách chia thành nhiều khung CAN. ISO-TP định nghĩa bốn loại khung:

| **Loại khung**         | **Mã** | **Mô tả**                                                                 |
|------------------------|--------|---------------------------------------------------------------------------|
| **Single Frame (SF)**  | 0x0    | Dùng cho dữ liệu nhỏ (<8 byte cho CAN tiêu chuẩn, <64 byte cho CAN FD).   |
| **First Frame (FF)**   | 0x1    | Khung đầu tiên của thông điệp đa khung, chỉ định độ dài dữ liệu.          |
| **Consecutive Frame (CF)** | 0x2 | Các khung tiếp theo của thông điệp đa khung.                             |
| **Flow Control (FC)**  | 0x3    | Điều khiển luồng truyền, chỉ định số khung và thời gian chờ.              |

ISO-TP đảm bảo truyền dữ liệu đáng tin cậy trên CAN, đặc biệt quan trọng khi flash firmware hoặc truyền dữ liệu lớn ([CSS Electronics](https://www.csselectronics.com/cdn/shop/files/CAN-ISO-TP-Frame-Types-Single-First-Consecutive-Flow-Control.svg)).

## 5. UDS trong thực tế

UDS được sử dụng rộng rãi trong các ứng dụng thực tế, bao gồm:
- **Đọc và xóa DTC**: Kiểm tra và xóa mã lỗi từ ECUs.
- **Truy xuất thông số xe**: Đọc VIN (DID 0xF190), tốc độ (DID 0xF40D), hoặc SoC (DID 0x0101 cho Kia EV6).
- **Kiểm soát phiên chẩn đoán**: Chuyển đổi giữa các phiên để thực hiện các tác vụ như lập trình hoặc kiểm tra mở rộng.
- **Flash firmware**: Cập nhật phần mềm ECU.
- **Kiểm tra thói quen**: Thực hiện các bài kiểm tra tự động hoặc cấu hình ECU.

Ví dụ, trong xe điện Kia EV6, UDS được sử dụng để đọc SoC qua DID 0x0101, hỗ trợ quản lý năng lượng ([CSS Electronics](https://www.csselectronics.com/pages/kia-ev6-can-bus-data-uds-dbc)).

## 6. Negative Response Codes (NRC)

Khi một yêu cầu UDS không thể thực hiện, ECU trả về phản hồi tiêu cực với SID **0x7F**, kèm theo mã **Negative Response Code (NRC)** trong byte thứ tư. Một số NRC phổ biến:

| **Mã NRC** | **Mô tả**                              |
|------------|----------------------------------------|
| 0x10       | General Reject                        |
| 0x11       | Service Not Supported                 |
| 0x12       | Sub-function Not Supported            |
| 0x13       | Incorrect Message Length or Format    |
| 0x22       | Conditions Not Correct                |

NRC giúp xác định nguyên nhân lỗi, hỗ trợ kỹ thuật viên xử lý sự cố ([CSS Electronics](https://www.csselectronics.com/cdn/shop/files/Negative-Response-Unified-Diagnostic-Services-0x7F-7F-NRC.svg)).

## 7. UDS Security & Access Levels

> **⚠️ Cảnh báo bảo mật**: UDS có các cơ chế bảo mật tích hợp, nhưng vẫn có thể bị tấn công nếu không được triển khai đúng cách. Các dịch vụ nhạy cảm như flash firmware cần được bảo vệ cẩn thận.
{: .prompt-warning }

UDS bảo vệ các dịch vụ quan trọng (như flash firmware) bằng cơ chế bảo mật:
- **Seed/Key Exchange**: Dịch vụ **SecurityAccess (0x27)** yêu cầu tester cung cấp khóa hợp lệ dựa trên seed do ECU cung cấp.
- **Tester Present**: Tester phải gửi định kỳ thông điệp "tester present" (SID 0x3E) để duy trì phiên chẩn đoán.
- **DiagnosticSessionControl (0x10)**: Chuyển đổi giữa các phiên (mặc định, lập trình, mở rộng) với các cấp độ truy cập khác nhau.

### Rủi ro bảo mật:
- **Unauthorized Access**: Truy cập trái phép vào các dịch vụ nhạy cảm
- **Session Hijacking**: Chiếm quyền điều khiển phiên chẩn đoán
- **Firmware Manipulation**: Thay đổi firmware ECU trái phép

### Giải pháp:
- **Strong Authentication**: Sử dụng thuật toán mã hóa mạnh cho seed/key exchange
- **Session Timeout**: Tự động kết thúc phiên sau thời gian không hoạt động
- **Access Control**: Phân quyền truy cập theo cấp độ bảo mật

Các cơ chế này đảm bảo chỉ các thiết bị được ủy quyền mới có thể truy cập các chức năng nhạy cảm ([ISO 14229-1](https://www.iso.org/standard/72439.html)).

## 8. DID & Routine ID

- **Data Identifier (DID)**: Là mã 2 byte (0x0000–0xFFFF) dùng trong dịch vụ **ReadDataByIdentifier (0x22)** để truy xuất dữ liệu cụ thể. Ví dụ:
  - 0xF190: Vehicle Identification Number (VIN).
  - 0xF40D: Tốc độ (WWH-OBD).
  - 0x0101: State of Charge (SoC) cho Kia EV6.

- **Routine ID**: Được sử dụng trong dịch vụ **RoutineControl (0x31, 0x32)** để thực hiện các thói quen như kiểm tra tự động, cấu hình phần cứng, hoặc các bài kiểm tra đặc biệt. Routine ID thường do nhà sản xuất định nghĩa ([CSS Electronics](https://www.csselectronics.com/pages/can-dbc-file-database-intro#public-dbc-files)).

## 9. UDS Tools

Các công cụ hỗ trợ phát triển, phân tích và kiểm tra UDS bao gồm:
- **Phần mềm**:
  - **CANalyzer**: Phân tích UDS từ Vector với hỗ trợ đầy đủ các dịch vụ UDS.
  - **CANoe**: Môi trường phát triển và kiểm tra UDS từ Vector.
  - **asammdf**: Hiển thị và phân tích dữ liệu UDS ([CSS Electronics](https://www.csselectronics.com/pages/uds-protocol-tutorial-unified-diagnostic-services)).
- **Phần cứng**:
  - **CANedge**: Logger CAN hỗ trợ UDS và ISO-TP ([CSS Electronics](https://www.csselectronics.com/pages/canedge-can-bus-logger)).
  - **Kvaser**: Adapter CAN với hỗ trợ UDS.
- **Công cụ mã nguồn mở**:
  - **Scapy**: Thư viện Python hỗ trợ UDS/ISOTP ([GitHub](https://github.com/iDoka/awesome-canbus)).
  - **python-uds**: Thư viện Python chuyên dụng cho UDS.

## 10. Flash over UDS

UDS hỗ trợ cập nhật firmware ECU (flashing) thông qua các dịch vụ như:
- **TransferData (0x36)**: Chuyển dữ liệu firmware.
- **RequestTransferExit (0x37)**: Kết thúc quá trình truyền.
- **SecurityAccess (0x27)**: Xác thực trước khi flash.

Quá trình flash thường yêu cầu chuyển sang phiên lập trình (programming session) qua **DiagnosticSessionControl (0x10)** và sử dụng ISO-TP để truyền dữ liệu lớn. Đây là một ứng dụng quan trọng trong việc bảo trì và nâng cấp ECU ([CSS Electronics](https://www.csselectronics.com/pages/uds-protocol-tutorial-unified-diagnostic-services)).

## 11. Ứng dụng thực tế

UDS được sử dụng rộng rãi trong các ứng dụng thực tế, bao gồm:
- **Chẩn đoán xe**: Đọc và xóa mã lỗi từ ECUs trong các trung tâm bảo hành.
- **Phát triển xe**: Kiểm tra và cấu hình ECUs trong quá trình phát triển.
- **Bảo trì xe**: Cập nhật firmware và kiểm tra hệ thống định kỳ.
- **Nghiên cứu và phân tích**: Thu thập dữ liệu chẩn đoán cho mục đích nghiên cứu.

Ví dụ: Trong xe điện hiện đại, UDS được sử dụng để đọc thông số pin (SoC, SoH), kiểm tra hệ thống sạc, và cập nhật firmware cho các ECU quản lý năng lượng.

## Kết luận

Giao thức UDS on CAN là một công cụ mạnh mẽ trong ngành công nghiệp ô tô, cung cấp khả năng chẩn đoán, cấu hình, và cập nhật phần mềm cho các ECU một cách thống nhất và đáng tin cậy. Với các dịch vụ đa dạng, cơ chế bảo mật, và hỗ trợ từ ISO-TP, UDS đáp ứng nhu cầu của các nhà sản xuất và kỹ thuật viên trong việc quản lý các hệ thống điện tử phức tạp. Các công cụ như CANedge và phần mềm như CANalyzer hỗ trợ triển khai UDS hiệu quả, đặc biệt trong các ứng dụng thực tế như đọc thông số xe và flash firmware.

**Nguồn tham khảo:**
- [CSS Electronics - UDS Protocol Tutorial](https://www.csselectronics.com/pages/uds-protocol-tutorial-unified-diagnostic-services)
- [Wikipedia - Unified Diagnostic Services](https://en.wikipedia.org/wiki/Unified_Diagnostic_Services)
- [Vector - UDS - Unified Diagnostic Services](https://www.vector.com/de/de/produkte/solutions/diagnose-standards/uds-unified-diagnostic-services-iso14229/)
- [Kvaser - Unified Diagnostic Services](https://kvaser.com/about-can/can-standards/unified-diagnostic-services/)
- [ISO 14229-1](https://www.iso.org/standard/72439.html)