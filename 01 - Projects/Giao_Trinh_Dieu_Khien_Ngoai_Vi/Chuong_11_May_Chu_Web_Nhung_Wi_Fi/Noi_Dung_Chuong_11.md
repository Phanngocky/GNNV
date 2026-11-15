# CHƯƠNG 11: MÁY CHỦ WEB NHÚNG QUA KẾT NỐI WI-FI TRÊN STM32F407
*(Biên soạn dựa trên Chapter 11: Embedded Web Server over a Wi-Fi Connection - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 11.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã thành thạo truyền thông nối tiếp và không dây tầm ngắn ở các chương trước; chương này mở rộng hệ thống nhúng lên môi trường mạng Internet và **Internet of Things (IoT)**: Biến vi điều khiển STM32F407 thành một **Máy chủ Web nhúng (Embedded Web Server)** có thể truy cập từ bất kỳ trình duyệt Web nào trên máy tính hoặc điện thoại thông qua kết nối Wi-Fi.
- **Mục tiêu chi tiết:**
  1. Nắm vững kiến trúc mạng máy tính cho hệ thống nhúng: Chồng giao thức TCP/IP, địa chỉ IP tĩnh/động DHCP, cổng Socket (Port 80 HTTP) và định dạng bản tin siêu văn bản (HTTP GET Request / HTTP 200 OK Response).
  2. Ghép nối module Wi-Fi ESP8266 (ESP-01 / ESP-12F) với STM32F407 qua cổng `USART6` (`PC6` - TX, `PC7` - RX) bằng tập lệnh chuẩn AT Commands.
  3. Hiện thực hóa các ví dụ từ Example 11.1 đến 11.4 trên STM32F407: Kiểm tra kết nối vi mạch Wi-Fi, tự động kết nối vào điểm truy cập Wi-Fi gia đình (Access Point), khởi tạo máy chủ TCP cổng 80 và phục vụ trang Web Dashboard động hiển thị thời gian thực thông số nhiệt độ, trạng thái rò rỉ khí gas và nút bấm điều khiển thiết bị.
  4. Phân tích nguyên lý bắt tay 3 bước của TCP (3-Way Handshake) và nghiên cứu Case Study quan trắc môi trường công nghiệp.

---

## 11.2. PHÂN BỔ PHẦN CỨNG MODULE WI-FI TRÊN STM32F407

```mermaid
flowchart LR
    Browser["Trình duyệt Web (PC / Smartphone)"] <-->|Sóng Wi-Fi 802.11 b/g/n| Router["Bộ phát Wi-Fi (Router / AP)"]
    Router <-->|Wi-Fi Station Mode| ESP8266["Module Wi-Fi ESP8266"]
    ESP8266 -->|TX Module nối RX MCU| PC7["Chân PC7 (USART6_RX)"]
    ESP8266 <--|RX Module nối TX MCU| PC6["Chân PC6 (USART6_TX)"]
    STM32["STM32F407 Web Server"]
```

- **Xung nhịp bus:** `USART6` nằm trên bus ngoại vi tốc độ cao APB2 ($f_{PCLK2} = 84\text{ MHz}$), cấu hình tốc độ Baud chuẩn của ESP8266 là $115200\text{ bps}$.

---

## 11.3. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 11.1 ĐẾN 11.4 TRÊN STM32F407

### VÍ DỤ 11.1 & 11.2: KHỞI TẠO VÀ KẾT NỐI ĐIỂM TRUY CẬP WI-FI (Listings 11.1 & 11.2)
- STM32F407 giao tiếp với ESP8266 thông qua chuỗi tập lệnh AT:
  1. `AT`: Kiểm tra module có phản hồi `OK`.
  2. `AT+CWMODE=1`: Thiết lập chế độ Station (thiết bị trạm kết nối vào mạng có sẵn).
  3. `AT+CWJAP="Ten_Wifi","Mat_Khau"`: Đăng nhập vào mạng Wi-Fi gia đình.
  4. `AT+CIFSR`: Đọc địa chỉ IP vừa được cấp (ví dụ `192.168.1.105`).

```c
#include "main.h"
#include <string.h>

UART_HandleTypeDef huart6;

bool ESP8266_SendCommand(char *cmd, char *expectedResponse, uint32_t timeout)
{
    HAL_UART_Transmit(&huart6, (uint8_t*)cmd, strlen(cmd), 100);

    char rxBuffer[256];
    memset(rxBuffer, 0, sizeof(rxBuffer));
    uint32_t start = HAL_GetTick();
    uint16_t idx = 0;

    while (HAL_GetTick() - start < timeout)
    {
        uint8_t ch;
        if (HAL_UART_Receive(&huart6, &ch, 1, 10) == HAL_OK)
        {
            if (idx < sizeof(rxBuffer) - 1) rxBuffer[idx++] = ch;
            if (strstr(rxBuffer, expectedResponse)) return true; // Thành công
        }
    }
    return false; // Hết thời gian chờ (Timeout)
}

void ESP8266_Init_WebServer(void)
{
    ESP8266_SendCommand("AT\r\n", "OK", 1000);
    ESP8266_SendCommand("AT+CWMODE=1\r\n", "OK", 1000);
    ESP8266_SendCommand("AT+CWJAP=\"SmartHome_AP\",\"12345678\"\r\n", "OK", 10000);
    ESP8266_SendCommand("AT+CIPMUX=1\r\n", "OK", 1000);       // Bật chế độ đa kết nối
    ESP8266_SendCommand("AT+CIPSERVER=1,80\r\n", "OK", 2000); // Khởi tạo Web Server Port 80
}
```

---

### VÍ DỤ 11.3 & 11.4: PHỤC VỤ TRANG WEB DASHBOARD ĐỘNG (Listings 11.3 & 11.4)
- Khi người dùng gõ địa chỉ IP `http://192.168.1.105` trên trình duyệt điện thoại, ESP8266 gửi chuỗi yêu cầu `+IPD,0,...: GET / HTTP/1.1`.
- STM32F407 nhận dạng yêu cầu, chèn giá trị nhiệt độ và khí gas thực tế vào mã nguồn HTML và gửi trả lại trình duyệt:

```c
void WebServer_Process_Requests(float currentTemp, bool gasDetected, bool alarmState)
{
    char rxPkt[256];
    // Nếu có bản tin HTTP GET từ trình duyệt gửi tới kênh 0 (+IPD,0,...)
    if (ESP8266_CheckIncomingConnection(rxPkt, sizeof(rxPkt)))
    {
        // 1. Tạo nội dung trang HTML động
        char htmlBody[512];
        snprintf(htmlBody, sizeof(htmlBody),
                 "<html><head><meta charset='UTF-8'><meta http-equiv='refresh' content='2'></head>"
                 "<body style='font-family:Arial;text-align:center;background:#eef2f5;'>"
                 "<h1>SMART HOME DASHBOARD</h1>"
                 "<p>Temperature: <b style='font-size:24px;'>%.1f &deg;C</b></p>"
                 "<p>Gas Status: <b style='color:%s;'>%s</b></p>"
                 "<p>Alarm: <b style='color:%s;'>%s</b></p>"
                 "</body></html>",
                 currentTemp,
                 gasDetected ? "red" : "green", gasDetected ? "DETECTED!" : "NORMAL",
                 alarmState ? "red" : "green", alarmState ? "ACTIVE!" : "OFF");

        // 2. Tạo Header HTTP chuẩn
        char httpResponse[768];
        snprintf(httpResponse, sizeof(httpResponse),
                 "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\nContent-Length: %d\r\n\r\n%s",
                 (int)strlen(htmlBody), htmlBody);

        // 3. Gửi lệnh AT+CIPSEND=0,<len> và truyền dữ liệu
        char sendCmd[32];
        snprintf(sendCmd, sizeof(sendCmd), "AT+CIPSEND=0,%d\r\n", (int)strlen(httpResponse));
        HAL_UART_Transmit(&huart6, (uint8_t*)sendCmd, strlen(sendCmd), 100);
        HAL_Delay(50);
        HAL_UART_Transmit(&huart6, (uint8_t*)httpResponse, strlen(httpResponse), 500);

        // 4. Đóng kết nối
        HAL_Delay(100);
        HAL_UART_Transmit(&huart6, (uint8_t*)"AT+CIPCLOSE=0\r\n", 15, 100);
    }
}
```

---

## 11.4. "UNDER THE HOOD" & CASE STUDY (THEO SÁCH ARM)
- **Under the Hood (11.3.1):** 
  - Mô hình mạng TCP/IP và quá trình thiết lập kết nối tin cậy bằng cái bắt tay 3 bước (SYN $\rightarrow$ SYN-ACK $\rightarrow$ ACK).
  - So sánh giao thức TCP (hướng kết nối, truyền tin cậy tuyệt đối, tự động truyền lại gói tin mất) và giao thức UDP (phi kết nối, tốc độ nhanh, phù hợp truyền âm thanh/hình ảnh trực tiếp).
- **Case Study (11.4.1):** Hệ thống Giám sát Môi trường Không khí trong Tòa nhà (Indoor Environment Monitoring): Đo nhiệt độ, độ ẩm, nồng độ bụi mịn PM2.5 và phát trang Web nhúng cho phép người quản lý tòa nhà giám sát chất lượng không khí trên mạng nội bộ LAN.
