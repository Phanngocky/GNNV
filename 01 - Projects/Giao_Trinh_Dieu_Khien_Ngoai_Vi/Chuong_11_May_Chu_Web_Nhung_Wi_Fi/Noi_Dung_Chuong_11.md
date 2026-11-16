# CHƯƠNG 11: MÁY CHỦ WEB NHÚNG QUA KẾT NỐI WI-FI VÀ GIAO THỨC HTTP TRÊN STM32F407
*(Biên soạn dựa trên Chapter 11: Embedded Web Server over a Wi-Fi Connection - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 11.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã thành thạo truyền thông nối tiếp và không dây tầm ngắn BLE ở các chương trước; chương này mở rộng hệ thống nhúng lên môi trường mạng Internet và **Internet of Things (IoT)**: Biến vi điều khiển STM32F407 thành một **Máy chủ Web nhúng (Embedded Web Server)** có thể truy cập từ bất kỳ trình duyệt Web nào trên máy tính hoặc điện thoại thông qua kết nối mạng Wi-Fi cục bộ (LAN) hoặc Internet.
- **Mục tiêu chi tiết:**
  1. **Lý thuyết Chồng Giao Thức TCP/IP & HTTP:** Nắm vững cấu trúc 4 tầng mạng (Network Interface 802.11b/g/n, Internet IPv4, Transport TCP Socket Port 80, và Application HTTP/1.1), cơ chế bắt tay 3 bước của TCP (3-Way Handshake), định dạng bản tin **HTTP GET Request** từ trình duyệt và cấu trúc bản tin phản hồi **HTTP 200 OK Response** (Header, `Content-Length`, dòng trống `\r\n\r\n` và thân mã nguồn siêu văn bản HTML).
  2. **Thiết Kế Mạch Nguồn Công Suất Cho Wi-Fi:** Phân tích đặc tính tiêu thụ dòng đỉnh (Peak Current) lên tới $200 \div 300\text{mA}$ của vi mạch ESP8266 khi phát sóng RF; thiết kế mạch nguồn hạ áp độc lập AMS1117-3.3V kèm bộ lọc tụ điện chống sụt áp đột ngột (Brown-out Reset).
  3. **Phân Bổ Phần Cứng & Giao Tiếp USART6:** Ghép nối module Wi-Fi ESP8266 (ESP-01 / ESP-12F) với STM32F407 qua cổng `USART6` (`PC6` TX, `PC7` RX) trên bus APB2 (84MHz) ở tốc độ $115200\text{ bps}$.
  4. **Lập Trình Điều Khiển Bằng Tập Lệnh AT (Examples 11.1 $\div$ 11.4):**
     - Quản lý máy trạng thái cấu hình ESP8266: Khởi động lại, chuyển sang chế độ Station (`CWMODE=1`), tự động đăng nhập Wi-Fi (`CWJAP`), bật chế độ đa kết nối (`CIPMUX=1`) và lắng nghe Socket cổng 80 (`CIPSERVER=1,80`).
     - Lập trình bộ phân tích cú pháp yêu cầu HTTP (HTTP Request Parser): Bắt chuỗi `GET /led_on` để bật đèn, `GET /led_off` để tắt đèn.
     - Soạn thảo và gửi động trang **Web Dashboard Giám Sát & Điều Khiển** hiển thị số liệu nhiệt độ thời gian thực và các nút nhấn điều khiển thiết bị gia dụng.
  5. **Đo Kiểm & Phân Tích Mạng:** Kiểm thử lưu lượng gói tin mạng và chẩn đoán lỗi truyền thông socket.

---

## 11.2. CƠ SỞ LÝ THUYẾT CHUYÊN SÂU

### 11.2.1. Chồng Giao Thức Mạng TCP/IP Cho Hệ Thống Nhúng
Mô hình mạng nhúng chia thành 4 tầng giao thức tiêu chuẩn:

```
+-------------------------------------------------------------------------------+
|                       TẦNG ỨNG DỤNG (APPLICATION LAYER)                       |
|   Giao thức truyền siêu văn bản: HTTP/1.1 (Cổng dịch vụ mặc định: Port 80)     |
+-------------------------------------------------------------------------------+
|                      TẦNG GIAO VẬN (TRANSPORT LAYER)                          |
|   Giao thức TCP (Transmission Control Protocol) tin cậy, hướng kết nối        |
+-------------------------------------------------------------------------------+
|                       TẦNG MẠNG (INTERNET LAYER)                              |
|   Giao thức IPv4 (Định địa chỉ logic dạng: 192.168.1.105) & DHCP             |
+-------------------------------------------------------------------------------+
|                    TẦNG LIÊN KẾT DỮ LIỆU & VẬT LÝ                             |
|   Chuẩn vô tuyến Wi-Fi IEEE 802.11 b/g/n (Băng tần 2.4 GHz, Tốc độ 150Mbps)   |
+-------------------------------------------------------------------------------+
```

#### 1. Cơ chế Bắt tay Ba bước của TCP (TCP 3-Way Handshake):
Trước khi bất kỳ dữ liệu HTTP nào được trao đổi, tầng giao vận TCP phải thiết lập một kênh kết nối logic tin cậy giữa Trình duyệt Web (Client) và STM32F407 (Web Server):
1. **Bước 1 (SYN):** Client gửi một gói tin có cờ đồng bộ `SYN = 1` cùng số thứ tự khởi tạo ngẫu nhiên $Seq = x$ để yêu cầu mở cổng 80.
2. **Bước 2 (SYN-ACK):** Server (ESP8266) chấp nhận, phản hồi gói tin có cờ `SYN = 1`, cờ `ACK = 1`, số xác nhận $Ack = x + 1$ và số thứ tự riêng $Seq = y$.
3. **Bước 3 (ACK):** Client gửi lại gói tin xác nhận `ACK = 1` với $Ack = y + 1$.  
$\implies$ Kết nối TCP được xác lập hoàn tất! Hai bên sẵn sàng truyền tải các bản tin HTTP.

#### 2. Cấu trúc Bản tin Yêu cầu HTTP GET Request:
Khi người dùng truy cập một địa chỉ IP trên trình duyệt web, trình duyệt tự động gửi một gói tin văn bản ASCII có định dạng:
```http
GET /led_on HTTP/1.1\r\n
Host: 192.168.1.105\r\n
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)\r\n
Accept: text/html,application/xhtml+xml\r\n
Connection: keep-alive\r\n
\r\n
```
- Dòng đầu tiên gọi là **Request Line**: Chứa phương thức (`GET`), đường dẫn tài nguyên (`/` hoặc `/led_on`) và phiên bản giao thức (`HTTP/1.1`).
- Kết thúc phần tiêu đề (Header) luôn là một **dòng trống bắt buộc: `\r\n\r\n`**.

#### 3. Cấu trúc Bản tin Phản hồi HTTP 200 OK Response:
Để trình duyệt web hiểu và hiển thị trang giao diện, vi điều khiển STM32F407 bắt buộc phải định dạng bản tin phản hồi gồm 3 phần:
```http
HTTP/1.1 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 345\r\n
Connection: close\r\n
\r\n
<!DOCTYPE html><html><body><h1>STM32F407 DASHBOARD</h1>...</body></html>
```
- **Status Line:** `HTTP/1.1 200 OK` (Mã trạng thái 200 báo hiệu yêu cầu đã được xử lý thành công).
- **Header:** 
  - `Content-Type: text/html`: Báo cho trình duyệt biết dữ liệu đi kèm là mã nguồn siêu văn bản HTML.
  - `Content-Length: <kích_thước>`: **Số lượng bytes chính xác của thân mã nguồn HTML**. Nếu tính sai trường này, trình duyệt sẽ bị treo chờ tải tiếp hoặc cắt cụt trang web!
  - `Connection: close`: Yêu cầu đóng socket sau khi truyền xong.
- **Dòng trống phân cách:** `\r\n\r\n`.
- **Thân mã nguồn (Payload Body):** Toàn bộ thẻ HTML, CSS, JavaScript hiển thị giao diện.

---

### 11.2.2. Vấn Đề Nguồn Điện Công Suất Của Vi Mạch Wi-Fi ESP8266
SoC Wi-Fi ESP8266 tích hợp một bộ khuếch đại công suất cao tần công suất phát sóng lên tới $+19.5\text{dBm}$ ($90\text{mW}$).
- Ở trạng thái bình thường (kết nối rỗi): Dòng tiêu thụ dao động từ $60 \div 80\text{mA}$.
- **Khi phát sóng RF công suất cao (TX Burst):** Dòng tiêu thụ tăng vọt thành các gai nhọn (Peak Current) lên tới **$200 \div 300\text{mA}$** trong khoảng thời gian vài micro-giây.
- Các mạch nạp ST-Link thông thường hoặc chân cấp nguồn $3.3\text{V}$ trên kit phát triển STM32 chỉ được thiết kế cho vi điều khiển, cấp dòng tối đa $< 100\text{mA}$.
- Nếu cấp nguồn ESP8266 chung với chân 3.3V của STM32, điện áp nguồn sẽ bị sụt áp đột ngột xuống dưới $2.7\text{V}$, kích hoạt mạch giám sát điện áp thấp bên trong vi điều khiển và ESP8266 gây ra hiện tượng: **ESP8266 liên tục khởi động lại (Brown-out Reset Loop) và làm vi điều khiển bị treo!**
- $\implies$ **Giải pháp bắt buộc:** Sử dụng một IC nguồn tuyến tính riêng biệt **AMS1117-3.3V** (dòng tải định mức $1\text{A}$) cấp riêng cho ESP8266, mắc song song một tụ hóa $100\mu\text{F}$ (tích trữ năng lượng cho dòng đỉnh) và một tụ gốm $100\text{nF}$ (triệt tiêu gai nhiễu cao tần).

---

### 11.2.3. Tập Lệnh Cấu Hình AT Commands Cho ESP8266 Trong Chế Độ Web Server

| Tập lệnh AT | Ý nghĩa kỹ thuật | Phản hồi từ Module |
| :--- | :--- | :--- |
| `AT+RST` | Khởi động lại vi mạch ESP8266 bằng phần mềm | `ready` |
| `AT+CWMODE=1` | Cấu hình chế độ Trạm kết nối (Station Mode) | `OK` |
| `AT+CWJAP="SSID","PASS"` | Đăng nhập vào Router Wi-Fi của phòng Lab / Nhà ở | `WIFI CONNECTED` $\rightarrow$ `WIFI GOT IP` $\rightarrow$ `OK` |
| `AT+CIFSR` | Đọc địa chỉ IP cục bộ được cấp phát qua DHCP | `+CIFSR:STAIP,"192.168.1.105"` |
| `AT+CIPMUX=1` | Bật chế độ đa kết nối (Hỗ trợ nhiều client cùng truy cập) | `OK` |
| `AT+CIPSERVER=1,80` | Khởi tạo TCP Web Server lắng nghe trên cổng mặc định 80 | `OK` |
| `AT+CIPSEND=<id>,<len>` | Khai báo trước độ dài dữ liệu chuẩn bị gửi ra kênh `<id>` | Ký tự nhắc lệnh `>` |
| `AT+CIPCLOSE=<id>` | Đóng kết nối TCP sau khi phục vụ xong trang web | `<id>,CLOSED` |

---

## 11.3. SƠ ĐỒ NGUYÊN LÝ & SƠ ĐỒ ĐẤU NỐI CHÂN CHI TIẾT

### 11.3.1. Sơ đồ mạch nguyên lý điện tử (Schematics)

```
=============================================================================================
SƠ ĐỒ GHÉP NỐI MODULE WI-FI ESP8266 VỚI STM32F407 VÀ MẠCH NGUỒN CÁCH LY
=============================================================================================
Nguồn Ngoài +5VDC           IC Hạ Áp AMS1117-3.3V             Module Wi-Fi ESP8266 (ESP-01)
     |                     +-----------------------+         +----------------------------+
     +---------------------| VIN              VOUT |----+----| VCC                        |
                           |                       |    |    |                            |
     +---[ 100uF ]---------| GND                   |  [===]  | CH_PD / EN (Treo 3.3V)     |
     |   Tụ hóa lọc        +-----------------------+  100uF  |                            |
     |                                                  |    | RST                        |
     |                                                =====  |                            |
     |                                                100nF  |                            |
    GND                                                 |    |                            |
     |                                                 GND   |                            |
     |   STM32F407                                           |                            |
     |  +------------------+                                 |                            |
     +--| GND              |---------------------------------| GND (Nối chung mass)       |
        |                  |                                 |                            |
        |  PC6 (USART6_TX) |---------------------------------| RXD (Nhận lệnh AT)         |
        |                  |                                 |                            |
        |  PC7 (USART6_RX) |<--------------------------------| TXD (Phản hồi dữ liệu AT)  |
        |                  |                                 |                            |
        |  PC13 (GPIO_Out) |---------------------------------| RST (Điều khiển Reset cứng)|
        +------------------+                                 +----------------------------+
```

---

### 11.3.2. Bảng Phân Bổ Chân Toàn Diện Trên STM32F407 (Master Pinout Table)

| Tên Chân STM32 | Tên Tín Hiệu | Chức Năng Phần Cứng / Alternate Function | Cấu Hình Trên STM32CubeMX | Kết Nối Chân Module ESP8266 | Mức Điện Áp & Lưu Ý Kỹ Thuật |
| :---: | :---: | :---: | :---: | :--- | :--- |
| `PC6` | `WIFI_TX` | `USART6_TX` (AF8) | Alternate Function Push-Pull, Very High Speed | Chân `RXD` của module ESP8266 | Mức logic $3.3\text{V}$, tốc độ $115200\text{ bps}$ |
| `PC7` | `WIFI_RX` | `USART6_RX` (AF8) | Alternate Function Push-Pull, Bật **Ngắt USART6** | Chân `TXD` của module ESP8266 | Mức logic $3.3\text{V}$, bộ đệm ngắt vòng (Circular Buffer) |
| `PC13` | `WIFI_RST`| GPIO_Output | Output Push-Pull, High Speed, Mặc định mức 1 | Chân `RST` của module ESP8266 | Mức 0 = Reset cứng, Mức 1 = Hoạt động |
| `PD12` | `LED_GREEN`| GPIO_Output | Output Push-Pull | Đèn LED phòng khách điều khiển qua Web | $3.3\text{V}$ qua trở hạn dòng $330\Omega$ |
| `PD14` | `LED_RED` | GPIO_Output | Output Push-Pull | Đèn báo trạng thái kết nối Wi-Fi | Nhấp nháy khi chưa có IP, Sáng khi Web Server sẵn sàng |
| `PA2` | `USART2_TX`| `USART2_TX` (AF7) | Alternate Function Push-Pull | Cổng nối tiếp in log gỡ lỗi lên máy tính PC | $115200\text{ bps}$, hiển thị địa chỉ IP nhận được |
| `3V3_EXT` | `VCC_WIFI`| Nguồn ngoài $3.3\text{V}$ | Nguồn nuôi độc lập từ AMS1117-3.3 | Chân `VCC` và `CH_PD` của ESP8266 | **Dòng cấp tối thiểu 500mA - 1A** |
| `GND` | `GND` | Mass hệ thống | Mass chung | Nối chung mass STM32, Nguồn ngoài và ESP | Bắt buộc chung mass |

---

## 11.4. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 11.1 ĐẾN 11.4 TRÊN STM32F407

### 11.4.1. Cấu hình STM32CubeMX Cho USART6
1. **Khối USART6:**
   - Nằm trên bus APB2 ($f_{PCLK2} = 84\text{ MHz}$).
   - Mode: **Asynchronous**.
   - Parameter Settings:
     - Baud Rate: **115200 Bits/s**.
     - Word Length: **8 Bits**, Parity: **None**, Stop Bits: **1**.
   - Thẻ *NVIC Settings* của USART6: **Tích chọn USART6 global interrupt** với Preemption Priority = **1**.
2. **Khối GPIO:**
   - Chân `PC13`: GPIO_Output, Initial Level: High.
   - Chân `PD12, PD14`: GPIO_Output.

---

### 11.4.2. Mã Nguồn Hoàn Chỉnh Của Chương

#### 1. Bộ Đệm Vòng Ngắt USART6 và Hàm Giao Tiếp Tập Lệnh AT
```c
#include "main.h"
#include <stdio.h>
#include <string.h>
#include <stdbool.h>

extern UART_HandleTypeDef huart6;
extern UART_HandleTypeDef huart2; /* Console PC */

#define ESP_BUF_SIZE 1500
static char espRxBuffer[ESP_BUF_SIZE];
static uint16_t espRxIndex = 0;
static uint8_t rxChar;

void ESP_Init(void)
{
    /* Kích hoạt nhận ký tự qua ngắt USART6 */
    HAL_UART_Receive_IT(&huart6, &rxChar, 1);
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART6)
    {
        if (espRxIndex < (ESP_BUF_SIZE - 1))
        {
            espRxBuffer[espRxIndex++] = (char)rxChar;
        }
        HAL_UART_Receive_IT(&huart6, &rxChar, 1);
    }
}

void ESP_ClearBuffer(void)
{
    memset(espRxBuffer, 0, ESP_BUF_SIZE);
    espRxIndex = 0;
}

/* Gửi lệnh AT và chờ phản hồi chuỗi mong muốn */
bool ESP_SendCommand(const char *cmd, const char *expectedResponse, uint32_t timeoutMs)
{
    ESP_ClearBuffer();
    HAL_UART_Transmit(&huart6, (uint8_t *)cmd, strlen(cmd), 100);
    HAL_UART_Transmit(&huart6, (uint8_t *)"\r\n", 2, 100);

    uint32_t start = HAL_GetTick();
    while (HAL_GetTick() - start < timeoutMs)
    {
        if (strstr(espRxBuffer, expectedResponse) != NULL)
        {
            return true;
        }
        HAL_Delay(10);
    }
    return false;
}
```

---

#### 2. Khởi Tạo Kết Nối Wi-Fi và Kích Hoạt Web Server Cổng 80 (Examples 11.1 & 11.2)
```c
bool ESP_StartWebServer(const char *ssid, const char *password)
{
    char atCmd[128];

    printf("[WIFI] Resetting module ESP8266...\r\n");
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
    HAL_Delay(50);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
    HAL_Delay(1500);

    /* 1. Kiểm tra giao tiếp */
    if (!ESP_SendCommand("AT", "OK", 1000)) {
        printf("[WIFI ERROR] Module khong phan hoi lenh AT!\r\n");
        return false;
    }

    /* 2. Cài đặt chế độ Station */
    printf("[WIFI] Chuyen sang che do Station Mode (CWMODE=1)...\r\n");
    ESP_SendCommand("AT+CWMODE=1", "OK", 1000);

    /* 3. Đăng nhập vào Router Wi-Fi */
    printf("[WIFI] Dang ket noi vao mang: %s...\r\n", ssid);
    snprintf(atCmd, sizeof(atCmd), "AT+CWJAP=\"%s\",\"%s\"", ssid, password);
    if (!ESP_SendCommand(atCmd, "WIFI CONNECTED", 15000)) {
        printf("[WIFI ERROR] Ket noi Wi-Fi that bai! Kiem tra mat khau.\r\n");
        return false;
    }

    /* 4. Kích hoạt đa kết nối */
    printf("[WIFI] Bat che do da ket noi (CIPMUX=1)...\r\n");
    ESP_SendCommand("AT+CIPMUX=1", "OK", 1000);

    /* 5. Khởi động Web Server cổng 80 */
    printf("[WIFI] Khoi dong TCP Web Server tren Cong 80...\r\n");
    if (!ESP_SendCommand("AT+CIPSERVER=1,80", "OK", 2000)) {
        printf("[WIFI ERROR] Khong the mo Web Server Port 80!\r\n");
        return false;
    }

    /* 6. Lấy địa chỉ IP được cấp */
    ESP_ClearBuffer();
    HAL_UART_Transmit(&huart6, (uint8_t *)"AT+CIFSR\r\n", 10, 100);
    HAL_Delay(500);
    printf("[WIFI SUCCESS] WEB SERVER DANG CHAY TAI DIA CHI IP:\r\n%s\r\n", espRxBuffer);

    HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET); /* Bật LED đỏ báo sẵn sàng */
    return true;
}
```

---

#### 3. Xử Lý Yêu Cầu HTTP GET và Phục Vụ Trang Web Dashboard (Examples 11.3 & 11.4)
```c
/* Gửi bản tin HTTP Response hoàn chỉnh về cho trình duyệt */
void ESP_SendHttpResponse(uint8_t linkId, const char *htmlPayload)
{
    char httpHeader[256];
    char sendCmd[32];
    int bodyLength = strlen(htmlPayload);

    /* Soạn thảo HTTP 200 OK Header chuẩn */
    snprintf(httpHeader, sizeof(httpHeader),
             "HTTP/1.1 200 OK\r\n"
             "Content-Type: text/html\r\n"
             "Content-Length: %d\r\n"
             "Connection: close\r\n\r\n", bodyLength);

    int totalLength = strlen(httpHeader) + bodyLength;

    /* Gửi lệnh báo trước độ dài dữ liệu: AT+CIPSEND=<linkId>,<len> */
    snprintf(sendCmd, sizeof(sendCmd), "AT+CIPSEND=%d,%d", linkId, totalLength);
    if (ESP_SendCommand(sendCmd, ">", 2000))
    {
        /* Đẩy Header và Body siêu văn bản */
        HAL_UART_Transmit(&huart6, (uint8_t *)httpHeader, strlen(httpHeader), 500);
        HAL_UART_Transmit(&huart6, (uint8_t *)htmlPayload, bodyLength, 1000);
        HAL_Delay(100);
    }

    /* Đóng kết nối socket giải phóng kênh */
    char closeCmd[20];
    snprintf(closeCmd, sizeof(closeCmd), "AT+CIPCLOSE=%d", linkId);
    ESP_SendCommand(closeCmd, "OK", 1000);
}

/* Xử lý các yêu cầu HTTP trong siêu vòng lặp chính */
void WebServer_ProcessRequests(float roomTemp)
{
    /* Kiểm tra gói dữ liệu đến dạng: +IPD,<linkId>,<len>:GET /... */
    char *ipdPtr = strstr(espRxBuffer, "+IPD,");
    if (ipdPtr != NULL)
    {
        uint8_t linkId = ipdPtr[5] - '0'; /* Lấy chỉ số liên kết (0 - 4) */

        /* Bộ phân tích cú pháp URL */
        if (strstr(ipdPtr, "GET /led_on") != NULL)
        {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET); /* Bật đèn LED */
        }
        else if (strstr(ipdPtr, "GET /led_off") != NULL)
        {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET); /* Tắt đèn LED */
        }

        /* Kiểm tra trạng thái đèn hiện thời */
        bool isLedOn = (HAL_GPIO_ReadPin(GPIOD, GPIO_PIN_12) == GPIO_PIN_SET);

        /* Tạo mã nguồn giao diện HTML Dashboard hiện đại responsive */
        char htmlPage[1024];
        snprintf(htmlPage, sizeof(htmlPage),
                 "<!DOCTYPE html><html>"
                 "<head><title>STM32F407 Smart Home</title>"
                 "<meta name='viewport' content='width=device-width, initial-scale=1'>"
                 "<meta http-equiv='refresh' content='5'>"
                 "<style>body{font-family:Arial;text-align:center;background:#f4f7f6;padding:20px;}"
                 ".card{background:white;padding:30px;border-radius:15px;box-shadow:0 6px 15px rgba(0,0,0,0.1);max-width:420px;margin:auto;}"
                 "h2{color:#333;margin-bottom:10px;} .temp{font-size:36px;color:#007bff;font-weight:bold;margin:15px 0;}"
                 ".btn{display:inline-block;padding:12px 28px;font-size:16px;font-weight:bold;color:white;border-radius:8px;text-decoration:none;margin:10px;}"
                 ".btn-on{background:#28a745;} .btn-off{background:#dc3545;}"
                 ".status{font-size:18px;margin-top:15px;color:#555;}"
                 "</style></head>"
                 "<body><div class='card'>"
                 "<h2>STM32F407 DASHBOARD</h2>"
                 "<p>He Thong Giam Sat Nha Thong Minh</p><hr>"
                 "<p>Nhiet do phong khach:</p>"
                 "<div class='temp'>%.1f &deg;C</div>"
                 "<div class='status'>Trang thai den: <b>%s</b></div>"
                 "<a href='/led_on' class='btn btn-on'>BAT DEN</a>"
                 "<a href='/led_off' class='btn btn-off'>TAT DEN</a>"
                 "</div></body></html>",
                 roomTemp, isLedOn ? "DANG BAT (ON)" : "DANG TAT (OFF)");

        ESP_SendHttpResponse(linkId, htmlPage);
        ESP_ClearBuffer();
    }
}
```

---

## 11.5. "UNDER THE HOOD" & CASE STUDY (THEO GIÁO TRÌNH ARM)

### 11.5.1. Under the Hood (11.3.1): Kiến Trúc Socket & Mô Hình Truyền Thông Mạng
- **Khái niệm Socket:** Là điểm cuối logic của một kênh truyền thông hai chiều trong mạng IP, được định danh duy nhất bởi cặp:
  $$\text{Socket} = (\text{Địa chỉ IP}, \text{Số hiệu cổng Port})$$
  - Ví dụ: `192.168.1.105:80` đại diện cho dịch vụ Web Server trên STM32F407.
- **Phân biệt HTTP Polling vs WebSockets:**
  - *HTTP Polling:* Trình duyệt định kỳ gửi yêu cầu `GET` lên server mỗi vài giây (sử dụng thẻ `<meta http-equiv='refresh' content='5'>`). Nhược điểm: Tạo nhiều gói tin Header thừa, gây tốn băng thông và tải CPU.
  - *WebSockets:* Duy trì một kết nối TCP liên tục hai chiều, cho phép server chủ động đẩy (Push) dữ liệu ngay khi cảm biến thay đổi mà không cần trình duyệt hỏi thăm.

### 11.5.2. Case Study (11.4.1): Hệ Thống Giám Sát Môi Trường Nhà Máy (Industrial IoT Web Gateway)
- **Bối cảnh ứng dụng:** Trong nhà xưởng sản xuất thực phẩm hoặc dược phẩm, nhiệt độ và độ ẩm phòng sạch bắt buộc phải duy trì nghiêm ngặt trong dải tiêu chuẩn 24/7.
- **Giải pháp thiết bị Gateway:**
  - Vi điều khiển STM32F407 thu thập số liệu từ các cảm biến nhiệt ẩm công nghiệp qua mạng RS-485 Modbus.
  - Module Wi-Fi đóng vai trò máy chủ Web Server nội bộ (Intranet Web Server).
  - Trưởng ca sản xuất không cần cài đặt bất kỳ phần mềm đặc thù nào; chỉ cần mở trình duyệt máy tính hoặc điện thoại truy cập địa chỉ IP nội bộ của trạm để xem biểu đồ nhiệt độ trực quan và tải tệp báo cáo định dạng CSV về máy tính.

---

## 11.6. CÂU HỎI ÔN TẬP VÀ BÀI TẬP THIẾT KẾ CHƯƠNG 11

1. **Câu hỏi 1 (Giao thức HTTP):** *Tại sao trong bản tin phản hồi HTTP 200 OK, trường `Content-Length` bắt buộc phải được tính toán chính xác tuyệt đối theo độ dài byte của chuỗi HTML? Nếu tính toán số lượng này lớn hơn độ dài thực tế thì hiện tượng gì sẽ xảy ra trên trình duyệt Web?*
2. **Câu hỏi 2 (Phần cứng nguồn điện):** *Hãy giải thích nguyên nhân vật lý tại sao module Wi-Fi ESP8266 lại tiêu thụ dòng đỉnh lên tới $300\text{mA}$ khi phát sóng? Tại sao việc cấp nguồn cho ESP8266 từ chân 3.3V của mạch nạp ST-Link thường dẫn đến hiện tượng treo hoặc reset liên tục?*
3. **Câu hỏi 3 (Bảo mật mạng nhúng):** *Giao thức HTTP truyền dữ liệu dưới dạng văn bản thuần túy (Plain text). Để bảo vệ an toàn cho hệ thống mở khóa cửa qua Web, người ta phải nâng cấp lên giao thức nào và những khó khăn khi triển khai giao thức đó trên vi điều khiển tài nguyên nhỏ là gì?*
4. **Bài tập thiết kế hệ thống:** *Thiết kế một trạm thời tiết IoT thông minh: Sử dụng STM32F407, module ESP8266 qua USART6, cảm biến nhiệt độ DS18B20 trên PB12, phục vụ một trang Web Dashboard hiển thị nhiệt độ và biểu đồ diễn biến trong 1 giờ gần nhất.*
