# CHƯƠNG 10: TRUYỀN THÔNG KHÔNG DÂY BLUETOOTH LOW ENERGY (BLE) VỚI ĐIỆN THOẠI TRÊN STM32F407
*(Biên soạn dựa trên Chapter 10: Bluetooth Low Energy Communication with a Smartphone - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 10.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã thành thạo giao tiếp nối tiếp có dây (UART) ở Chương 2; chương này mở rộng sang **Truyền thông vô tuyến tầm ngắn tiết kiệm năng lượng Bluetooth Low Energy (BLE)** để kết nối trực tiếp hệ thống nhúng STM32F407 với ứng dụng di động trên Smartphone.
- **Mục tiêu chi tiết:**
  1. **Lý thuyết Truyền thông Vô tuyến BLE:** Nắm vững dải tần số $2.4\text{ GHz}$ ISM, 40 kênh tần số vô tuyến (3 kênh quảng bá Advertising và 37 kênh truyền dữ liệu Data), kiến trúc phân tầng Controller - Host, vai trò kết nối **GAP** (Central vs Peripheral, Broadcasting vs Scanning) và kiến trúc dữ liệu thuộc tính **GATT** (GATT Server/Client, Services, Characteristics, Descriptors, UUID 128-bit).
  2. **Tối ưu hóa Năng lượng Thiết bị Chạy Pin:** Hiểu rõ cơ chế chu kỳ hoạt động ngắt quãng (Duty Cycling) cho phép thiết bị ngủ sâu với dòng tiêu thụ $< 1\mu\text{A}$ và đánh thức phát sóng trong vài mili-giây.
  3. **Thiết kế Mạch Ngoại vi & Cấu hình AT:** Ghép nối module BLE (HC-08 / HM-10) với STM32F407 qua khối `USART3` (`PB10` TX, `PB11` RX), theo dõi trạng thái kết nối phần cứng qua chân `STATE` (`PE6`), cấu hình module bằng tập lệnh AT Commands.
  4. **Lập trình Điều khiển Không dây (Examples 10.1 $\div$ 10.4):**
     - Nhận và giải mã gói lệnh điều khiển đóng/mở/dừng cổng trượt tự động từ Smartphone.
     - Đóng gói bản tin Telemetry định kỳ gửi nhiệt độ và trạng thái báo động lên ứng dụng di động.
  5. **Thực nghiệm Ứng dụng Di động:** Vận hành thực tế trên ứng dụng *Serial Bluetooth Terminal* hoặc *nRF Connect* trên Android / iOS.

---

## 10.2. CƠ SỞ LÝ THUYẾT CHUYÊN SÂU

### 10.2.1. Kiến Trúc Phân Tầng Giao Thức Bluetooth Low Energy (BLE)
Bluetooth Low Energy (từ chuẩn Bluetooth 4.0 trở lên) là một công nghệ hoàn toàn độc lập với Bluetooth cổ điển (Classic Bluetooth), được thiết kế tối ưu hóa riêng cho các thiết bị IoT năng lượng thấp:

```
+-------------------------------------------------------------------------------+
|                       TẦNG ỨNG DỤNG (APPLICATION LAYER)                       |
|   Ứng dụng di động Smartphone App  <---->  Chương trình C trên STM32F407      |
+-------------------------------------------------------------------------------+
|                      TẦNG HOST (GIAO THỨC CẤP CAO)                            |
|  - GAP  (Generic Access Profile): Quảng bá, Dò tìm, Thiết lập kết nối          |
|  - GATT (Generic Attribute Profile): Cơ sở dữ liệu Dịch vụ & Đặc tính          |
|  - ATT  (Attribute Protocol): Bản tin Read/Write 16/128-bit UUID              |
|  - SM   (Security Manager): Mã hóa AES-128, Ghép đôi (Pairing/Bonding)        |
|  - L2CAP: Phân đoạn và gom gói dữ liệu logic                                  |
+-------------------------------------------------------------------------------+
|                     TẦNG CONTROLLER (PHẦN CỨNG VÔ TUYẾN)                      |
|  - Link Layer (LL): Quản lý máy trạng thái vô tuyến, ACK, Retransmission      |
|  - Physical Layer (PHY): Dải tần 2.4GHz ISM, Điều chế tần số số GFSK          |
+-------------------------------------------------------------------------------+
```

#### 1. Phân bổ kênh tần số vô tuyến:
- BLE hoạt động trên băng tần $2.4\text{ GHz}$ ISM ($2400 \div 2483.5\text{ MHz}$), chia thành **40 kênh tần số** cách nhau $2\text{ MHz}$:
  - **3 kênh quảng bá (Advertising Channels - Kênh 37, 38, 39):** Nằm ở các khoảng tần số đặc biệt ($2402\text{ MHz}, 2426\text{ MHz}, 2480\text{ MHz}$) để tránh nhiễu với các kênh phổ biến của sóng Wi-Fi (Kênh 1, 6, 11).
  - **37 kênh truyền dữ liệu (Data Channels - Kênh 0 $\div$ 36):** Dùng để trao đổi dữ liệu hai chiều khi đã thiết lập kết nối, sử dụng thuật toán nhảy tần thích ứng (Adaptive Frequency Hopping - AFH) để chống nghẽn sóng.

#### 2. Mô hình kết nối GAP (Generic Access Profile):
- **Thiết bị Ngoại vi (Peripheral - Module BLE / STM32F407):** Ở trạng thái chờ, module phát các gói tin quảng bá (Advertising Packets) chu kỳ đều đặn (ví dụ $100\text{ms}$). Gói tin này chứa tên thiết bị (`SmartHome_F407`), công suất phát (TX Power) và danh sách dịch vụ hỗ trợ.
- **Thiết bị Trung tâm (Central - Smartphone / Tablet):** Bật bộ thu vô tuyến quét liên tục (Scanning) để tìm kiếm gói tin quảng bá. Khi người dùng bấm nút kết nối, Central gửi gói tin yêu cầu kết nối (`CONNECT_REQ`), hai bên đồng bộ thời gian và chuyển sang kênh dữ liệu riêng.

#### 3. Kiến trúc dữ liệu phân cấp GATT (Generic Attribute Profile):
Dữ liệu trong BLE không truyền tự do dưới dạng chuỗi thô mà được cấu trúc thành một cơ sở dữ liệu phân cấp hình cây:

```
Profile (Hồ sơ tổng thể: Ví dụ "Smart Home System Profile")
   |
   +---> Service 1 (Dịch vụ Điều khiển Cổng - UUID: 0xFFE0)
   |        |
   |        +---> Characteristic 1 (Lệnh Đóng/Mở - UUID: 0xFFE1)
   |                 - Thuộc tính (Properties): Write, Read
   |                 - Giá trị (Value): "OPEN" / "CLOSE" / "STOP"
   |
   +---> Service 2 (Dịch vụ Giám sát Môi trường - UUID: 0x181A - Environmental Sensing)
            |
            +---> Characteristic 2 (Nhiệt độ phòng - UUID: 0x2A6E - Temperature)
                     - Thuộc tính (Properties): Read, Notify (Tự động đẩy khi đổi)
                     - Giá trị (Value): 29.5 độ C
```

---

### 10.2.2. Cơ Chế Tiết Kiệm Năng Lượng Đột Phá Của BLE
So sánh dòng tiêu thụ giữa Bluetooth cổ điển và BLE:
- **Bluetooth cổ điển (Classic Bluetooth 2.1 EDR):** Thiết bị phải duy trì kết nối liên tục, dòng tiêu thụ luôn ở mức $20 \div 40\text{mA}$, một viên pin nhỏ chỉ duy trì được vài giờ.
- **Bluetooth Low Energy (BLE):**
  - Thời gian phát sóng một gói tin chỉ mất từ $0.5\text{ ms} \div 3\text{ ms}$ với dòng đỉnh khoảng $10 \div 15\text{mA}$.
  - Giữa các lần phát sóng, phần cứng vi mạch tự động rơi vào chế độ ngủ sâu (Deep Sleep / Standby) với dòng tiêu thụ chỉ từ **$0.4 \div 1.5\mu\text{A}$**!
  - Hệ số công tác (Duty Cycle):
    $$\text{Duty Cycle} = \frac{t_{active}}{T_{interval}} = \frac{2\text{ ms}}{1000\text{ ms}} = 0.2\%$$
  - $\implies$ **Dòng tiêu thụ trung bình:**
    $$I_{avg} = (15\text{mA} \times 0.002) + (1\mu\text{A} \times 0.998) \approx 31\mu\text{A}.$$
    Với một viên pin cúc áo CR2032 dung lượng $220\text{mAh}$, thiết bị có thể chạy liên tục gần **1 năm** mà không cần thay pin!

---

### 10.2.3. Tập Lệnh Cấu Hình AT Commands Cho Module BLE (HC-08 / HM-10)
Module BLE HC-08/HM-10 tích hợp sẵn vi điều khiển độc lập chạy firmware truyền dẫn trong suốt (UART Transparent Mode). Khi chưa kết nối Bluetooth, module hoạt động ở chế độ lệnh **AT Command Mode** qua cổng nối tiếp:

| Cú pháp Lệnh AT | Chức năng chi tiết | Phản hồi từ Module BLE |
| :--- | :--- | :--- |
| `AT` | Kiểm tra kết nối nối tiếp | `OK` |
| `AT+NAME=<Tên>` | Đặt tên hiển thị quảng bá (Tối đa 12 ký tự) | `OK+NAME=<Tên>` |
| `AT+BAUD=<Mã>` | Cài đặt tốc độ Baudrate UART (Ví dụ `AT+BAUD=4` cho 9600 bps) | `OK+Get:9600` |
| `AT+ROLE=<0/1>` | Cấu hình vai trò: `0` = Peripheral (Slave), `1` = Central (Master) | `OK+ROLE:0` |
| `AT+ADDR?` | Đọc địa chỉ vật lý MAC Address 48-bit của module | `OK+ADDR:123456ABCDEF` |
| `AT+RX` | Đọc thông số cấu hình lưu trong Flash của module | Bảng thông số |

---

## 10.3. SƠ ĐỒ NGUYÊN LÝ & SƠ ĐỒ ĐẤU NỐI CHÂN CHI TIẾT

### 10.3.1. Sơ đồ mạch nguyên lý điện tử (Schematics)

```
=============================================================================================
SƠ ĐỒ GHÉP NỐI MODULE BLE HC-08 / HM-10 VÀO STM32F407 (USART3)
=============================================================================================
STM32F407                                                    Module BLE HC-08
+------------------+                                        +---------------------+
|             3.3V |----------------------------------------| VCC (Nguồn cấp 3.3V)|
|                  |                                        |                     |
|              GND |----------------------------------------| GND (Mass chung)    |
|                  |                                        |                     |
| PB10 (USART3_TX) |----------------------------------------| RXD (Nhận dữ liệu)  |
|                  |                                        |                     |
| PB11 (USART3_RX) |<---------------------------------------| TXD (Phát dữ liệu)  |
|                  |                                        |                     |
|  PE6 (GPIO_In)   |<---------------------------------------| STATE / LED         |
|                  |     (Mức 0: Đang chờ kết nối (Chớp LED)|                     |
+------------------+      Mức 1: Điện thoại ĐÃ KẾT NỐI)     +---------------------+
```

---

### 10.3.2. Bảng Phân Bổ Chân Toàn Diện Trên STM32F407 (Master Pinout Table)

| Tên Chân STM32 | Tên Tín Hiệu | Chức Năng Phần Cứng / Alternate Function | Cấu Hình Trên STM32CubeMX | Kết Nối Chân Module BLE | Mức Điện Áp & Lưu Ý Kỹ Thuật |
| :---: | :---: | :---: | :---: | :--- | :--- |
| `PB10` | `BLE_TX` | `USART3_TX` (AF7) | Alternate Function Push-Pull, High Speed | Nối vào chân `RXD` của module BLE | Mức logic $3.3\text{V}$ |
| `PB11` | `BLE_RX` | `USART3_RX` (AF7) | Alternate Function Push-Pull, Bật **Ngắt USART3** | Nối vào chân `TXD` của module BLE | Mức logic $3.3\text{V}$ |
| `PE6` | `BLE_STATE`| GPIO_Input | Input Mode, Pull-down | Nối vào chân `STATE` của module BLE | Mức 0 = Chờ kết nối; Mức 1 = Đã kết nối |
| `PC8` | `RELAY_FWD`| GPIO_Output | Output Push-Pull | Lệnh mở cổng tự động qua Relay 1 | Tải động cơ cổng |
| `PC9` | `RELAY_REV`| GPIO_Output | Output Push-Pull | Lệnh đóng cổng tự động qua Relay 2 | Tải động cơ cổng |
| `3V3` | `VCC` | Nguồn cấp $3.3\text{V}$ | Nguồn nuôi hệ thống | Cấp cho chân `VCC` module BLE | Dòng tiêu thụ phát sóng $< 15\text{mA}$ |
| `GND` | `GND` | Mass hệ thống | Mass chung | Nối chung mass STM32 và module | Bắt buộc chung mass |

---

## 10.4. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 10.1 ĐẾN 10.4 TRÊN STM32F407

### 10.4.1. Cấu hình STM32CubeMX Cho USART3
1. **Khối USART3:**
   - Mode: **Asynchronous**.
   - Baud Rate: **9600 Bits/s** (hoặc 115200 theo cấu hình module).
   - Word Length: **8 Bits**, Parity: **None**, Stop Bits: **1**.
   - Vào tab *NVIC Settings* của USART3: **Tích chọn USART3 global interrupt** với Preemption Priority = **1**.
2. **Chân PE6:** Cấu hình GPIO_Input, Pull-down.

---

### 10.4.2. Mã Nguồn Hoàn Chỉnh Của Chương

#### 1. Khởi Tạo Ngắt Nhận Ký Tự Không Chặn (Non-blocking UART RX)
```c
#include "main.h"
#include <string.h>
#include <stdio.h>
#include <stdbool.h>

extern UART_HandleTypeDef huart3;

#define BLE_BUF_SIZE 64
static char bleRxBuffer[BLE_BUF_SIZE];
static uint8_t bleRxIndex = 0;
static uint8_t incomingByte;
static volatile bool hasNewCommand = false;

/* Bắt đầu kích hoạt ngắt nhận từng byte */
void BLE_StartReceiveIT(void)
{
    HAL_UART_Receive_IT(&huart3, &incomingByte, 1);
}

/* Callback ngắt nhận của STM32 HAL */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART3)
    {
        /* Ký tự kết thúc lệnh (\r hoặc \n) */
        if (incomingByte == '\r' || incomingByte == '\n')
        {
            if (bleRxIndex > 0)
            {
                bleRxBuffer[bleRxIndex] = '\0'; /* Chốt chuỗi */
                hasNewCommand = true;
            }
        }
        else
        {
            if (bleRxIndex < (BLE_BUF_SIZE - 1))
            {
                bleRxBuffer[bleRxIndex++] = (char)incomingByte;
            }
        }
        /* Tiếp tục nhận byte tiếp theo */
        HAL_UART_Receive_IT(&huart3, &incomingByte, 1);
    }
}

/* Gửi chuỗi phản hồi không dây lên điện thoại */
void BLE_SendString(const char *msg)
{
    HAL_UART_Transmit(&huart3, (uint8_t *)msg, strlen(msg), 100);
}
```

---

#### 2. Máy Phân Phối Lệnh Điều Khiển Đóng Mở Cổng (Example 10.1)
```c
void BLE_CommandDispatcher_Task(void)
{
    if (!hasNewCommand) return;
    hasNewCommand = false;

    printf("[BLE RECEIVED]: %s\r\n", bleRxBuffer);

    /* Phân loại và thực thi lệnh */
    if (strcmp(bleRxBuffer, "OPEN") == 0 || strcmp(bleRxBuffer, "O") == 0)
    {
        /* Kích hoạt Relay mở cổng */
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_SET);
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_RESET);
        BLE_SendString("ACK: DANG MO CONG...\r\n");
    }
    else if (strcmp(bleRxBuffer, "CLOSE") == 0 || strcmp(bleRxBuffer, "C") == 0)
    {
        /* Kích hoạt Relay đóng cổng */
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_SET);
        BLE_SendString("ACK: DANG DONG CONG...\r\n");
    }
    else if (strcmp(bleRxBuffer, "STOP") == 0 || strcmp(bleRxBuffer, "S") == 0)
    {
        /* Dừng phanh động cơ tức thì */
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_RESET);
        BLE_SendString("ACK: DA DUNG CONG TUC THOI!\r\n");
    }
    else if (strcmp(bleRxBuffer, "GET_STATUS") == 0)
    {
        /* Gửi thông tin trạng thái đầy đủ */
        char telemetryBuf[64];
        float tempVal = 28.6f;
        snprintf(telemetryBuf, sizeof(telemetryBuf), "STATUS: TEMP=%.1fC,ALARM=SAFE\r\n", tempVal);
        BLE_SendString(telemetryBuf);
    }
    else
    {
        BLE_SendString("ERR: Lenh khong hop le! Dung OPEN, CLOSE, STOP, GET_STATUS\r\n");
    }

    bleRxIndex = 0; /* Xóa bộ đệm */
}
```

---

#### 3. Báo Cáo Định Kỳ Trạng Thái Môi Trường Lên Smartphone (Example 10.2)
```c
void BLE_PeriodicTelemetry_Task(void)
{
    static uint32_t lastReport = 0;
    if (HAL_GetTick() - lastReport < 2000) return;
    lastReport = HAL_GetTick();

    /* Kiểm tra chân PE6 xem điện thoại đã kết nối Bluetooth hay chưa */
    if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_6) == GPIO_PIN_SET)
    {
        char reportPacket[64];
        float currentTemp = 29.2f; /* Đọc từ DS18B20 */
        bool gasAlert = (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) == GPIO_PIN_SET);

        snprintf(reportPacket, sizeof(reportPacket),
                 "[TELEMETRY] T:%.1f C | GAS:%s\r\n",
                 currentTemp, gasAlert ? "NGUY HIEM" : "AN TOAN");

        BLE_SendString(reportPacket);
    }
}
```

---

## 10.5. "UNDER THE HOOD" & CASE STUDY (THEO GIÁO TRÌNH ARM)

### 10.5.1. Under the Hood (10.3.1): Kiến Trúc Bảo Mật & Ghép Đôi (BLE Security & Bonding)
- **Tầng Security Manager (SM):** Hỗ trợ mã hóa dữ liệu vô tuyến bằng thuật toán chuẩn quân đội **AES-128 bit**.
- **Cơ chế ghép đôi (Pairing & Bonding):**
  1. *Just Works:* Không cần nhập mã, dùng cho thiết bị không có màn hình/bàn phím.
  2. *Passkey Entry:* Vi điều khiển sinh mã số 6 chữ số ngẫu nhiên hiển thị trên màn hình LCD, người dùng phải nhập đúng 6 số này trên điện thoại để xác thực.
  3. *Lưu trữ khóa liên kết (Bonding):* Sau khi ghép đôi lần đầu, hai bên lưu khóa giải mã Long Term Key (LTK) vào bộ nhớ Flash. Ở các lần sau, thiết bị tự động nhận diện và kết nối được mã hóa mà không cần hỏi lại mã xác nhận.

### 10.5.2. Case Study (10.4.1): Thiết Bị Cầu Nối Không Dây Công Nghiệp (Industrial Wireless Bolt)
- **Bối cảnh ứng dụng:** Trong các nhà máy thép và hóa chất, các tủ điện điều khiển chứa điện áp cao thế $380\text{V} \div 10\text{kV}$ và môi trường kín chống bụi chống cháy nổ IP67.
- **Giải pháp thiết bị Bolt:**
  - Một module BLE công nghiệp được đục lỗ lắp xuyên qua nóc tủ điện kim loại.
  - Bên trong tủ: Module kết nối với vi điều khiển qua bus UART/CAN cách ly quang.
  - Bên ngoài tủ: Phần ăng-ten vòm bán cầu phát sóng BLE ra ngoài.
  - **Lợi ích an toàn vượt trội:** Kỹ sư bảo trì chỉ cần cầm máy tính bảng đứng cách xa $10\text{ mét}$ là có thể đọc toàn bộ lịch sử nhiệt độ cuộn dây, chẩn đoán lỗi biến tần mà **không cần mở cửa tủ điện**, loại bỏ hoàn toàn nguy cơ phóng hồ quang điện (Arc Flash) gây tử vong.

---

## 10.6. CÂU HỎI ÔN TẬP VÀ BÀI TẬP THIẾT KẾ CHƯƠNG 10

1. **Câu hỏi 1 (Lý thuyết vô tuyến BLE):** *Tại sao trong 40 kênh của BLE, 3 kênh quảng bá (Advertising) lại được đặt ở các kênh 37 (2402MHz), 38 (2426MHz) và 39 (2480MHz)? Ý nghĩa của việc sắp xếp này đối với sự cùng tồn tại của sóng Wi-Fi là gì?*
2. **Câu hỏi 2 (Kiến trúc GATT):** *Giải thích sự khác biệt cơ bản giữa thuộc tính đặc tính `Notify` và `Indicate` trong giao thức GATT. Khi nào ta nên chọn `Notify` và khi nào bắt buộc dùng `Indicate`?*
3. **Câu hỏi 3 (Lập trình ngắt UART):** *Tại sao khi nhận dữ liệu từ module BLE, ta bắt buộc phải sử dụng hàm ngắt `HAL_UART_Receive_IT` thay vì hàm chờ thăm dò `HAL_UART_Receive`?*
4. **Bài tập thiết kế hệ thống:** *Thiết kế một thiết bị đeo tay theo dõi nhịp tim bệnh nhân chạy bằng pin CR2032 sử dụng vi điều khiển ARM và module BLE. Hãy tính toán năng lượng tiêu thụ trung bình và thời gian hoạt động của pin nếu thiết bị ngủ sâu $99\%$ thời gian và thức dậy phát gói tin đo nhịp tim mỗi 1 giây một lần.*
