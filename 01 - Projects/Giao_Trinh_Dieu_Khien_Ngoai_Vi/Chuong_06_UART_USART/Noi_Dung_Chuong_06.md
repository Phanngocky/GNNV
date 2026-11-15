# CHƯƠNG 6: GIAO THỨC TRUYỀN THÔNG NỐI TIẾP DỊ BỘ UART / USART & GIAO TIẾP MÁY TÍNH
*(Biên soạn dựa trên Chapter 2: Fundamentals of Serial Communication - Arm Education Textbook kết hợp Chuẩn kỹ thuật phần cứng STM32F4)*

---

## 6.1. MỤC TIÊU CỦA CHƯƠNG
Sau khi hoàn thành chương này, sinh viên ngành Kỹ thuật Điện tử có khả năng:
1. **Làm chủ nguyên lý truyền thông nối tiếp dị bộ:** Giải thích được cấu trúc khung truyền UART (Start bit, Data bits, Parity bit, Stop bit), cơ chế đồng bộ hóa tại sườn âm của Start bit và kỹ thuật lấy mẫu đa tần (Oversampling 8x / 16x).
2. **Nắm vững phần cứng tầng vật lý (Physical Layer):**
   - Phân biệt các mức điện áp logic: UART TTL 3.3V, TTL 5V, chuẩn RS-232 lưỡng cực ($\pm 3\text{V} \div \pm 15\text{V}$) và chuẩn truyền thông công nghiệp vi sai RS-485 (Half-duplex hai dây $A, B$).
   - Thiết kế mạch chuyển đổi mức điện áp sử dụng vi mạch MAX3232 và vi mạch thu phát RS-485 SP3485 (điều khiển chân $DE/\overline{RE}$).
3. **Tính toán và cấu hình thanh ghi phần cứng:** 
   - Tính toán chính xác giá trị thanh ghi tốc độ Baud (`USART_BRR`) từ tần số xung nhịp bus $PCLK1$ / $PCLK2$, đánh giá phần trăm sai số baud rate do sai số thạch anh.
   - Thao tác thành thạo các thanh ghi trạng thái `SR`, thanh ghi dữ liệu `DR`, và các thanh ghi điều khiển `CR1`, `CR2`.
4. **Phát triển phần mềm nhúng giao tiếp máy tính (PC Interfacing):**
   - Triển khai truyền nhận chuỗi ký tự bằng cả 3 phương pháp: Polling (Thăm dò cờ), Ngắt (Interrupt với bộ đệm vòng Ring Buffer) và thư viện HAL.
   - Hiện thực hóa hoàn chỉnh hệ thống giám sát và điều khiển Smart Home từ xa qua giao diện dòng lệnh máy tính (Examples 2.1 $\div$ 2.5 của ARM).

---

## 6.2. NGUYÊN LÝ TRUYỀN THÔNG NỐI TIẾP UART / USART

### 1. Khung truyền dữ liệu (UART Frame Format)
Trong truyền thông dị bộ (Asynchronous), hai thiết bị không chia sẻ chung một đường dây xung nhịp (Clock line). Để bên nhận có thể giải mã chính xác các bit dữ liệu, hai bên phải thống nhất trước một tốc độ truyền gọi là **Baud rate** (số bit truyền trong 1 giây).

```
Đường truyền nghỉ (Idle: Mức '1')
   │   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───────
   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘
   Start    Bit 0   Bit 1   Bit 2   Bit 3   Bit 4   Bit 5   Bit 6   Bit 7  Parity   Stop
   ('0')   (LSB)                                                   (MSB)  (Tùy chọn) ('1')
   ├─1Tbit─┼─1Tbit─┼─1Tbit─┼─1Tbit─┼─1Tbit─┼─1Tbit─┼─1Tbit─┼─1Tbit─┼─1Tbit─┼─1Tbit─┤
```

- **Đường truyền nghỉ (Idle State):** Luôn duy trì ở mức điện áp cao logic '1' (Marking state).
- **Bit bắt đầu (Start Bit):** Luôn là 1 bit mức logic '0' (Spacing state). Sườn âm ($1 \rightarrow 0$) này là tín hiệu kích hoạt bộ thu UART bắt đầu đếm thời gian.
- **Các bit dữ liệu (Data Bits):** Thường gồm 7, 8 hoặc 9 bit (phổ biến nhất là 8-bit). Bit có trọng số thấp nhất (**LSB**) luôn được truyền đi trước, bit có trọng số cao nhất (**MSB**) truyền sau cùng.
- **Bit kiểm tra chẵn lẻ (Parity Bit - Tùy chọn):** 
  - *Even Parity (Chẵn):* Tổng số bit '1' trong Data + Parity là một số chẵn.
  - *Odd Parity (Lẻ):* Tổng số bit '1' trong Data + Parity là một số lẻ.
- **Bit kết thúc (Stop Bit):** Là mức logic cao '1' có độ dài 1, 1.5 hoặc 2 bit, cho phép đường truyền quay lại trạng thái nghỉ và chuẩn bị cho khung truyền tiếp theo.

### 2. Kỹ thuật lấy mẫu tín hiệu (Oversampling) & Dung sai tốc độ Baud
Khối thu UART bên trong STM32F4 sử dụng bộ đếm xung nhịp lấy mẫu cao hơn tần số baud rate (thường là **16 lần** hoặc **8 lần**):
- Khi phát hiện sườn âm của Start bit, bộ thu bắt đầu đếm xung nhịp lấy mẫu.
- Tại xung nhịp thứ 8 (giữa Start bit), bộ thu kiểm tra lại: Nếu tín hiệu vẫn là mức 0 $\rightarrow$ Khẳng định đây là Start bit hợp lệ, loại bỏ các xung nhiễu nhọn ngẫu nhiên.
- Sau đó, cứ sau mỗi 16 chu kỳ xung nhịp nội (đúng tâm điểm của từng bit), bộ thu thực hiện lấy mẫu 3 lần (tại các xung thứ 7, 8, 9) và dùng giải thuật biểu quyết số đông (Majority rule) để xác định giá trị bit là 0 hay 1.
- **Giới hạn sai số Baud rate:** Để không xảy ra hiện tượng lệch pha tích lũy dẫn đến lỗi khung (Framing Error), sai số tần số giữa hai thiết bị phát và thu **không được vượt quá $\pm 2.5\%$**.

---

## 6.3. TẦNG VẬT LÝ VÀ THIẾT KẾ MẠCH ĐIỆN TỬ GHÉP NỐI

### 1. Bảng so sánh các chuẩn truyền thông nối tiếp tầng vật lý:
| Tiêu chí kỹ thuật | UART TTL (3.3V / 5V) | Chuẩn RS-232 | Chuẩn công nghiệp RS-485 |
| :--- | :---: | :---: | :---: |
| **Kiểu tín hiệu** | Bất đối xứng (Single-ended) | Bất đối xứng (Single-ended) | Vi sai cân bằng (Differential) |
| **Mức điện áp logic** | Mức 0: $0\text{V}$; Mức 1: $3.3\text{V}/5\text{V}$ | Mức 0: $+3\text{V} \div +15\text{V}$<br>Mức 1: $-3\text{V} \div -15\text{V}$ | $V_A - V_B > +200\text{mV}$: Mức 1<br>$V_A - V_B < -200\text{mV}$: Mức 0 |
| **Chế độ truyền** | Song công toàn phần (Full-duplex) | Full-duplex | Bán song công (Half-duplex 2 dây) |
| **Khoảng cách truyền tối đa** | $< 1\text{ mét}$ | $\le 15\text{ mét}$ | **Lên tới 1200 mét** |
| **Tốc độ truyền tối đa** | Lên tới vài Mbps | Giới hạn $\le 115.2\text{ kbps}$ | Lên tới 10 Mbps (ở khoảng cách ngắn) |
| **Số thiết bị trên bus** | 1 Master - 1 Slave | 1 Điểm - 1 Điểm (Point-to-Point) | **Đa điểm (Multi-drop, lên tới 32/128 nút)** |

### 2. Mạch ghép nối RS-232 với vi mạch MAX3232
STM32F4 hoạt động ở mức $3.3\text{V}$, trong khi cổng COM RS-232 của máy tính công nghiệp sử dụng điện áp lưỡng cực âm/dương. Vi mạch MAX3232 tích hợp sẵn các mạch bơm điện áp dùng tụ điện (Charge Pump) để nhân đôi và đảo cực tính điện áp từ nguồn $3.3\text{V}$ thành $\pm 6\text{V} \div \pm 9\text{V}$ mà không cần nguồn phụ ngoài.

### 3. Mạch truyền thông công nghiệp RS-485 (Half-duplex)
Trong nhà máy, nhiễu điện từ do động cơ và biến tần rất lớn. Chuẩn RS-485 truyền tín hiệu trên cặp dây xoắn vi sai ($A$ và $B$). Khi có nhiễu cảm ứng, cả 2 dây đều bị nâng áp như nhau, nhưng hiệu điện thế $(V_A - V_B)$ không thay đổi, giúp loại bỏ hoàn toàn nhiễu đồng pha (Common-mode rejection).

```mermaid
flowchart LR
    MCU["STM32F4 (3.3V)"] -->|TX (PA9)| SP485["IC SP3485 (3.3V)"]
    SP485 -->|RX (PA10)| MCU
    MCU -->|DE/RE Control (PD4)| SP485
    SP485 <-->|Đường vi sai A, B| Bus["Cặp cáp xoắn đôi RS-485
    + Trở đầu cuối 120 Ohm"]
```

---

## 6.4. TÍNH TOÁN THANH GHI TỐC ĐỘ BAUD (`USART_BRR`) TRÊN STM32F4

Tốc độ Baud được tạo ra bằng cách chia tần số xung nhịp của bus APB chứa khối USART thông qua thanh ghi `USART_BRR`:
$$Baud = \frac{f_{PCLK}}{8 \times (2 - OVER8) \times USARTDIV}$$

Trong chế độ mặc định lấy mẫu 16 lần ($OVER8 = 0$):
$$USARTDIV = \frac{f_{PCLK}}{16 \times Baud}$$

Thanh ghi `USART_BRR` (32-bit) lưu trữ giá trị $USARTDIV$ dưới dạng:
- Các bit `[15:4]` (12-bit): Phần nguyên (`DIV_Mantissa`).
- Các bit `[3:0]` (4-bit): Phần thập phân (`DIV_Fraction`), tính bằng: $\text{Fraction} = \text{Phần lẻ} \times 16$.

### Ví dụ tính toán chi tiết:
- Giả sử cấu hình `USART1` (nằm trên bus APB2 có $f_{PCLK2} = 84\text{ MHz}$), cần thiết lập tốc độ truyền chuẩn $Baud = 115200\text{ bps}$:
  $$USARTDIV = \frac{84 \times 10^6}{16 \times 115200} = \frac{84000000}{1843200} \approx 45.5729$$
- Phần nguyên: $Mantissa = 45 = 0x2D$.
- Phần lẻ: $Fraction = 0.5729 \times 16 = 9.166 \approx 9 = 0x9$.
- Giá trị nạp vào thanh ghi `USART1->BRR`:
  $$BRR = (45 \ll 4) | 9 = (0x2D \ll 4) | 0x9 = 0x02D9$$
- **Tính toán sai số thực tế:**
  $$Baud_{actual} = \frac{84000000}{16 \times (45 + \frac{9}{16})} = \frac{84000000}{16 \times 45.5625} = \frac{84000000}{729} \approx 115226.3\text{ bps}$$
  $$Error = \frac{115226.3 - 115200}{115200} \times 100\% = +0.023\%$$
  *(Sai số chỉ $+0.023\%$, nhỏ hơn rất nhiều so với ngưỡng an toàn $2.5\%$, đường truyền hoạt động hoàn hảo).*

---

## 6.5. HIỆN THỰC HÓA CÁC VÍ DỤ CỐT LÕI (EXAMPLES 2.1 $\div$ 2.5)

### VÍ DỤ 2.1: GIÁM SÁT TRẠNG THÁI HỆ THỐNG BÁO ĐỘNG QUA MÁY TÍNH PC (Example 2.1)
- **Mô tả hành vi:** Vi điều khiển liên tục theo dõi hệ thống. Khi người dùng nhấn phím `'1'` trên bàn phím phần mềm Terminal máy tính, hệ thống sẽ gửi chuỗi phản hồi báo cáo trạng thái còi báo động: `"The alarm is activated"` hoặc `"The alarm is not activated"`.

#### 1. Triển khai theo chuẩn Mbed C++ (Giáo trình ARM - Listing 2.1)
```cpp
#include "mbed.h"
#include "arm_book_lib.h"

// Khởi tạo đối tượng cổng nối tiếp USB ảo nối với PC
UnbufferedSerial uartUsb(USBTX, USBRX, 115200);

DigitalIn gasDetector(D2);
DigitalIn overTempDetector(D3);
DigitalOut alarmLed(LED1);

void uartTask(bool alarmState);

int main()
{
    bool alarmState = OFF;
    gasDetector.mode(PullDown);
    overTempDetector.mode(PullDown);
    alarmLed = OFF;

    while (true) {
        if (gasDetector || overTempDetector) {
            alarmState = ON;
        }
        alarmLed = alarmState;
        
        // Gọi tác vụ giám sát giao tiếp UART
        uartTask(alarmState);
    }
}

void uartTask(bool alarmState)
{
    char receivedChar = '\0';
    // Đọc không chặn 1 byte từ cổng nối tiếp nếu có
    if (uartUsb.read(&receivedChar, 1)) {
        if (receivedChar == '1') {
            if (alarmState) {
                uartUsb.write("The alarm is activated\r\n", 24);
            } else {
                uartUsb.write("The alarm is not activated\r\n", 28);
            }
        }
    }
}
```

#### 2. Triển khai theo chuẩn C Công nghiệp (STM32 HAL Library)
```c
#include "main.h"
#include <string.h>

UART_HandleTypeDef huart3; // USART3 kết nối Virtual COM Port qua ST-Link

void UART_SendString(char *str)
{
    HAL_UART_Transmit(&huart3, (uint8_t*)str, strlen(str), 100);
}

void UART_Task(uint8_t alarmState)
{
    uint8_t rxData;
    // Kiểm tra nhận 1 byte với Timeout ngắn (10ms - Non-blocking polling)
    if (HAL_UART_Receive(&huart3, &rxData, 1, 10) == HAL_OK)
    {
        if (rxData == '1')
        {
            if (alarmState)
            {
                UART_SendString("The alarm is activated\r\n");
            }
            else
            {
                UART_SendString("The alarm is not activated\r\n");
            }
        }
    }
}
```

#### 3. Triển khai cấp thanh ghi trực tiếp (Bare-Metal CMSIS)
```c
#include "stm32f4xx.h"

void USART3_Init(void)
{
    // 1. Cấp clock cho GPIOD (TX: PD8, RX: PD9) và USART3
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIODEN;
    RCC->APB1ENR |= RCC_APB1ENR_USART3EN;

    // 2. Cấu hình PD8, PD9 sang chế độ Alternate Function (AF7)
    GPIOD->MODER &= ~((3U << 16) | (3U << 18));
    GPIOD->MODER |=  ((2U << 16) | (2U << 18));
    GPIOD->AFR[1] |= (7U << 0) | (7U << 4); // AF7: USART3

    // 3. Cấu hình Baudrate 115200 với APB1 = 42MHz
    // USARTDIV = 42000000 / (16 * 115200) = 22.786 -> Mantissa = 22 (0x16), Fraction = 0.786 * 16 = 13 (0xD)
    USART3->BRR = (22 << 4) | 13;

    // 4. Kích hoạt Bộ phát (TE), Bộ thu (RE) và Khối USART (UE)
    USART3->CR1 |= (USART_CR1_TE | USART_CR1_RE | USART_CR1_UE);
}

void USART3_SendChar(char c)
{
    // Chờ cờ TXE (Transmit data register empty) được set lên 1
    while (!(USART3->SR & USART_SR_TXE));
    USART3->DR = (uint8_t)c;
}

uint8_t USART3_ReadCharNonBlocking(char *c)
{
    // Kiểm tra cờ RXNE (Read data register not empty)
    if (USART3->SR & USART_SR_RXNE)
    {
        *c = (char)(USART3->DR & 0xFF);
        return 1; // Nhận thành công
    }
    return 0; // Chưa có dữ liệu
}
```

---

### VÍ DỤ 2.2: GIÁM SÁT CHI TIẾT TỪNG CẢM BIẾN DÙNG LỆNH `switch-case` (Example 2.2)
- Khi nhấn `'1'`: Báo cáo tình trạng còi báo động.
- Khi nhấn `'2'`: Báo cáo chi tiết nguồn gây báo động (do Khí Gas hay do Quá nhiệt).
```c
void UART_Task_MultiCommand(uint8_t alarmState, uint8_t gas, uint8_t temp)
{
    char cmd;
    if (USART3_ReadCharNonBlocking(&cmd))
    {
        switch (cmd)
        {
            case '1':
                UART_SendString(alarmState ? "Alarm: ON\r\n" : "Alarm: OFF\r\n");
                break;
            case '2':
                if (gas)  UART_SendString("Gas detected!\r\n");
                if (temp) UART_SendString("Over-temperature detected!\r\n");
                if (!gas && !temp) UART_SendString("All sensors normal.\r\n");
                break;
            default:
                UART_SendString("Unknown command!\r\n");
                break;
        }
    }
}
```

---

### VÍ DỤ 2.3: TẮT BÁO ĐỘNG TỪ XA TỪ BÀN PHÍM MÁY TÍNH (Example 2.3)
- Bổ sung phím `'3'`: Cho phép người quản trị từ xa gửi lệnh tắt chốt báo động mà không cần bấm nút trực tiếp trên mạch.
```c
case '3':
    if (alarmState)
    {
        alarmState = 0;
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
        UART_SendString("Alarm deactivated from PC terminal successfully.\r\n");
    }
    else
    {
        UART_SendString("The alarm is already deactivated.\r\n");
    }
    break;
```

---

### VÍ DỤ 2.4 & 2.5: TỐI ƯU QUẢN LÝ MÃ BẰNG MẢNG VÀ ĐỔI MẬT MÃ TỪ XA (Examples 2.4 & 2.5)
- **Tư duy thiết kế phần mềm nhúng:** Thay vì khai báo rời rạc từng biến nút bấm (`aButton`, `bButton`, `cButton`...), mã bảo mật được lưu trong mảng số nguyên:
```c
#define CODE_LENGTH 4
uint8_t secretCode[CODE_LENGTH] = {1, 2, 1, 2}; // Mật mã mặc định

// Khi người dùng bấm phím '5' từ PC: Chuyển sang chế độ nhập mật mã mới
case '5':
    UART_SendString("Please enter new 4-digit security code from PC:\r\n");
    for (int i = 0; i < CODE_LENGTH; i++)
    {
        char newDigit;
        while (!USART3_ReadCharNonBlocking(&newDigit)); // Chờ nhận từng số
        secretCode[i] = newDigit - '0';
        USART3_SendChar(newDigit); // Echo lại ký tự ra màn hình
    }
    UART_SendString("\r\nNew code configured successfully!\r\n");
    break;
```

---

## 6.6. BÀI TẬP VÀ CÂU HỎI THIẾT KẾ MẠCH NỐI TIẾP
1. **Tính toán:** Tính toán giá trị thanh ghi `USART_BRR` cho `USART2` (trên bus APB1 có $f_{PCLK1} = 42\text{ MHz}$) để đạt tốc độ Baud chuẩn $9600\text{ bps}$. Đánh giá phần trăm sai số.
2. **Thiết kế mạch:** Vẽ sơ đồ ghép nối vi điều khiển STM32F4 với vi mạch thu phát RS-485 SP3485. Phân tích nguyên lý điều khiển chân $DE/\overline{RE}$ để chuyển đổi giữa trạng thái phát và nhận.
3. **Lập trình mở rộng:** Viết hàm nhận dữ liệu UART sử dụng **Ngắt nhận ký tự (`USART_IT_RXNE`)** và kỹ thuật bộ đệm vòng **Ring Buffer 128 byte** để không bị mất dữ liệu khi máy tính truyền một gói tin dài với tốc độ 115200 bps.
