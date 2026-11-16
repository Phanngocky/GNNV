# CHƯƠNG 2: CƠ SỞ TRUYỀN THÔNG NỐI TIẾP VÀ GIAO TIẾP MÁY TÍNH (UART) TRÊN STM32F407
*(Biên soạn dựa trên Chapter 2: Fundamentals of Serial Communication - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 2.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã hiểu nguyên lý truyền dữ liệu nối tiếp cơ bản từ học phần trước; chương này tập trung vào làm chủ giao thức phần cứng trên STM32F407, kỹ thuật phân tách hàm chức năng (Modularization into functions), quản lý bộ nhớ mảng và giao tiếp máy tính thời gian thực.
- **Mục tiêu chi tiết:**
  1. **Lý thuyết Truyền thông Nối tiếp Bất đồng bộ (UART/USART):** Nắm vững nguyên lý đồng bộ hóa bằng Start bit, cấu trúc khung truyền NRZ chuẩn 8N1, cơ chế lấy mẫu 16 lần (Oversampling by 16) khử nhiễu đa điểm, và phân tích chi tiết 4 cờ lỗi phần cứng (`PE`, `FE`, `NE`, `ORE`) trong thanh ghi trạng thái `USART_SR`.
  2. **So Sánh Các Chuẩn Vật Lý Nối Tiếp:** Phân tích sự khác biệt về điện áp, cự ly và khả năng chống nhiễu giữa TTL 3.3V, RS-232 (chuẩn lưỡng cực $\pm 12\text{V}$) và RS-485 (chuẩn truyền vi sai cân bằng $A-B$ cự ly $1200\text{m}$).
  3. **Tính Toán & Phân Bổ Phần Cứng:**
     - Thiết lập công thức và tự tay tính toán giá trị nạp vào thanh ghi tốc độ Baud `USART_BRR` trên các bus APB1 ($42\text{ MHz}$) và APB2 ($84\text{ MHz}$) ở các tốc độ công nghiệp (9600, 57600, 115200 bps).
     - Ghép nối cổng `USART2` (`PA2` TX, `PA3` RX) với máy tính PC qua mạch chuyển đổi USB-to-UART.
  4. **Lập Trình Điều Khiển Hệ Thống (Examples 2.1 $\div$ 2.5):**
     - Xây dựng hàm `uartTask()` kiểm tra không chặn (Non-blocking polling).
     - Tái định tuyến hàm `printf()` chuẩn C sang cổng UART để in báo cáo có định dạng.
     - Lập trình bộ giải mã lệnh (Command Parser) nhận lệnh từ xa: Giám sát trạng thái còi báo, kiểm tra nguồn gây báo động bằng `switch-case`, tắt báo động từ xa và đổi mật mã hệ thống bằng mảng ký tự.

---

## 2.2. CƠ SỞ LÝ THUYẾT CHUYÊN SÂU

### 2.2.1. Bản Chất Của Truyền Thông Bất Đồng Bộ UART (Universal Asynchronous Receiver/Transmitter)
Trong truyền thông bất đồng bộ (Asynchronous), giữa bên phát (Transmitter) và bên thu (Receiver) **không có đường dây truyền xung nhịp đồng bộ (Clock line) dùng chung**. Thay vào đó, hai bên phải thỏa thuận trước:
1. **Tốc độ truyền (Baud rate):** Số lượng bit truyền đi trong 1 giây (bps). Thời gian của 1 bit là $T_{bit} = \frac{1}{\text{BaudRate}}$.
2. **Khung truyền (Frame format):** Chuẩn phổ biến nhất là **8N1** (1 Start bit, 8 Data bits, No Parity, 1 Stop bit).

```
Idle (3.3V)                                                             Idle (3.3V)
-----+       +-------+-------+-------+-------+-------+-------+-------+-------+-----
     | Start | Bit 0 | Bit 1 | Bit 2 | Bit 3 | Bit 4 | Bit 5 | Bit 6 | Bit 7 | Stop
     |  (0)  | (LSB) |       |       |       |       |       |       | (MSB) | (1) 
     +-------+-------+-------+-------+-------+-------+-------+-------+-------+
     |<----->|
      T_bit = 1/Baud (8.68 us tại 115200 bps)
```

#### Trình tự truyền nhận một byte dữ liệu:
- **Trạng thái nghỉ (Idle State):** Đường truyền luôn duy trì ở mức điện áp cao $V_{DD} = 3.3\text{V}$ (Logic 1).
- **Start Bit (Mức 0):** Bên phát kéo đường truyền từ mức cao xuống thấp trong đúng $1 \times T_{bit}$. Sườn xuống này báo hiệu cho máy thu biết bắt đầu có một ký tự mới truyền tới.
- **Data Bits (8 bits):** Truyền nối tiếp từng bit một, theo quy chuẩn **bit có trọng số nhỏ nhất (LSB - Least Significant Bit) truyền trước**, bit trọng số lớn nhất (MSB) truyền sau cùng.
- **Stop Bit (Mức 1):** Bên phát kéo đường truyền lên mức cao tối thiểu $1 \times T_{bit}$ (hoặc 1.5, 2 bits) để máy thu có thời gian nghỉ nạp dữ liệu vào thanh ghi đệm và sẵn sàng chờ sườn xuống của byte tiếp theo.

---

### 2.2.2. Cơ Chế Lấy Mẫu 16 Lần (Oversampling by 16) & Khử Nhiễu Đa Điểm
Để chống nhiễu trên đường truyền UART, khối phần cứng của STM32F407 không đọc tín hiệu 1 lần đơn lẻ mà chia mỗi khoảng thời gian bit $T_{bit}$ thành **16 khoảng thời gian con (Samples $0 \div 15$)**:

```
      |<------------------------- 1 Chu kỳ T_bit ------------------------->|
      | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15|
Tín hiệu: -----------------------------[ S8 ][ S9 ][ S10 ]---------------------
                                       |<- Lấy mẫu ->|
                                        Đánh giá 2/3
```
- Khi phát hiện sườn xuống của Start bit: Bộ đếm xung mẫu bắt đầu đếm. Máy thu bỏ qua các mẫu đầu ($0 \div 6$) do sườn tín hiệu có thể bị dốc hoặc rung nhiễu.
- Tại điểm giữa của bit (Mẫu thứ 8, 9, 10): Phần cứng lấy mẫu điện áp 3 lần liên tiếp:
  - Nếu cả 3 mẫu (hoặc ít nhất 2 trong 3 mẫu) bằng 1 $\implies$ Giá trị bit được chốt là **1**.
  - Nếu cả 3 mẫu (hoặc ít nhất 2 trong 3 mẫu) bằng 0 $\implies$ Giá trị bit được chốt là **0**.
  - Nếu 3 mẫu có giá trị không đồng nhất ($1-0-1$ hoặc $0-1-0$): Cờ báo nhiễu **Noise Error (`NE`)** trong thanh ghi `USART_SR` sẽ được bật lên để cảnh báo phần mềm.

---

### 2.2.3. Bốn Cờ Báo Lỗi Phần Cứng Trong Thanh Ghi `USART_SR`
Trong các ứng dụng công nghiệp, kỹ sư nhúng phải đọc thanh ghi trạng thái `USARTx->SR` để chẩn đoán sự cố:
1. **Lỗi Quá Tải Bộ Đệm (Overrun Error - `ORE`, bit 3):** Xảy ra khi một byte mới đã nhận xong trên thanh ghi dịch nhưng CPU chưa kịp đọc byte trước đó ra khỏi thanh ghi dữ liệu `USART_DR`. Byte mới sẽ bị ghi đè làm mất mát dữ liệu.
2. **Lỗi Nhiễu Đường Truyền (Noise Error - `NE`, bit 2):** Phát hiện có xung gai nhiễu điện từ xuất hiện ở điểm lấy mẫu giữa bit.
3. **Lỗi Khung Truyền (Framing Error - `FE`, bit 1):** Xảy ra khi tại thời điểm lấy mẫu Stop bit, tín hiệu đo được lại là mức thấp $0\text{V}$ thay vì mức cao $3.3\text{V}$. Nguyên nhân phổ biến nhất là do **lệch tốc độ Baudrate** giữa bên phát và bên thu.
4. **Lỗi Chẵn Lẻ (Parity Error - `PE`, bit 0):** Xảy ra khi bit kiểm tra Parity thu được không khớp với tổng số bit 1 trong dữ liệu đã thỏa thuận.

---

### 2.2.4. So Sánh Chi Tiết 3 Chuẩn Vật Lý: TTL 3.3V vs RS-232 vs RS-485

| Tiêu chí kỹ thuật | Chuẩn UART TTL (STM32) | Chuẩn RS-232 (PC Cũ) | Chuẩn RS-485 (Công Nghiệp) |
| :--- | :---: | :---: | :---: |
| **Kiểu đường truyền** | Đơn cực (Single-ended) | Đơn cực (Single-ended) | **Vi sai cân bằng (Differential $A - B$)** |
| **Mức điện áp Logic 1** | $V_{OH} = 3.3\text{V}$ | $-3\text{V} \div -15\text{V}$ (Mark) | $V_A - V_B < -200\text{mV}$ |
| **Mức điện áp Logic 0** | $V_{OL} = 0\text{V}$ | $+3\text{V} \div +15\text{V}$ (Space) | $V_A - V_B > +200\text{mV}$ |
| **Khoảng cách truyền tối đa**| $< 1 \div 2\text{ mét}$ | $\approx 15\text{ mét}$ | **Lên tới $1200\text{ mét}$** |
| **Tốc độ truyền tối đa** | Lên tới $10.5\text{ Mbps}$ | $20\text{ kbps} \div 115.2\text{ kbps}$ | Lên tới $10\text{ Mbps}$ (ở cự ly ngắn) |
| **Khả năng chống nhiễu** | Kém (dễ bắt nhiễu môi trường) | Trung bình | **Cực cao** (Nhiễu đồng pha bị triệt tiêu) |
| **Cấu hình mạng** | Điểm - Điểm (Point-to-Point) | Điểm - Điểm (Point-to-Point) | **Đa điểm (Multi-drop, tới 32 thiết bị)** |
| **IC Chuyển đổi thông dụng**| Không cần (kết nối trực tiếp) | MAX3232 / MAX232 | SP3485 (3.3V) / MAX485 (5V) |

---

### 2.2.5. Công Thức Tính Toán Thanh Ghi `USART_BRR` (Baud Rate Register)
Thanh ghi `USART_BRR` là một thanh ghi 16-bit:
- 12 bit cao (`DIV_Mantissa` $[15:4]$): Chứa phần nguyên của hệ số chia `USARTDIV`.
- 4 bit thấp (`DIV_Fraction` $[3:0]$): Chứa phần thập phân đã quy đổi sang cơ số 16.

$$\text{USARTDIV} = \frac{f_{PCLK}}{8 \times (2 - \text{OVER8}) \times \text{BaudRate}}$$

Với chế độ mặc định lấy mẫu 16 lần (`OVER8 = 0`):
$$\text{USARTDIV} = \frac{f_{PCLK}}{16 \times \text{BaudRate}}$$

#### Bảng tra cứu giá trị nạp vào `USART_BRR` trên STM32F407:
| Tốc độ Baud | Bus APB1 ($f_{PCLK1} = 42\text{ MHz}$) - `USART2` | Bus APB2 ($f_{PCLK2} = 84\text{ MHz}$) - `USART1, USART6` |
| :---: | :---: | :---: |
| **9600 bps** | $\text{USARTDIV} = \frac{42\times 10^6}{16 \times 9600} \approx 273.4375 \implies \text{BRR} = 273 \times 16 + 7 = \mathbf{0x1117}$ | $\text{USARTDIV} = \frac{84\times 10^6}{16 \times 9600} \approx 546.875 \implies \text{BRR} = 546 \times 16 + 14 = \mathbf{0x222E}$ |
| **57600 bps** | $\text{USARTDIV} = \frac{42\times 10^6}{16 \times 57600} \approx 45.5729 \implies \text{BRR} = 45 \times 16 + 9 = \mathbf{0x02D9}$ | $\text{USARTDIV} = \frac{84\times 10^6}{16 \times 57600} \approx 91.1458 \implies \text{BRR} = 91 \times 16 + 2 = \mathbf{0x05B2}$ |
| **115200 bps**| $\text{USARTDIV} = \frac{42\times 10^6}{16 \times 115200} \approx 22.7865 \implies \text{BRR} = 22 \times 16 + 13 = \mathbf{0x016D}$ | $\text{USARTDIV} = \frac{84\times 10^6}{16 \times 115200} \approx 45.5729 \implies \text{BRR} = 45 \times 16 + 9 = \mathbf{0x02D9}$ |

---

## 2.3. SƠ ĐỒ NGUYÊN LÝ & SƠ ĐỒ ĐẤU NỐI CHÂN CHI TIẾT

### 2.3.1. Sơ đồ mạch nguyên lý điện tử (Schematics)

```
=============================================================================================
1. MẠCH GHÉP NỐI STM32F407 VỚI MÁY TÍNH QUA MODULE USB-TO-UART (CP2102 / CH340)
=============================================================================================
STM32F407                                                    Module USB-to-UART (CP2102)
+------------------+                                        +---------------------+
|  PA2 (USART2_TX) |--------------------------------------->| RXD (Chân nhận PC)  |
|                  |                                        |                     |
|  PA3 (USART2_RX) |<---------------------------------------| TXD (Chân phát PC)  |
|                  |                                        |                     |
|  GND             |----------------------------------------| GND (Mass chung)    |
|                  |                                        |                     |
|  3.3V (KHÔNG NỐI)|   X  CẢNH BÁO: KHÔNG NỐI NGUỒN 5V  X   | 5V / 3.3V (BỎ TRỐNG)|
+------------------+                                        +---------------------+
                                                                      |
                                                                   [Cáp USB]
                                                                      v
                                                                 Máy tính PC


=============================================================================================
2. MẠCH GHÉP NỐI CÔNG NGHIỆP RS-485 VI SAI (SP3485) CỰ LY DÀI 1200 MÉT
=============================================================================================
STM32F407                      IC Thu Phát Vi Sai SP3485 (3.3V)
+------------------+             +-----------------------+
|  PA2 (USART2_TX) |------------>| DI (Driver In)        |
|                  |             |                       |            Đường dây xoắn đôi (Twisted Pair)
|  PA3 (USART2_RX) |<------------| RO (Receiver Out)     |             Dây A (Data +)
|                  |             |                     A |-------------------+--------------------
|  PD4 (DE/RE Ctrl)|------+----->| DE (Driver Enable)    |                   |
|                  |      |      |                     B |------------+     [R] R_term = 120 Ohm
|                  |      +----->| /RE (Receiver Enable) |            |      |   (Trở kết thúc)
|  GND             |-------------| GND                   |            |      +--------------------
|  3.3V            |-------------| VCC (+3.3V)           |            |       Dây B (Data -)
+------------------+             +-----------------------+            +---------------------------
```

---

### 2.3.2. Bảng Phân Bổ Chân Toàn Diện Trên STM32F407 (Master Pinout Table)

| Tên Chân STM32 | Tên Tín Hiệu | Chức Năng Phần Cứng / Alternate Function | Cấu Hình Trên STM32CubeMX | Kết Nối Chân Module USB-UART | Mức Điện Áp & Lưu Ý An Toàn |
| :---: | :---: | :---: | :---: | :--- | :--- |
| `PA2` | `USART2_TX` | `USART2_TX` (AF7) | Alternate Function Push-Pull, High Speed | Nối vào chân `RXD` của module CP2102/CH340 | Mức logic $3.3\text{V}$, đấu chéo TX $\to$ RX |
| `PA3` | `USART2_RX` | `USART2_RX` (AF7) | Alternate Function Push-Pull, High Speed | Nối vào chân `TXD` của module CP2102/CH340 | Mức logic $3.3\text{V}$, đấu chéo RX $\leftarrow$ TX |
| `GND` | `GND` | Mass hệ thống | Mass chung | Nối vào cọc `GND` của module USB-UART | **Bắt buộc nối chung mass để đồng mức 0V** |
| `PD12` | `ALARM_LED` | GPIO_Output | Output Push-Pull | Đèn LED xanh chỉ thị báo động bật/tắt | Mức logic $3.3\text{V}$ qua trở $330\Omega$ |
| `PE2` | `GAS_SENS` | GPIO_Input | Input Mode, Pull-down | Tín hiệu cảm biến khí gas MQ-2 | Mức logic $3.3\text{V}$ |
| `PE3` | `TEMP_SENS`| GPIO_Input | Input Mode, Pull-down | Tín hiệu công tắc nhiệt độ bảo vệ | Mức logic $3.3\text{V}$ |

---

## 2.4. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 2.1 ĐẾN 2.5 TRÊN STM32F407

### 2.4.1. Tái Định Tuyến Hàm `printf()` Chuẩn Thư Viện C Sang UART
Trong môi trường GCC (STM32CubeIDE), thêm đoạn mã sau vào `Core/Src/main.c`:

```c
#include <stdio.h>
#include <string.h>

extern UART_HandleTypeDef huart2;

/* Ghi đè hàm cấp thấp của thư viện stdio để chuyển hướng printf */
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}

int _write(int file, char *ptr, int len)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)ptr, len, HAL_MAX_DELAY);
    return len;
}
```

---

### 2.4.2. Mã Nguồn Hoàn Chỉnh Vận Hành Cổng Console PC

#### 1. Hàm Phục Vụ Truyền Thông UART Không Chặn (Example 2.1 & 2.2)
```c
/* Gửi chuỗi văn bản thuần túy */
void UART_SendString(const char *str)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)str, strlen(str), 100);
}

/* Máy phân tích lệnh không chặn (Non-blocking UART Task) */
void UART_CommandTask(uint8_t *pAlarmState, char *pSystemPIN)
{
    uint8_t rxChar;

    /* Thăm dò nhận 1 byte với thời gian timeout ngắn 5ms */
    if (HAL_UART_Receive(&huart2, &rxChar, 1, 5) == HAL_OK)
    {
        /* Phản hồi ký tự vừa bấm lên màn hình Terminal (Local Echo) */
        printf("%c\r\n", rxChar);

        switch (rxChar)
        {
            /* LỆNH '1': Xem trạng thái tổng quan báo động */
            case '1':
                if (*pAlarmState) {
                    printf("[STATUS] The alarm is ACTIVATED!\r\n");
                } else {
                    printf("[STATUS] The alarm is NOT activated.\r\n");
                }
                break;

            /* LỆNH '2': Kiểm tra nguồn gốc kích hoạt báo động (Gas hay Nhiệt độ) */
            case '2':
            {
                bool gasActive  = (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) == GPIO_PIN_SET);
                bool tempActive = (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3) == GPIO_PIN_SET);

                if (!gasActive && !tempActive) {
                    printf("[DIAGNOSTIC] No alarm source detected.\r\n");
                } else {
                    if (gasActive)  printf("[DIAGNOSTIC] Alarm triggered by GAS LEAK!\r\n");
                    if (tempActive) printf("[DIAGNOSTIC] Alarm triggered by OVER TEMPERATURE!\r\n");
                }
                break;
            }

            /* LỆNH '3': Tắt báo động từ xa qua máy tính (Clear Alarm) */
            case '3':
                *pAlarmState = 0;
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
                printf("[ACTION] Alarm has been CLEARED remotely.\r\n");
                break;

            /* LỆNH '4': Đổi mật mã bảo mật hệ thống (Change PIN Code) */
            case '4':
            {
                printf("Nhap ma PIN moi gom dung 4 chu so: ");
                char newPIN[5] = {0};
                /* Chờ nhận đủ 4 ký tự trong tối đa 5000ms */
                if (HAL_UART_Receive(&huart2, (uint8_t *)newPIN, 4, 5000) == HAL_OK)
                {
                    newPIN[4] = '\0';
                    strncpy(pSystemPIN, newPIN, 4);
                    printf("\r\n[SUCCESS] Ma PIN he thong da doi thanh: %s\r\n", pSystemPIN);
                }
                else
                {
                    printf("\r\n[TIMEOUT] Het thoi gian cho nhap ma!\r\n");
                }
                break;
            }

            default:
                printf("[HELP] Menu lenh hop le:\r\n");
                printf("  '1': Xem trang thai bao dong\r\n");
                printf("  '2': Chan doan nguon gay bao dong\r\n");
                printf("  '3': Tat bao dong tu xa\r\n");
                printf("  '4': Thay doi ma PIN he thong\r\n");
                break;
        }
        printf("\r\nNhap lua chon tiep theo: ");
    }
}
```

---

#### 2. Vòng Lặp Chính Điều Khiển
```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    uint8_t isAlarmActive = 0;
    char systemPIN[5] = "1234";

    printf("\r\n======================================================\r\n");
    printf("    SMART HOME CONSOLE SYSTEM - STM32F407 READY       \r\n");
    printf("======================================================\r\n");
    printf("Nhap lenh (1..4): ");

    while (1)
    {
        /* 1. Kiểm tra cảm biến phần cứng */
        if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) || HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3))
        {
            isAlarmActive = 1;
        }

        /* 2. Điều khiển đèn LED báo động PD12 */
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, isAlarmActive ? GPIO_PIN_SET : GPIO_PIN_RESET);

        /* 3. Phục vụ tác vụ truyền thông nối tiếp */
        UART_CommandTask(&isAlarmActive, systemPIN);
    }
}
```

---

## 2.5. "UNDER THE HOOD" & CASE STUDY (THEO GIÁO TRÌNH ARM)

### 2.5.1. Under the Hood (2.3.1): Độ Lệch Baudrate & Biên Độ Dung Sai Cho Phép
- Tại sao sai số Baudrate giữa hai thiết bị không được vượt quá **$2.5\%$**?
  - Trong khung truyền 10-bit (8N1), điểm lấy mẫu của Stop bit nằm ở chu kỳ bit thứ 10.
  - Sau 10 chu kỳ bit, độ trôi pha thời gian tích lũy bằng:
    $$\Delta t_{accumulated} = 10 \times \Delta T_{bit} = 10 \times \text{Error} \times T_{bit}$$
  - Để máy thu vẫn lấy mẫu trúng vào khoảng an toàn của Stop bit ($0.5 \times T_{bit}$ từ điểm giữa), độ trôi pha tối đa không được vượt quá $0.5 \times T_{bit}$:
    $$10 \times \text{Error} < 0.5 \implies \text{Error} < \frac{0.5}{10} = 5\%$$
  - Trong thực tế có nhiễu đường truyền và méo sườn xung, dung sai an toàn chuẩn công nghiệp quy định: **Sai số Baudrate tuyệt đối không được vượt quá $2.0\% \div 2.5\%$**.

### 2.5.2. Case Study (2.4.1): Bộ Truyền Phát Đo Lường Công Nghiệp (Industrial Transmitters)
- **Bối cảnh ứng dụng:** Trong nhà máy lọc dầu và xử lý hóa chất, các cảm biến áp suất và lưu lượng đặt cách xa phòng điều khiển trung tâm hàng nghìn mét.
- **Giải pháp kỹ thuật:**
  - Vi điều khiển thu thập tín hiệu áp suất, mã hóa thành chuỗi bản tin văn bản theo giao thức công nghiệp **Modbus-RTU** hoặc **HART Protocol**.
  - Tầng vật lý sử dụng IC SP3485 chuyển đổi tín hiệu UART sang chuẩn vi sai RS-485 truyền trên cáp xoay đôi có vỏ bọc chống nhiễu (Shielded Twisted Pair - STP).
  - Tốc độ truyền chọn ở mức $9600\text{ bps}$ kèm điện trở kết thúc đường dây $120\Omega$ ở hai đầu cáp để triệt tiêu sóng phản xạ cao tần, giúp dữ liệu truyền xa $1200\text{m}$ xuyên qua môi trường biến tần công suất lớn mà không bị sai sót một bit nào.

---

## 2.6. CÂU HỎI ÔN TẬP VÀ BÀI TẬP THIẾT KẾ CHƯƠNG 2

1. **Câu hỏi 1 (Khung truyền UART):** *Vẽ giản đồ thời gian chi tiết của khung truyền UART khi truyền ký tự chữ cái 'B' (Mã ASCII $0x42 = 01000010_2$) ở chuẩn 8N1. Đánh dấu rõ vị trí Start bit, các Data bits (LSB đến MSB), và Stop bit.*
2. **Câu hỏi 2 (Tính toán thanh ghi):** *Nếu xung nhịp bus APB1 của STM32F407 được cấu hình lại thành $36\text{ MHz}$, hãy tính toán giá trị nạp vào thanh ghi `USART2->BRR` để đạt tốc độ 115200 bps. Tính sai số phần trăm thực tế.*
3. **Câu hỏi 3 (Lỗi phần cứng):** *Trong trường hợp nào thì cờ báo lỗi Overrun Error (`ORE`) trong thanh ghi `USART_SR` bị kích hoạt? Nêu 2 phương pháp phần mềm để triệt tiêu hoàn toàn lỗi này.*
4. **Bài tập thiết kế mạch:** *Thiết kế sơ đồ nguyên lý mạch chuyển đổi giao tiếp giữa chân `PA2, PA3` của STM32F407 với chuẩn công nghiệp RS-485 sử dụng vi mạch SP3485 (nguồn 3.3V). Bổ sung chân GPIO `PD4` điều khiển chiều truyền nhận dữ liệu (`DE` và `\overline{RE}`).*
