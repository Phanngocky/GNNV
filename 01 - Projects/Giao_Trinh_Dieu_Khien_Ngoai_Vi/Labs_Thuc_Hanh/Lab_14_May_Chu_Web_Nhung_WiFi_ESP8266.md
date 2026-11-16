# BÀI THỰC HÀNH SỐ 14: MÁY CHỦ WEB NHÚNG (EMBEDDED WEB SERVER) QUA KẾT NỐI WI-FI VÀ GIAO THỨC HTTP
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết TCP/IP & HTTP $\rightarrow$ Điều khiển tập lệnh AT $\rightarrow$ Lập trình Web Server $\rightarrow$ Kiểm thử Dashboard)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Mô hình TCP/IP Socket & HTTP), CLO 2 (Mạch Wi-Fi ESP8266 & USART6), CLO 3 (Lập trình Web Server), CLO 5 (Kiểm thử mạng LAN)  
**Căn cứ giáo trình:** *ARM Education - Chương 11: Wi-Fi and Embedded Web Servers (Ex 11.1 $\div$ 11.4)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Chồng Giao thức Mạng TCP/IP và Giao thức HTTP:**
   - Phân tích mô hình kiến trúc mạng Internet nhúng: Tầng liên kết dữ liệu Wi-Fi 802.11b/g/n, Tầng mạng IP, Tầng giao vận TCP (Cổng Socket Port 80), và Tầng ứng dụng HTTP (HyperText Transfer Protocol).
   - Nắm vững cấu trúc bản tin **HTTP GET Request** gửi từ trình duyệt web của máy tính/điện thoại:
     `GET / HTTP/1.1\r\nHost: 192.168.1.50\r\n...`
   - Nắm vững định dạng bản tin phản hồi **HTTP 200 OK Response** do vi điều khiển sinh ra:
     Mã trạng thái HTTP Header $\rightarrow$ Dòng trống `\r\n\r\n` $\rightarrow$ Thân mã nguồn siêu văn bản HTML (HTML Payload).
2. **Kỹ thuật Lập trình:**
   - Cấu hình khối `USART6` (`PC6` TX, `PC7` RX) trên STM32F407 giao tiếp với SoC Wi-Fi ESP8266 ở tốc độ $115200\text{ bps}$.
   - Lập trình máy trạng thái điều khiển ESP8266 bằng **Tập lệnh AT chuẩn**:
     - `AT+RST`: Khởi động lại vi mạch.
     - `AT+CWMODE=1`: Chế độ Trạm kết nối (Station Mode).
     - `AT+CWJAP="SSID","Password"`: Tự động kết nối vào Router Wi-Fi phòng lab.
     - `AT+CIFSR`: Lấy địa chỉ IP cục bộ được cấp phát qua DHCP.
     - `AT+CIPMUX=1`: Kích hoạt chế độ đa kết nối TCP (Multiple Connections).
     - `AT+CIPSERVER=1,80`: Khởi động TCP Web Server lắng nghe cổng 80.
   - Xây dựng trang giao diện **Web Dashboard Giám sát & Điều khiển**: Hiển thị nhiệt độ phòng thời gian thực và cung cấp các nút bấm bật/tắt đèn chiếu sáng, đóng/mở cổng ngay trên trình duyệt web bất kỳ (Ex 11.1 $\div$ 11.4).
3. **Đo kiểm & Đánh giá Hiệu năng:**
   - Đo kiểm tính ổn định của kết nối TCP và thời gian xử lý yêu cầu HTTP Request.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Bộ xử lý trung tâm |
| 2 | Module Wi-Fi ESP8266 | ESP-01 hoặc ESP-12E/F kèm đế cắm | 1 | Nạp sẵn firmware AT chuẩn (115200) |
| 3 | Mạch nguồn hạ áp độc lập | Nguồn 3.3V / 1A (ASM1117-3.3) | 1 | Cấp nguồn riêng cho ESP8266 |
| 4 | Tụ hóa & Tụ gốm | Tụ $100\mu\text{F}$ và $100\text{nF}$ mắc song song nguồn ESP | 1 cặp | Chống sụt áp khi phát sóng RF |
| 5 | Máy tính cá nhân / Điện thoại | Trình duyệt Google Chrome / Safari | 1 | Truy cập Dashboard |

---

## 3. THIẾT KẾ MẠCH NGUYÊN LÝ VÀ ĐẤU NỐI CHÂN (PINOUT)

### 3.1. Lưu ý sống còn về nguồn cấp cho ESP8266
Khi phát sóng Wi-Fi công suất cao ở dải $2.4\text{ GHz}$, vi mạch ESP8266 có thể tiêu thụ dòng đỉnh (Peak Current) lên tới **$200 \div 300\text{mA}$** trong vài micro-giây. Chân cấp nguồn $3.3\text{V}$ trên các mạch nạp ST-Link thông thường chỉ cung cấp tối đa $100\text{mA}$, dẫn đến hiện tượng ESP8266 liên tục bị Reset do sụt nguồn (Brown-out reset).  
$\implies$ **Bắt buộc sử dụng IC nguồn hạ áp riêng biệt AMS1117-3.3V cấp riêng cho ESP8266 kèm tụ lọc $100\mu\text{F}$!**

### 3.2. Bảng phân bổ chân kết nối Module ESP8266 vào STM32F407:
| Chân Module ESP8266 | Chân STM32F407 | Chức năng phần cứng | Lưu ý kỹ thuật |
| :---: | :---: | :--- | :--- |
| `VCC` | Nguồn $3.3\text{V}_{EXT}$ | Nguồn nuôi dương $3.3\text{V}$ | Từ mạch nguồn riêng AMS1117 |
| `GND` | Mass chung | Mass hệ thống | **Nối chung Mass STM32 và Mass Nguồn ngoài** |
| `TXD` | `PC7` (USART6_RX) | ESP8266 truyền phản hồi AT vào STM32 | Mức logic 3.3V |
| `RXD` | `PC6` (USART6_TX) | STM32 gửi lệnh AT xuống ESP8266 | Mức logic 3.3V |
| `CH_PD` / `EN` | Nguồn $3.3\text{V}_{EXT}$ | Chân kích hoạt vi mạch (Chip Enable) | Kéo lên mức cao $3.3\text{V}$ |
| `RST` | `PC13` (hoặc treo 3.3V) | Reset phần cứng ESP8266 | STM32 điều khiển Reset tự động |

---

## 4. QUY TRÌNH CẤU HÌNH TRÊN STM32CubeMX
1. **Clock Configuration:** Cấu hình chuẩn SYSCLK 168MHz, APB2 84MHz.
2. **Cấu hình USART6 (Khối giao tiếp Wi-Fi):**
   - Vào *Connectivity* $\rightarrow$ *USART6*.
   - Mode: **Asynchronous**.
   - Parameter Settings:
     - Baud Rate: **115200 Bits/s**.
     - Word Length: **8 Bits**.
     - Parity: **None**.
     - Stop Bits: **1**.
   - Vào tab *NVIC Settings* của USART6: **Tích chọn USART6 global interrupt** với Preemption Priority = **1**.
3. **Cấu hình Chân Reset ESP8266 (`PC13`):**
   - Cấu hình GPIO_Output Push-Pull, High-Speed.

---

## 5. MÃ NGUỒN DRIVER ESP8266 VÀ MÁY CHỦ WEB EMBEDDED

### 5.1. Thư viện điều khiển Tập lệnh AT ESP8266
Tạo file `Core/Src/esp8266_web.c`:

```c
#include "stm32f4xx_hal.h"
#include <stdio.h>
#include <string.h>
#include <stdbool.h>

extern UART_HandleTypeDef huart6;
extern UART_HandleTypeDef huart2; /* Console PC */

#define ESP_RX_BUF_SIZE 1024
static char espRxBuffer[ESP_RX_BUF_SIZE];
static uint16_t espRxIndex = 0;
static uint8_t rxChar;

void ESP_Init(void)
{
    /* Bắt đầu nhận ký tự qua ngắt USART6 */
    HAL_UART_Receive_IT(&huart6, &rxChar, 1);
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART6)
    {
        if (espRxIndex < (ESP_RX_BUF_SIZE - 1)) {
            espRxBuffer[espRxIndex++] = (char)rxChar;
        }
        HAL_UART_Receive_IT(&huart6, &rxChar, 1);
    }
}

/* Xóa bộ đệm nhận */
void ESP_ClearBuffer(void)
{
    memset(espRxBuffer, 0, ESP_RX_BUF_SIZE);
    espRxIndex = 0;
}

/* Gửi lệnh AT và chờ chuỗi phản hồi mong muốn trong khoảng thời gian timeout */
bool ESP_SendCommand(const char *cmd, const char *expectedResponse, uint32_t timeoutMs)
{
    ESP_ClearBuffer();
    HAL_UART_Transmit(&huart6, (uint8_t *)cmd, strlen(cmd), 100);
    HAL_UART_Transmit(&huart6, (uint8_t *)"\r\n", 2, 100);

    uint32_t start = HAL_GetTick();
    while (HAL_GetTick() - start < timeoutMs)
    {
        if (strstr(espRxBuffer, expectedResponse) != NULL) {
            return true;
        }
        HAL_Delay(10);
    }
    return false;
}
```

### 5.2. Khởi tạo TCP Web Server trên Cổng 80
```c
bool ESP_StartWebServer(const char *wifiSSID, const char *wifiPassword)
{
    char atCommand[128];

    printf("[WIFI] Resetting ESP8266...\r\n");
    ESP_SendCommand("AT+RST", "ready", 3000);
    HAL_Delay(1000);

    printf("[WIFI] Setting Station Mode...\r\n");
    if (!ESP_SendCommand("AT+CWMODE=1", "OK", 1000)) return false;

    printf("[WIFI] Connecting to SSID: %s...\r\n", wifiSSID);
    snprintf(atCommand, sizeof(atCommand), "AT+CWJAP=\"%s\",\"%s\"", wifiSSID, wifiPassword);
    if (!ESP_SendCommand(atCommand, "WIFI CONNECTED", 15000)) {
        printf("[WIFI ERROR] Khong the ket noi vao Wi-Fi!\r\n");
        return false;
    }

    printf("[WIFI] Enabling Multi-connections...\r\n");
    if (!ESP_SendCommand("AT+CIPMUX=1", "OK", 1000)) return false;

    printf("[WIFI] Starting TCP Server on Port 80...\r\n");
    if (!ESP_SendCommand("AT+CIPSERVER=1,80", "OK", 1000)) return false;

    /* Lấy IP được cấp */
    ESP_ClearBuffer();
    HAL_UART_Transmit(&huart6, (uint8_t *)"AT+CIFSR\r\n", 10, 100);
    HAL_Delay(500);
    printf("[WIFI] WEB SERVER DANG CHAY! IP INFO:\r\n%s\r\n", espRxBuffer);

    return true;
}
```

### 5.3. Xử lý yêu cầu HTTP GET và Phục vụ Trang Dashboard HTML
```c
/* Gửi chuỗi phản hồi HTTP Response về cho trình duyệt Web */
void ESP_SendHttpResponse(uint8_t linkId, const char *htmlContent)
{
    char headerBuffer[256];
    char sendCmd[32];
    int contentLength = strlen(htmlContent);

    /* Soạn thảo HTTP 200 OK Header */
    snprintf(headerBuffer, sizeof(headerBuffer),
             "HTTP/1.1 200 OK\r\n"
             "Content-Type: text/html\r\n"
             "Content-Length: %d\r\n"
             "Connection: close\r\n\r\n", contentLength);

    int totalLength = strlen(headerBuffer) + contentLength;

    /* Gửi lệnh báo trước độ dài gói dữ liệu AT+CIPSEND=<linkId>,<len> */
    snprintf(sendCmd, sizeof(sendCmd), "AT+CIPSEND=%d,%d", linkId, totalLength);
    if (ESP_SendCommand(sendCmd, ">", 2000))
    {
        /* Đẩy dữ liệu Header và HTML Body */
        HAL_UART_Transmit(&huart6, (uint8_t *)headerBuffer, strlen(headerBuffer), 500);
        HAL_UART_Transmit(&huart6, (uint8_t *)htmlContent, contentLength, 1000);
        HAL_Delay(100);
    }

    /* Đóng kết nối TCP */
    char closeCmd[20];
    snprintf(closeCmd, sizeof(closeCmd), "AT+CIPCLOSE=%d", linkId);
    ESP_SendCommand(closeCmd, "OK", 1000);
}

/* Xử lý các yêu cầu HTTP trong vòng lặp chính */
void WebServer_ProcessRequests(void)
{
    /* Kiểm tra gói dữ liệu nhận được dạng: +IPD,<linkId>,<len>:GET /... */
    char *ipdPtr = strstr(espRxBuffer, "+IPD,");
    if (ipdPtr != NULL)
    {
        uint8_t linkId = ipdPtr[5] - '0'; /* Lấy chỉ số liên kết kết nối */

        /* Kiểm tra URL người dùng yêu cầu */
        if (strstr(ipdPtr, "GET /led_on") != NULL) {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET); /* Bật LED */
        }
        else if (strstr(ipdPtr, "GET /led_off") != NULL) {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET); /* Tắt LED */
        }

        /* Tạo mã nguồn giao diện HTML Dashboard hiện đại */
        char htmlPage[1024];
        float tempVal = 28.5f; /* Lấy từ DS18B20 */

        snprintf(htmlPage, sizeof(htmlPage),
                 "<!DOCTYPE html><html>"
                 "<head><title>STM32F407 Smart Home</title><meta name='viewport' content='width=device-width, initial-scale=1'>"
                 "<style>body{font-family:Arial;text-align:center;background:#f0f2f5;padding:20px;}"
                 ".card{background:white;padding:25px;border-radius:12px;box-shadow:0 4px 10px rgba(0,0,0,0.1);max-width:400px;margin:auto;}"
                 ".btn{padding:12px 24px;font-size:16px;margin:10px;border-radius:8px;border:none;cursor:pointer;text-decoration:none;display:inline-block;}"
                 ".btn-on{background:#28a745;color:white;} .btn-off{background:#dc3545;color:white;}"
                 "</style></head>"
                 "<body><div class='card'>"
                 "<h2>STM32F407 DASHBOARD</h2>"
                 "<p>Nhiet do phong: <b style='color:#007bff;font-size:24px;'>%.1f &deg;C</b></p>"
                 "<hr>"
                 "<p>Dieu khien thiet bi den phong khach:</p>"
                 "<a href='/led_on' class='btn btn-on'>BAT DEN</a>"
                 "<a href='/led_off' class='btn btn-off'>TAT DEN</a>"
                 "</div></body></html>", tempVal);

        ESP_SendHttpResponse(linkId, htmlPage);
        ESP_ClearBuffer();
    }
}
```

---

## 6. THỰC NGHIỆM TRUY CẬP VÀ ĐIỀU KHIỂN QUA TRÌNH DUYỆT WEB

### 6.1. Trình tự thử nghiệm
1. Kết nối máy tính hoặc điện thoại vào cùng một mạng Wi-Fi của phòng thí nghiệm.
2. Mở cửa sổ Console Serial PC (115200 bps): Quan sát dòng thông báo địa chỉ IP được cấp phát cho STM32 (Ví dụ: `192.168.1.105`).
3. Mở trình duyệt Web (Google Chrome) trên máy tính hoặc điện thoại, gõ thanh địa chỉ:
   `http://192.168.1.105`
4. Giao diện Web Dashboard màu sắc trang nhã hiện ra với số liệu nhiệt độ thực tế.
5. Nhấn nút **BẬT ĐÈN**: Trình duyệt gửi lệnh `GET /led_on`, đèn LED `PD12` trên bo STM32 lập tức bật sáng!
6. Nhấn nút **TẮT ĐÈN**: Đèn LED tắt ngay tức khắc.

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Ảnh chụp màn hình trình duyệt Web trên điện thoại Smartphone đang điều khiển thiết bị qua địa chỉ IP của STM32F407.
- Log giao tiếp AT giữa STM32 và ESP8266 khi một máy khách (Client) kết nối.

### 7.2. Câu hỏi đánh giá
1. *Bản tin HTTP Header có ý nghĩa gì đối với trình duyệt web? Tại sao trường `Content-Length` phải tính toán chính xác số lượng ký tự của thân mã nguồn HTML?*
2. *Hiện tượng "3-Way Handshake" trong giao thức TCP bao gồm 3 bước nào giữa Client và Web Server?*
3. *Sự khác nhau giữa mô hình truyền thông Client-Server (HTTP) và mô hình Xuất bản - Đăng ký (MQTT Publish/Subscribe) trong các hệ thống IoT công nghiệp là gì?*