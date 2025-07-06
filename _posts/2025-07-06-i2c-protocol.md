---
title: "Phân tích giao thức I2C trong Embedded Systems"
date: 2025-07-06 10:00:00 +0700
categories: [Automotive, Protocols]
tags: [i2c, embedded, automotive, linux, zynqmp, zcu102]
description: Bài viết phân tích chi tiết giao thức I2C trong hệ thống nhúng và ứng dụng thực tế trên bo mạch ZCU102.
---


## 1. Giới thiệu

Để thực hiện việc giao tiếp giữa Processor với các ngoại vi sẽ cần đến các giao thức phù hợp, với
từng ngoại vi cụ thể sẽ có từng kiểu giao thức tương ứng khác nhau. Có thể kể đến như PCI, UART,
SPI, ... Tài liệu này được viết phục vụ mục đích tìm hiểu về giao thức I2C, topology I2C, SMBus, cách
triển khai phần cứng, nguyên lý hoạt động, tìm hiểu và nghiên cứu sâu hơn về các cấu trúc của I2C
Subsystem, SMBus trong Linux, thêm vào đó là bộ công cụ i2c-tools trên dòng chip ZynqMP cũng như
trên bo mạch ZCU102.

---

## 2. Giao thức I2C

I2C hay IIC (Inter-Integrated Circuit, eye-squared-C), là một bus giao tiếp nối tiếp đồng bộ sử dụng
2 dây, đa bộ điều khiển/đa mục tiêu (chính/phụ), chuyển mạch gói, được phát minh vào năm 1982 bởi
Philips Semiconductors.

### 2.1. Giới thiệu về giao thức I2C

Giao thức I2C là một giao thức phổ biến được sử dụng để giao tiếp với nhiều loại thiết bị cảm biến,
ví dụ như ADXL345, MLX90614, ... Ngoài ra, giao thức I2C được coi là giao thức kết hợp những đặc tính
ưu việt nhất của 2 giao thức SPI và UART.
Giao thức I2C cho phép kết nối nhiều slave tới một master hoặc nhiều master điều khiển một slave.
Điều này thật sự hữu ích khi cần sử dụng nhiều processor. Như đã đề cập từ trước, giao thức I2C chỉ sử
dụng 2 dây để vận chuyển dữ liệu giữa các thiết bị. Thêm vào đó, vì là một kiểu giao thức giao tiếp nối
tiếp, nên dữ liệu được chuyển đổi dưới dạng các bit và truyền tuần tự dọc theo dây (SDA).
Giống như SPI, I2C là kiểu giao thức đồng bộ, nên đầu ra của các bit được đồng bộ hóa với việc lấy
mẫu các bit dựa theo tín hiệu clock (SCL) và được chia sẻ giữa master và slave. Chú ý rằng, tín hiệu
clock luôn được điều khiển bởi master.

### 2.2. Topology giao thức I2C

Giao thức I2C sẽ bao gồm 3 thành phần chính là: Master, Slave và Bus.
Đối với thiết bị Master, đây là thiết bị chính điều khiển giao tiếp trên I2C Bus. Nó khởi tạo liên lạc
tới các thiết bị Slave, gửi câu lệnh và dữ liệu tới chúng, và nhận dữ liệu phản hồi từ các slave đó. Giao
thức I2C cho phép sự có mặt của nhiều thiết bị Master trên cùng một bus.
Trong khi đó, thiết bị Slave là thiết bị nhận lệnh từ Master và phản hồi dữ liệu lại cho nó. Cần lưu ý
rằng, mỗi thiết bị Slave đều có một địa chỉ trên I2C bus là duy nhất, và Master có thể liên lạc tới chúng
bằng việc gửi các bản tin thông qua I2C bus.
Cuối cùng I2C Bus dùng để vận chuyển các tín hiệu qua lại giữa Master và Slave. Ta sẽ nói kĩ hơn
về nó trong phần sau.

### 2.3. Nguyên lý truyền tín hiệu trong I2C

Với giao thức I2C, dữ liệu được truyền đi dưới dạng các bản tin. Bản tin được chia thành các frame dữ liệu. Mỗi bản tin có:
- Một frame địa chỉ (bao gồm các bit địa chỉ của slave trên bus I2C)
- Một hoặc nhiều frame dữ liệu (chứa dữ liệu được vận chuyển qua lại)
- 2 trạng thái bắt đầu và kết thúc.

![I2C Communication Protocol](/assets/img/IIC/i2c-frame.webp)

<p style="text-align: center;"><em>Hình 1: Khung truyền tín hiệu I2C</em></p>

#### **Các thành phần trong bản tin:**

Các thành phần có trong bản tin sẽ mang một nhiệm vụ thông báo tình trạng khác nhau của khung
truyền, cụ thể:
- Start Condition: Đường SDA kéo điện áp mức cao xuống điện áp mức thấp (1 → 0) trước khi
đường SCL làm điều đó.
- Stop Condition: Đường SDA chuyển từ mức điện áp thấp lên mức điện áp cao sau khi đường
SCL đã thực hiện điều này.
- Frame địa chỉ: Nó bao gồm 7 hoặc 10 bit tuần tự ghép lại thành 1 địa chỉ duy nhất của slave,
được dùng để xác định slave khi master giao tiếp với nó.
- Frame dữ liệu: bao gồm 8 bit dữ liệu, được master gửi đi trong trường hợp muốn ghi vào thanh
ghi hoặc do slave phản hồi lại khi cần đọc dữ liệu từ slave.
- Read/Write bit: Là một bit riêng lẻ, bất cứ khi nào master muốn gửi dữ liệu tới slave hoặc đọc
dữ liệu từ thanh ghi của slave, 0 là write trong khi 1 sẽ là read.
- ACK/NACK bit: Mỗi frame trong bản tin được theo sau là một bit ACK/NACK dùng để xác
nhận sự thành công của việc gửi tín hiệu. Nếu một địa chỉ frame hoặc dữ liệu được gửi đi
thành công, bit ACK được trả về cho master từ slave. Cụ thể ACK sẽ bằng 0 nếu frame được
gửi thành công, và được xác nhận bởi slave.

### 2.4. Kết nối phần cứng trong hệ thống I2C

Để có thể truyền các tín hiệu thông qua giao thức I2C, các thành phần phần cứng đóng vai trò cực kỳ quan trọng. Trong hệ thống I2C của ZCU102 bao gồm 4 thành phần phần cứng chính:

#### **2.4.1. CPU trong hệ thống I2C**

Trong hệ thống I2C, CPU đóng vai trò trung tâm trong việc điều khiển giao tiếp giữa các thiết bị. Nó
chịu trách nhiệm bắt đầu giao tiếp với thiết bị Slave và gửi cũng như nhận dữ liệu qua bus I2C.
CPU giao tiếp với I2C controller, chịu trách nhiệm quản lý các Bus I2C và giao tiếp giữa các thiết bị.
CPU gửi các lệnh đến I2C controller để bắt đầu truyền và các controller sẽ tạo các tín hiệu đồng hồ đồng
bộ hóa việc truyền dữ liệu giữa các thiết bị. CPU cũng nhận được dữ liệu từ I2C controller và xử lý nó
khi cần thiết.
CPU có thể hoạt động như một thiết bị Master hoặc Slave trong hệ thống I2C. Khi là Master, nó bắt
đầu liên lạc với các thiết bị Slave và điều khiển luồng dữ liệu trên các bus. Trong khi nếu là thiết bị
Slave, nó sẽ trả lời các yêu cầu từ thiết bị chính và gửi và nhận dữ liệu theo chỉ dẫn.

![I2C Communication Protocol](/assets/img/IIC/cpu.png)

<p style="text-align: center;"><em>Hình 2: CPU (Control Processor Unit)</em></p>

Nhìn chung, vai trò của CPU trong hệ thống I2C là điều khiển giao tiếp giữa các thiết bị và xử lý dữ
liệu được truyền qua bus.


#### **2.4.2. PS/PL I2C Controller**

CPU không thể giao tiếp trực tiếp với các thiết bị thông qua 2 tín hiệu SCL và SDA, thay vào đó nó
cần một thành phần trung gian để chuyển đổi tín hiệu từ CPU gửi đến cho các thiết bị cũng như tín hiệu
phản hồi lại của thiết bị tới CPU.Thành phần đó là I2C controller. Cần lưu ý thêm, mỗi một controller
sẽ có một driver phù hợp, điều này phục vụ mục đích cung cấp các chức năng riêng cho từng controller.
Cụ thể, mỗi một driver tương ứng với I2C controller sẽ được đăng ký với I2C_core thông qua hàm
smbus_xfer, từ đó thông báo cho CPU biết được, với controller nào có thể sử dụng được chức năng nào,
điều này giúp CPU phân biệt được các I2C controller với nhau. Như vậy, mỗi một I2C controller sẽ
được điều khiển bởi CPU thông qua một driver với các chức năng riêng biệt.

![I2C Communication Protocol](/assets/img/IIC/i2c-controller.png)

<p style="text-align: center;"><em>Hình 3: Mô hình I2C controller tương ứng với driver</em></p>

Cụ thể hơn, I2C controller là một module phần cứng có thể được tích hợp cả bên trong hoặc ngoài
CPU, đối với Zynq nói chung hay trên bo mạch ZCU102 nói riêng, I2C controller sẽ được chia thành 2
loại:
- PS I2C controller: Được đặt bên trong CPU dưới dạng module và được cung cấp các chức năng
thông qua Cadence I2C driver. Lưu ý, có hai PS I2C controller được đặt bên trong CPU là I2C0
và I2C1, tuy nhiên chúng vẫn sử dụng chung một driver.
- PL I2C controller: Được đặt bên trong phần Programmable Logic (ngoài CPU), có tên gọi là Axi
I2C và được cung cấp chức năng bởi Xilinx I2C driver (axi I2C driver). Chú ý rằng, PL I2C
controller cần phải được kích hoạt driver tương ứng của nó để có thể sử dụng. Đối với petalinux,
có thể sử dụng câu lệnh petalinux-config -c kernel để điều chỉnh kernel, sau đó cho phép Xilinx
I2C driver hoạt động. Từ đó có thể sử dụng được PL I2C controller.

![I2C Communication Protocol](/assets/img/IIC/module-i2c.png)

<p style="text-align: center;"><em>Hình 4: Module I2C controller trong ZynqMP</em></p>

#### **2.4.3. I2C Bus**

Bus I2C là kết nối vật lý giữa các thiết bị. Nó bao gồm hai dây: dòng clock (SCL) và dòng dữ liệu
(SDA). Các thiết bị được kết nối với bus trong cấu hình nhiều drop, có nghĩa là nhiều thiết bị có thể
được kết nối với cùng một bus.

Các thiết bị I2C là các thiết bị điện tử được kết nối với bus I2C và có thể giao tiếp với nhau bằng giao thức I2C.

Như đã nói trong phần trước, giao thức I2C là giao thức nối tiếp sử dụng 2 dây cho phép nhiều thiết
bị được kết nối trên cùng một bus. Nó thường được sử dụng để giao tiếp giữa các cảm biến, màn hình
và bộ vi điều khiển trong nhiều ứng dụng, bao gồm điện tử tiêu dùng, hệ thống ô tô, ...
Bus I2C sử dụng tín hiệu clock nối tiếp (SCL) và tín hiệu dữ liệu nối tiếp (SDA) để truyền dữ liệu
giữa các thiết bị. Tín hiệu SCL được sử dụng để đồng bộ hóa việc truyền dữ liệu, trong khi tín hiệu SDA
được sử dụng để truyền dữ liệu thực tế.
Mỗi thiết bị trên bus I2C có địa chỉ 7 bit hoặc 10 bit, nó được sử dụng bởi Master để liên lạc với các
Slave cụ thể. Master có thể gửi các lệnh và dữ liệu đến các Slave và các Slave có thể phản hồi các lệnh
và truyền dữ liệu cho nó.
I2C có một số lợi thế so với các giao thức truyền thông khác, bao gồm chi phí thấp, hệ thống dây đơn
giản và hỗ trợ cho nhiều thiết bị trên cùng một Bus.


Hai tín hiệu SCL và SDA hoạt động ở chế độ Open drain. Khi hoạt động ở chế độ Open drain, mỗi
một slave chỉ có thể kéo điện áp từ mức High (1) xuống mức Low (0) mà không đẩy được từ Low (0) lên
High (1).
Khác với chế độ Push pull, đầu ra mức logic luôn nằm trong hai lựa chọn là kéo xuống Low (0) hoặc
đẩy lên High (1), thì Open drain chỉ có thể kéo tín hiệu xuống Low (0). Việc này được thực hiện nhằm
mục đích tránh xung đột giữa các slave với nhau hoặc giữa slave với master, khi có một thiết bị kéo lên
mức High (1) và một thiết bị kéo xuống mức Low (0) dẫn đến chập mạch.
Để làm được điều này, mỗi một slave và master sẽ được kết nối với 1 transistor NMOS, transistor
này có nhiệm vụ kéo điện áp xuống thấp khi bit logic = 0 (bật) và thả nổi khi bit logic bằng 1 (tắt). Để
dễ hình dung hơn về thiết kế của transistor này, chúng ta cùng so sánh giữa 2 thiết kế của chế độ Push
pull và Open drain.

![I2C Communication Protocol](/assets/img/IIC/open-drain.png)

<p style="text-align: center;"><em>Hình 5: Cấu trúc push-pull và open-drain</em></p>

Trong khi push pull có cả 2 transistor NMOS và PMOS để thực hiện việc kéo vào đẩy thì Open drain
chỉ có một transistor NMOS để phục vụ nhiệm vụ kéo tín hiệu xuống thấp mà không thể đẩy nó lên.
Như vậy, việc kéo trạng thái của các tín hiệu trở lại mức High(1) sẽ được đảm nhiệm bởi một thành
phần khác. Đó là điện trở Pull up, trong trường hợp không có slave nào khớp với địa chỉ được master
gửi đi hoặc khi đã kết thúc một khung truyền, hai tín hiệu SDA và SCL cần được kéo trở lại trạng thái
High, lúc này nó sẽ cần tới điện trở pull-up.

![I2C Communication Protocol](/assets/img/IIC/resister.png)

<p style="text-align: center;"><em>Hình 6: Điện trở pull-up trên I2C bus</em></p>


#### **2.4.4. I2C Device**

Các thiết bị I2C là các thiết bị điện tử được kết nối với bus I2C và có thể giao tiếp với nhau bằng giao
thức I2C. Các thiết bị này có thể bao gồm các cảm biến, bộ điều khiển và các thiết bị ngoại vi khác cần
trao đổi dữ liệu với bộ xử lý trung tâm (CPU) hoặc các thiết bị khác trên bus.
Các thành phần phần cứng của thiết bị I2C thường bao gồm:
- Địa chỉ I2C: Mỗi thiết bị I2C có một địa chỉ duy nhất, được sử dụng để xác định thiết bị bus.
Địa chỉ thường là số 7 bit hoặc 10 bit và nó được đặt bởi nhà sản xuất thiết bị.
- Chân hoặc đầu nối: Các thiết bị I2C thường được kết nối với bus thông qua một bộ chân hoặc
đầu nối trên thiết bị. Các chân này được sử dụng để kết nối thiết bị với các đường SCL và SDA
của bus.
- Các thành phần khác: Tùy thuộc vào loại thiết bị I2C cụ thể, có thể có các thành phần phần cứng
khác. Ví dụ, một cảm biến có thể bao gồm các cảm biến hoặc các phần tử phát hiện khác và bộ
điều khiển có thể bao gồm các nút hoặc các phần tử đầu vào khác.

---

## 3. I2C trong ngành ô tô

### 3.1. Tổng quan

Trong ngành ô tô, I2C được sử dụng rộng rãi trong các đơn vị điều khiển điện tử (ECUs) để kết nối cảm biến, màn hình, và bộ điều khiển. Tuy nhiên, do môi trường ô tô có nhiễu điện từ cao (từ động cơ, máy phát), I2C phù hợp cho các kết nối ngắn (dưới vài mét) trong nội bộ ECU, trong khi CAN hoặc RS-485 được ưu tiên cho mạng lưới toàn xe.

### 3.2. Ứng dụng cụ thể

- **Cảm biến và bộ nhớ**: I2C kết nối cảm biến nhiệt độ (MLX90614) hoặc EEPROM (AT24HC04B) để lưu trữ dữ liệu cấu hình trong ECUs.
- **Hệ thống ADAS**: I2C truyền dữ liệu từ cảm biến radar/camera đến bộ xử lý, hỗ trợ các tính năng như phát hiện va chạm.
- **Hệ thống giải trí**: I2C kết nối màn hình cảm ứng với bộ điều khiển để hiển thị giao diện người dùng.

### 3.3. Lợi ích và hạn chế

- **Lợi ích**:
  - Chi phí thấp, giảm số lượng dây.
  - Hỗ trợ đa thiết bị trên cùng một bus.
  - Dễ triển khai trong không gian hạn chế.
- **Hạn chế**:
  - Không phù hợp cho truyền tín hiệu dài do nhạy với nhiễu.
  - Quản lý địa chỉ phức tạp trong hệ thống nhiều thiết bị.
  - Tốc độ giới hạn (thường 100-400 kbps) trong môi trường ô tô.

### 3.4. Thông số kỹ thuật

| **Chế độ I2C**       | **Tốc độ tối đa (kbps)** |
|-----------------------|--------------------------|
| Standard-mode         | 100                      |
| Fast-mode            | 400                      |
| Fast-mode Plus       | 1000                     |
| High-speed mode      | 3400                     |
| Ultra-Fast mode      | 5000                     |

Thiết bị ô tô như AT24HC04B đáp ứng dải nhiệt -40°C đến 125°C, phù hợp với điều kiện khắc nghiệt.

### 3.5. So sánh với các giao thức khác

| **Tiêu chí**          | **I2C**                     | **CAN**                     | **RS-485**                  |
|-----------------------|-----------------------------|-----------------------------|-----------------------------|
| **Khoảng cách**       | Ngắn (ít mét)               | Dài (lên đến 1 km)          | Dài (lên đến 1.2 km)        |
| **Tốc độ**            | 100 kbps - 5 Mbps           | 125 kbps - 1 Mbps           | 10 Mbps (tùy cấu hình)      |
| **Ứng dụng**          | Giao tiếp nội bộ ECU        | Mạng lưới nội bộ xe         | Kết nối dài, công nghiệp    |
| **Kháng nhiễu**       | Thấp                        | Cao                        | Cao                        |

### 3.6. Xu hướng năm 2025

Trong bối cảnh năm 2025, ngành ô tô tập trung vào xe điện, xe tự lái (Level 3/4), và xe kết nối. I2C tiếp tục đóng vai trò quan trọng trong giao tiếp nội bộ ECU, hỗ trợ các công nghệ như ADAS và hệ thống giải trí, dù không có cải tiến lớn về giao thức này.

---

## 4. Kết luận

I2C là giao thức quan trọng trong hệ thống nhúng và ngành ô tô, đặc biệt cho các kết nối ngắn trong ECUs. Trên bo mạch ZCU102, I2C được triển khai hiệu quả với PS/PL controller và i2c-tools. Trong ô tô, I2C hỗ trợ cảm biến, bộ nhớ, và giao diện, nhưng bị giới hạn bởi nhiễu và khoảng cách. Các giao thức như CAN hoặc RS-485 phù hợp hơn cho mạng lưới toàn xe. I2C vẫn là lựa chọn tối ưu cho các ứng dụng chi phí thấp và không gian hạn chế.

---

## 5. Tài liệu tham khảo

- [Phân tích giao thức I2C trong Embedded Systems](/posts/i2c-protocol/)
- [The I2C bus in the automotive sector](https://evision-webshop.eu/blog/the-i2c-bus-in-the-automotive-sector)
- [A Basic Guide to I²C (Texas Instruments)](https://www.ti.com/lit/an/sbaa565/sbaa565.pdf)
- [Automotive Technology Developments in 2025](https://www.simplycarbuyers.com/blog/automotive-technology-developments-in-2025/)