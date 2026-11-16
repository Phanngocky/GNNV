# BÀI THỰC HÀNH SỐ 13: TRUYỀN THÔNG KHÔNG DÂY BLUETOOTH LOW ENERGY (BLE) VỚI ĐIỆN THOẠI THÔNG MINH
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết GAP/GATT BLE $\rightarrow$ Cấu hình AT Commands $\rightarrow$ Lập trình Điều khiển $\rightarrow$ Thử nghiệm App Smartphone)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Kiến trúc BLE GAP/GATT), CLO 2 (Mạch BLE & UART3), CLO 3 (Lập trình truyền thông không dây), CLO 5 (Phân tích gói tin BLE)  
**Căn cứ giáo trình:** *ARM Education - Chương 10: Wireless Bluetooth Low Energy (Ex 10.1 $\div$ 10.4)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Truyền thông Vô tuyến Năng lượng Thấp (BLE):**
   - Phân tích kiến trúc giao thức BLE (Bluetooth 4.0 / 4.2 / 5.0) trên dải tần số $2.4\text{ GHz}$ ISM: Phân tầng Controller, Host và Application.
   - Nắm vững mô hình phân vai **GAP (Generic Access Profile)**: Chế độ Ngoại vi (Peripheral - Quảng bá gói tin Advertising) và Thiết bị Trung tâm (Central - Điện thoại quét và kết nối).
   - Nắm vững kiến trúc dữ liệu **GATT (Generic Attribute Profile)**: Máy chủ dữ liệu (GATT Server), Hồ sơ (Profile), Dịch vụ (Services), và Đặc tính (Characteristics) với định danh UUID 128-bit/16-bit.
   - Hiểu rõ cơ chế tiết kiệm năng lượng (Low-Power Duty Cycling): Thiết bị ngủ phần lớn thời gian và chỉ thức dậy phát xung vô tuyến trong vài mili-giây.
2. **Kỹ thuật Lập trình:**
   - Cấu hình module BLE (HC-08 hoặc HM-10) bằng tập lệnh **AT Commands** qua `USART3` (`PB10` TX, `PB11` RX): Đổi tên thiết bị (`AT+NAMESmartHome`), đặt tốc độ Baudrate, cấu hình công suất phát.
   - Lập trình nhận chuỗi ký tự điều khiển từ xa gửi từ Smartphone: Đóng/Mở cổng trượt tự động ('O' / 'C'), Tắt báo động ('K').
   - Lập trình gửi gói tin định kỳ cập nhật nhiệt độ và trạng thái cảm biến lên giao diện điện thoại thông minh (Ex 10.1 $\div$ 10.4).
3. **Thực nghiệm Ứng dụng Di động:**
   - Cài đặt ứng dụng *Serial Bluetooth Terminal* hoặc *nRF Connect for Mobile* trên điện thoại Android/iOS.
   - Quét sóng vô tuyến, kết nối GATT và thực nghiệm điều khiển hệ thống nhà thông minh trong bán kính $10\text{m}$.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Module thu phát BLE | Chip CC2541 (HM-10 / HC-08 / JDY-08) | 1 | Chuẩn Bluetooth 4.0 BLE |
| 3 | Điện thoại thông minh | Smartphone Android hoặc Apple iOS | 1 | Cài sẵn App BLE Terminal |
| 4 | Mạch cầu Relay & Động cơ | Đã chuẩn bị từ Bài Lab 9 | 1 bộ | Điều khiển đóng mở cổng |
| 5 | Dây cắm đực-cái Breadboard | Dây nối mềm | 4 sợi | Đấu nối module BLE |

---

## 3. THIẾT KẾ MẠCH NGUYÊN LÝ VÀ ĐẤU NỐI CHÂN (PINOUT)

### Bảng phân bổ chân kết nối Module BLE vào STM32F407:
| Chân Module BLE | Chân STM32F407 | Chức năng tín hiệu | Lưu ý an toàn |
| :---: | :---: | :--- | :--- |
| `VCC` | `3V3` (hoặc `5V`) | Nguồn nuôi module BLE | Hầu hết module có sẵn IC hạ áp 3.3V |
| `GND` | `GND` | Mass chung hệ thống | Bắt buộc chung mass |
| `TXD` | `PB11` (USART3_RX) | Dữ liệu không dây BLE gửi vào STM32 | Mức logic 3.3V |
| `RXD` | `PB10` (USART3_TX) | Dữ liệu từ STM32 gửi ra module BLE | Mức logic 3.3V |
| `STATE` / `LED` | `PE6` | Tín hiệu báo trạng thái kết nối Bluetooth | Logic 1 khi điện thoại đã kết nối |

```
       STM32F407                       Module BLE HC-08
     +-----------+                     +---------------+
     |      3.3V |---------------------| VCC           |
     |       GND |---------------------| GND           |
     | PB10 (TX) |---------------------| RXD           |
     | PB11 (RX) |---------------------| TXD           |
     |       PE6 |<--------------------| STATE         |
     +-----------+                     +---------------+
```

---

## 4. CẤU HÌNH TRÊN STM32CubeMX
1. **Clock Tree:** SYSCLK 168MHz, APB1 42MHz.
2. **Cấu hình USART3 (Kênh giao tiếp BLE):**
   - Vào *Connectivity* $\rightarrow$ *USART3*.
   - Mode: **Asynchronous**.
   - Parameter Settings:
     - Baud Rate: **9600 Bits/s** (Tốc độ mặc định của hầu hết module HC-08/HM-10 khi xuất xưởng; có thể nâng lên 115200 bằng lệnh AT).
     - Word Length: **8 Bits**.
     - Parity: **None**.
     - Stop Bits: **1**.
   - Vào tab *NVIC Settings* của USART3: **Tích chọn USART3 global interrupt** (Bật ngắt nhận ký tự không chặn CPU).
3. **Cấu hình Chân Báo Trạng thái Kết nối (`PE6`):**
   - Cấu hình `PE6` là GPIO_Input, Pull-down.

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### 5.1. Khởi tạo ngắt nhận chuỗi lệnh BLE
Trong file `Core/Src/main.c`:

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>
#include <string.h>
#include <stdbool.h>

extern UART_HandleTypeDef huart3;
extern UART_HandleTypeDef huart2; /* Console PC */

#define RX_BUFFER_SIZE 32
static uint8_t rxByte;
static char bleRxBuffer[RX_BUFFER_SIZE];
static uint8_t bleRxIndex = 0;
static volatile bool isCommandReceived = false;

/* Bắt đầu kích hoạt nhận 1 byte qua ngắt USART3 */
void BLE_Init(void)
{
    HAL_UART_Receive_IT(&huart3, &rxByte, 1);
}

/* Callback ngắt nhận UART */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART3)
    {
        /* Bắt ký tự kết thúc dòng (\r hoặc \n) */
        if (rxByte == '\r' || rxByte == '\n')
        {
            if (bleRxIndex > 0)
            {
                bleRxBuffer[bleRxIndex] = '\0';
                isCommandReceived = true;
            }
        }
        else
        {
            if (bleRxIndex < (RX_BUFFER_SIZE - 1))
            {
                bleRxBuffer[bleRxIndex++] = (char)rxByte;
            }
        }
        /* Tiếp tục kích hoạt nhận byte kế tiếp */
        HAL_UART_Receive_IT(&huart3, &rxByte, 1);
    }
}

/* Gửi chuỗi phản hồi không dây lên Smartphone */
void BLE_SendString(const char *str)
{
    HAL_UART_Transmit(&huart3, (uint8_t *)str, strlen(str), 100);
}
/* USER CODE END 0 */
```

### 5.2. Máy phân tích lệnh điều khiển không dây (Ex 10.1 $\div$ 10.4)
```c
/* USER CODE BEGIN 2 */
BLE_Init();
printf("BLE HC-08 USART3 READY! Cho ket noi tu Smartphone...\r\n");
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    /* Kiểm tra có gói lệnh mới nhận được từ điện thoại hay không */
    if (isCommandReceived)
    {
        isCommandReceived = false;
        printf("[BLE RX]: %s\r\n", bleRxBuffer);

        if (strcmp(bleRxBuffer, "OPEN") == 0 || strcmp(bleRxBuffer, "O") == 0)
        {
            /* Kích hoạt Relay mở cổng (Bài Lab 9) */
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
            /* Dừng động cơ */
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET);
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_RESET);
            BLE_SendString("ACK: DUNG CONG TUC THOI!\r\n");
        }
        else if (strcmp(bleRxBuffer, "GET_TEMP") == 0)
        {
            /* Gửi nhiệt độ thực tế về Smartphone */
            char tempMsg[32];
            float currentTemp = 29.8f;
            snprintf(tempMsg, sizeof(tempMsg), "TEMP: %.1f C\r\n", currentTemp);
            BLE_SendString(tempMsg);
        }
        else
        {
            BLE_SendString("ERR: Lenh khong hop le (OPEN/CLOSE/STOP/GET_TEMP)\r\n");
        }

        bleRxIndex = 0; /* Xóa bộ đệm */
    }

    /* Định kỳ mỗi 2000ms gửi thông báo trạng thái lên điện thoại */
    static uint32_t lastBleTelemetry = 0;
    if (HAL_GetTick() - lastBleTelemetry >= 2000)
    {
        lastBleTelemetry = HAL_GetTick();
        /* Kiểm tra chân PE6 xem điện thoại đã kết nối chưa */
        if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_6) == GPIO_PIN_SET)
        {
            BLE_SendString("[HEARTBEAT] STM32F407 Smart Home OK\r\n");
        }
    }
}
/* USER CODE END WHILE */
```

---

## 6. THỰC NGHIỆM KẾT NỐI VÀ ĐIỀU KHIỂN TRÊN SMARTPHONE

### 6.1. Hướng dẫn thiết lập trên ứng dụng điện thoại
1. Bật tính năng **Bluetooth** và **Vị trí (GPS)** trên điện thoại thông minh (bắt buộc theo quy định quét BLE của Android/iOS).
2. Khởi động ứng dụng **Serial Bluetooth Terminal**:
   - Vào menu $\rightarrow$ *Devices* $\rightarrow$ Chọn tab **Bluetooth LE**.
   - Nhấn nút **Scan** $\rightarrow$ Tìm tên thiết bị (thường là `HC-08` hoặc `HMSoft`).
   - Nhấn chọn thiết bị để tiến hành kết nối (Pairing).
3. Sau khi kết nối thành công:
   - Đèn LED trên module BLE sẽ chuyển từ trạng thái **nhấp nháy** sang **sáng liên tục**.
   - Chân `PE6` trên STM32F407 được kéo lên mức $3.3\text{V}$.
   - Màn hình điện thoại hiển thị thông báo `Connected`.

### 6.2. Kiểm thử kịch bản điều khiển không dây
1. Gõ lệnh `OPEN` trên điện thoại và nhấn Send $\rightarrow$ Quan sát Relay 1 bật sáng, động cơ quay thuận mở cổng.
2. Gõ lệnh `CLOSE` $\rightarrow$ Quan sát Relay 2 bật sáng, động cơ quay ngược đóng cổng.
3. Gõ lệnh `GET_TEMP` $\rightarrow$ Quan sát màn hình điện thoại nhận về bản tin `TEMP: 29.8 C`.

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Ảnh chụp màn hình ứng dụng điện thoại Smartphone trao đổi lệnh và nhận dữ liệu telemetry từ kit STM32F407.
- Bảng tổng hợp các tập lệnh AT cấu hình module BLE đã thử nghiệm.

### 7.2. Câu hỏi đánh giá
1. *Sự khác nhau cơ bản giữa Bluetooth truyền thống (Classic Bluetooth 2.1 EDR) và Bluetooth Năng lượng thấp (BLE 4.0/5.0) về mặt thông lượng dữ liệu và mức tiêu thụ điện năng là gì?*
2. *Tại sao khi lập trình giao tiếp với module BLE, ta bắt buộc phải sử dụng cơ chế ngắt UART (`HAL_UART_Receive_IT`) thay vì cơ chế chờ thăm dò vòng lặp (`HAL_UART_Receive` blocking)?*
3. *Đặc tính GATT Characteristic trong BLE có những quyền hạn (Properties) nào? (Ví dụ: Read, Write, Notify, Indicate)?*