# CHƯƠNG 2: CƠ SỞ TRUYỀN THÔNG NỐI TIẾP VÀ GIAO TIẾP MÁY TÍNH (UART) TRÊN STM32F407
*(Biên soạn dựa trên Chapter 2: Fundamentals of Serial Communication - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 2.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã hiểu nguyên lý truyền dữ liệu nối tiếp cơ bản từ học phần trước; chương này tập trung vào làm chủ giao thức phần cứng trên STM32F407, kỹ thuật phân tách hàm chức năng (Modularization into functions), quản lý bộ nhớ mảng và giao tiếp máy tính thời gian thực.
- **Mục tiêu chi tiết:**
  1. Cấu hình khối `USART2` (hoặc `USART1`) trên STM32F407 để truyền nhận ký tự, chuỗi ký tự với phần mềm Terminal trên máy tính PC (Baud rate 115200 bps, 8 bit data, no parity, 1 stop bit).
  2. Kỹ thuật tổ chức mã nguồn nhúng thành các hàm mô-đun chức năng độc lập (`uartTask`), tránh viết dồn toàn bộ mã vào hàm `main()`.
  3. Hiện thực hóa các ví dụ từ Example 2.1 đến 2.5 của giáo trình ARM trên STM32F407: Giám sát trạng thái còi báo, kiểm tra nguồn gây báo động bằng lệnh `switch-case`, tắt báo động từ xa, tối ưu hóa quản lý mã bảo mật bằng mảng số nguyên và thay đổi mật mã hệ thống từ xa qua giao diện PC.
  4. Phân tích chi tiết tầng vật lý và thanh ghi: Tính toán thanh ghi `USART_BRR` theo xung nhịp bus APB1 ($42\text{ MHz}$) và APB2 ($84\text{ MHz}$) của STM32F407, các cờ báo lỗi phần cứng (`FE`, `NE`, `ORE`).

---

## 2.2. CẤU HÌNH PHẦN CỨNG UART TRÊN STM32F407

### 1. Phân bổ chân phần cứng:
- Sử dụng khối **`USART2`** (nằm trên bus APB1 với tần số $f_{PCLK1} = 42\text{ MHz}$):
  - Chân truyền dữ liệu (TX): `PA2` (Alternate Function AF7).
  - Chân nhận dữ liệu (RX): `PA3` (Alternate Function AF7).
  - Cổng kết nối: Nối với cáp chuyển đổi USB-TTL (CP2102 / CH340) cắm vào cổng USB máy tính.

### 2. Tính toán thanh ghi tốc độ Baud (`USART2_BRR`) tại 115200 bps:
Với $f_{PCLK1} = 42\text{ MHz}$ và chế độ lấy mẫu 16 lần ($OVER8 = 0$):
$$USARTDIV = \frac{42 \times 10^6}{16 \times 115200} \approx 22.78645$$
- Phần nguyên: $Mantissa = 22 = 0x16$.
- Phần thập phân: $Fraction = 0.78645 \times 16 = 12.58 \approx 13 = 0x0D$.
- Giá trị nạp vào thanh ghi: `USART2->BRR = (22 << 4) | 13 = 0x016D`.
- Sai số tốc độ: Chỉ $+0.06\%$ (hoàn toàn triệt tiêu nguy cơ lỗi khung truyền).

---

## 2.3. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 2.1 ĐẾN 2.5 TRÊN STM32F407

### VÍ DỤ 2.1: GIÁM SÁT TRẠNG THÁI BÁO ĐỘNG TỪ MÁY TÍNH PC (Listing 2.1)
- **Yêu cầu:** Tách hàm `uartTask()` độc lập. Khi người dùng bấm phím `'1'` trên máy tính, STM32F407 gửi lại chuỗi báo cáo: `"The alarm is activated\r\n"` hoặc `"The alarm is not activated\r\n"`.

```c
#include "main.h"
#include <string.h>

UART_HandleTypeDef huart2;

void UART_SendString(char *str)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)str, strlen(str), 100);
}

// Hàm mô-đun giám sát UART theo cấu trúc giáo trình ARM
void uartTask(uint8_t alarmState)
{
    uint8_t rxChar;
    // Kiểm tra nhận không chặn (Non-blocking polling 10ms)
    if (HAL_UART_Receive(&huart2, &rxChar, 1, 10) == HAL_OK)
    {
        if (rxChar == '1')
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

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    uint8_t alarmState = 0;

    while (1)
    {
        // 1. Quét cảm biến
        if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) || HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3))
        {
            alarmState = 1;
        }

        // 2. Xuất còi/đèn
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, alarmState ? GPIO_PIN_SET : GPIO_PIN_RESET);

        // 3. Thực thi tác vụ truyền thông UART
        uartTask(alarmState);
    }
}
```

---

### VÍ DỤ 2.2: GIÁM SÁT NGUỒN GÂY BÁO ĐỘNG SỬ DỤNG CẤU TRÚC `switch` (Listing 2.2)
- Khi nhận phím `'1'`: Báo cáo trạng thái chung của còi báo động.
- Khi nhận phím `'2'`: Báo cáo chi tiết nguồn gây báo động (do Gas `PE2` hay do Quá nhiệt `PE3`).
```c
void uartTask(uint8_t alarmState, uint8_t gas, uint8_t temp)
{
    uint8_t rxChar;
    if (HAL_UART_Receive(&huart2, &rxChar, 1, 10) == HAL_OK)
    {
        switch (rxChar)
        {
            case '1':
                UART_SendString(alarmState ? "The alarm is activated\r\n" : "The alarm is not activated\r\n");
                break;
            case '2':
                if (gas)  UART_SendString("Gas is detected\r\n");
                if (temp) UART_SendString("Over-temperature is detected\r\n");
                if (!gas && !temp) UART_SendString("No alarm triggers detected\r\n");
                break;
            default:
                break;
        }
    }
}
```

---

### VÍ DỤ 2.3: TẮT BÁO ĐỘNG TỪ XA QUA MÁY TÍNH PC (Listing 2.3)
- Bổ sung lệnh `'3'`: Cho phép người quản trị tắt chốt báo động trực tiếp từ bàn phím máy tính.
```c
case '3':
    if (alarmState)
    {
        alarmState = 0; // Tắt chốt báo động
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
        UART_SendString("The alarm has been deactivated\r\n");
    }
    else
    {
        UART_SendString("The alarm is already not active\r\n");
    }
    break;
```

---

### VÍ DỤ 2.4 & 2.5: TỐI ƯU HÓA QUẢN LÝ MÃ BẰNG MẢNG VÀ ĐỔI MẬT MÃ TỪ XA (Listing 2.4 & 2.5)
- **Tư duy quản trị phần mềm nhúng:** Thay vì viết lặp lại các nút bấm bằng biến đơn lẻ, tổ chức các nút và mã PIN dưới dạng mảng:
```c
#define NUMBER_OF_BUTTONS 4
uint8_t codeSequence[NUMBER_OF_BUTTONS] = {1, 2, 1, 2}; // Mật mã mặc định

// Bổ sung case '5': Người dùng nhập mật mã mới 4 chữ số từ máy tính
case '5':
    UART_SendString("Enter new 4-digit security code:\r\n");
    for (int i = 0; i < NUMBER_OF_BUTTONS; i++)
    {
        uint8_t newDigit;
        while (HAL_UART_Receive(&huart2, &newDigit, 1, HAL_MAX_DELAY) != HAL_OK);
        codeSequence[i] = newDigit - '0';
        HAL_UART_Transmit(&huart2, &newDigit, 1, 10); // Echo phản hồi
    }
    UART_SendString("\r\nNew security code updated successfully!\r\n");
    break;
```

---

## 2.4. "UNDER THE HOOD" & CASE STUDY (THEO SÁCH ARM)
- **Under the Hood (2.3.1):** 
  - Khung truyền UART chuẩn NRZ (Non-Return-to-Zero).
  - Phân tích 4 lỗi phần cứng phổ biến của UART: **Parity Error (PE)**, **Framing Error (FE - sai Stop bit do lệch baudrate)**, **Noise Error (NE - nhiễu đường truyền)**, và **Overrun Error (ORE - dữ liệu mới tràn vào khi CPU chưa kịp đọc byte cũ)**.
- **Case Study (2.4.1):** Bộ truyền phát thông minh trong công nghiệp (Industrial Transmitter): Ứng dụng giao tiếp UART kết hợp modem HART và giao tiếp truyền xa RS-485 Modbus để truyền thông số nhiệt độ/áp suất về phòng điều khiển trung tâm SCADA.
