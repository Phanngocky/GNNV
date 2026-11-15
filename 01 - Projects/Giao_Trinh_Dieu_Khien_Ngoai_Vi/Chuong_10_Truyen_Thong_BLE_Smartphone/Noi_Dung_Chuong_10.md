# CHƯƠNG 10: TRUYỀN THÔNG KHÔNG DÂY BLUETOOTH LOW ENERGY (BLE) VỚI ĐIỆN THOẠI TRÊN STM32F407
*(Biên soạn dựa trên Chapter 10: Bluetooth Low Energy Communication with a Smartphone - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 10.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã thành thạo giao tiếp nối tiếp có dây (UART) ở Chương 2; chương này mở rộng sang **Truyền thông không dây tầm ngắn tiết kiệm năng lượng Bluetooth Low Energy (BLE)** để kết nối trực tiếp hệ thống nhúng STM32F407 với ứng dụng di động trên Smartphone.
- **Mục tiêu chi tiết:**
  1. Nắm vững kiến trúc giao thức BLE: Băng tần vô tuyến $2.4\text{ GHz}$, vai trò kết nối Generic Access Profile (GAP: Central vs Peripheral, Advertising), kiến trúc cơ sở dữ liệu thuộc tính Generic Attribute Profile (GATT: Server vs Client, Services, Characteristics, UUID 128-bit).
  2. Ghép nối module BLE (HC-08 / HM-10) với STM32F407 qua khối `USART3` (`PB10` - TX, `PB11` - RX).
  3. Hiện thực hóa các ví dụ từ Example 10.1 đến 10.4 trên STM32F407: Nhận chuỗi lệnh từ ứng dụng Smartphone để điều khiển động cơ đóng/mở cổng, định dạng gói tin bản tin gửi báo cáo trạng thái hệ thống và triển khai hàm trễ không chặn bằng con trỏ hàm.
  4. Phân tích tối ưu hóa mức tiêu thụ năng lượng (Low Power Consumption) trong thiết bị nhúng chạy pin.

---

## 10.2. PHÂN BỔ PHẦN CỨNG BLE TRÊN STM32F407

```mermaid
flowchart LR
    Phone["Smartphone App (BLE Central / Client)"] <-->|Sóng vô tuyến BLE 2.4GHz| BLE_Module["Module BLE HC-08 / HM-10 (Peripheral)"]
    BLE_Module -->|TX Module nối RX MCU| PB11["Chân PB11 (USART3_RX)"]
    BLE_Module <--|RX Module nối TX MCU| PB10["Chân PB10 (USART3_TX)"]
    STM32["STM32F407 Controller"] --> MotorGate["Động cơ mở cổng tự động"]
```

---

## 10.3. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 10.1 ĐẾN 10.4 TRÊN STM32F407

### VÍ DỤ 10.1: ĐIỀU KHIỂN ĐÓNG MỞ CỔNG TỪ ĐIỆN THOẠI THÔNG MINH (Listing 10.1)
- Trên điện thoại thông minh, người dùng sử dụng ứng dụng BLE Scanner / BLE Terminal để kết nối tới module.
- Khi người dùng gửi chuỗi `"OPEN\r\n"` $\rightarrow$ STM32F407 kích hoạt Relay mở cổng; khi gửi `"CLOSE\r\n"` $\rightarrow$ Kích hoạt Relay đóng cổng:

```c
#include "main.h"
#include <string.h>

UART_HandleTypeDef huart3;

void BLE_Task(void)
{
    static char bleRxBuffer[32];
    static uint8_t index = 0;
    uint8_t rxChar;

    // Đọc không chặn từng ký tự nhận từ Module BLE
    if (HAL_UART_Receive(&huart3, &rxChar, 1, 5) == HAL_OK)
    {
        if (rxChar == '\r' || rxChar == '\n')
        {
            if (index > 0)
            {
                bleRxBuffer[index] = '\0'; // Kết thúc chuỗi

                if (strcmp(bleRxBuffer, "OPEN") == 0)
                {
                    Motor_OpenGate();
                    HAL_UART_Transmit(&huart3, (uint8_t*)"GATE: OPENED\r\n", 14, 50);
                }
                else if (strcmp(bleRxBuffer, "CLOSE") == 0)
                {
                    Motor_CloseGate();
                    HAL_UART_Transmit(&huart3, (uint8_t*)"GATE: CLOSED\r\n", 14, 50);
                }
                index = 0; // Reset bộ đệm
            }
        }
        else if (index < sizeof(bleRxBuffer) - 1)
        {
            bleRxBuffer[index++] = rxChar;
        }
    }
}
```

---

### VÍ DỤ 10.2: BÁO CÁO TRẠNG THÁI HỆ THỐNG LÊN SMARTPHONE (Listing 10.2)
- Định kỳ mỗi 2 giây, STM32F407 tự động đóng gói trạng thái toàn bộ ngôi nhà gửi sang điện thoại:
```c
void BLE_Report_Status(float temp, bool gasAlert, bool gateOpen)
{
    char statusPkt[64];
    snprintf(statusPkt, sizeof(statusPkt), "TEMP:%.1f,GAS:%d,GATE:%d\r\n",
             temp, gasAlert ? 1 : 0, gateOpen ? 1 : 0);

    HAL_UART_Transmit(&huart3, (uint8_t*)statusPkt, strlen(statusPkt), 50);
}
```

---

## 10.4. "UNDER THE HOOD" & CASE STUDY (THEO SÁCH ARM)
- **Under the Hood (10.3.1):** 
  - Kiến trúc phân lớp giao thức BLE: Controller (Physical Layer, Link Layer) và Host (L2CAP, SM, ATT, GATT, GAP).
  - So sánh Bluetooth Cổ điển (Classic Bluetooth - BR/EDR) và Bluetooth Năng lượng thấp (BLE): BLE tối ưu hóa thời gian phát sóng cực ngắn (vài mili-giây) và dòng điện ở chế độ nghỉ chỉ vài micro-Ampe, cho phép thiết bị hoạt động nhiều năm chỉ với 1 viên pin cúc áo CR2032.
- **Case Study (10.4.1):** Thiết bị Giao tiếp Không dây Công nghiệp (Wireless Bolt): Sử dụng module BLE gắn trên nóc máy móc nhà xưởng để cho phép kỹ sư bảo trì sử dụng máy tính bảng đọc thông số cảm biến và chẩn đoán lỗi không dây an toàn mà không cần mở tủ điện cao thế.
