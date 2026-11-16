# BÀI THỰC HÀNH SỐ 08: CHUẨN GIAO TIẾP ĐỒNG BỘ SPI VÀ MÀN HÌNH ĐỒ HỌA OLED (SSD1306)
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết SPI 4 Modes $\rightarrow$ Frame Buffer $\rightarrow$ Lập trình Đồ họa $\rightarrow$ Bắt sóng Logic)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Giao thức SPI), CLO 2 (Mạch SPI & OLED), CLO 3 (Lập trình SPI Driver & Frame Buffer), CLO 5 (Bắt sóng SPI MHz)  
**Căn cứ giáo trình:** *ARM Education - Chương 6: Graphical Displays and SPI Bus (Ex 6.4 $\div$ 6.5)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Chuẩn Giao tiếp Đồng bộ Tốc độ Cao SPI:**
   - Phân tích kiến trúc bus SPI 4 dây: $SCK$ (Serial Clock), $MOSI$ (Master Out Slave In), $MISO$ (Master In Slave Out), và $\overline{CS} / NSS$ (Chip Select tích cực mức thấp).
   - Nắm vững 4 chế độ hoạt động của SPI dựa trên cực tính xung nhịp (`CPOL`) và pha lấy mẫu (`CPHA`):
     - Mode 0: `CPOL = 0, CPHA = 0` (Xung nhịp nghỉ mức 0, lấy mẫu ở sườn lên đầu tiên).
     - Mode 1: `CPOL = 0, CPHA = 1` (Xung nhịp nghỉ mức 0, lấy mẫu ở sườn xuống thứ hai).
     - Mode 2: `CPOL = 1, CPHA = 0` (Xung nhịp nghỉ mức 1, lấy mẫu ở sườn xuống đầu tiên).
     - Mode 3: `CPOL = 1, CPHA = 1` (Xung nhịp nghỉ mức 1, lấy mẫu ở sườn lên thứ hai).
2. **Kỹ thuật Lập trình Màn hình Đồ họa:**
   - Cấu hình khối phần cứng `SPI1` trên STM32F407 (Bus APB2 84MHz, tốc độ truyền lên tới $10.5\text{ Mbps}$).
   - Xây dựng kiến trúc **Bộ đệm khung hình (Frame Buffer)** $1024\text{ bytes}$ ($128 \times 64 / 8$) trong bộ nhớ SRAM của STM32F407.
   - Hiện thực hóa các hàm đồ họa cơ bản: Chấm điểm ảnh (`DrawPixel`), vẽ đoạn thẳng (Thuật toán Bresenham), hiển thị ký tự chữ và số font $5\times 7$, và vẽ **Đồ thị diễn biến nhiệt độ thời gian thực** (Real-time Temperature Graph) (Ex 6.4, 6.5).
3. **Đo kiểm & Đánh giá Hiệu năng:**
   - Dùng Logic Analyzer bắt sóng chùm xung SPI tốc độ cao và đo thời gian làm tươi toàn bộ khung hình (Frame Refresh Latency).

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Module OLED 0.96 inch SPI | Chip điều khiển SSD1306, $128 \times 64$ điểm ảnh | 1 | Chuẩn giao tiếp SPI 7 chân |
| 3 | Cảm biến DS18B20 (hoặc LM35) | Đã chuẩn bị từ bài trước | 1 | Nguồn cấp dữ liệu nhiệt độ |
| 4 | Thiết bị đo | Logic Analyzer 8CH 24MHz | 1 | Bắt dạng sóng SPI MHz |

---

## 3. SƠ ĐỒ ĐẤU NỐI MẠCH VÀ PHÂN BỔ CHÂN (PINOUT)

### Bảng kết nối Module OLED SSD1306 SPI vào STM32F407:
| Chân Module OLED | Chân STM32F407 | Chức năng phần cứng | Chế độ cấu hình CubeMX |
| :---: | :---: | :--- | :--- |
| `GND` | `GND` | Mass chung | Nguồn âm |
| `VCC` | `3V3` | Nguồn nuôi $3.3\text{V}$ | Nguồn dương |
| `D0` / `CLK` / `SCK` | `PA5` | Xung nhịp SPI1_SCK | Alternate Function SPI1 |
| `D1` / `MOSI` / `DIN` | `PA7` | Dữ liệu Master Out SPI1_MOSI | Alternate Function SPI1 |
| `RES` / `RST` | `PB2` | Reset phần cứng màn hình (Active-LOW) | GPIO_Output Push-Pull |
| `DC` / `D/C` | `PB1` | Lựa chọn Lệnh (0) hay Dữ liệu (1) | GPIO_Output Push-Pull |
| `CS` | `PB0` | Chọn chip màn hình (Active-LOW) | GPIO_Output Push-Pull |

> [!NOTE]
> Màn hình OLED SSD1306 chỉ nhận dữ liệu từ vi điều khiển và không cần truyền dữ liệu ngược lại, do đó đường `MISO` (`PA6`) có thể bỏ trống để tiết kiệm chân GPIO cho ứng dụng khác.

---

## 4. CẤU HÌNH TRÊN STM32CubeMX
1. **Clock Tree:** SYSCLK 168MHz, APB2 84MHz.
2. **Cấu hình SPI1:**
   - Chọn mục *Connectivity* $\rightarrow$ *SPI1*.
   - Mode: **Transmit Only Master** (Chỉ phát dữ liệu).
   - Parameter Settings:
     - Frame Format: **Motorola**.
     - Data Size: **8 Bits**.
     - First Bit: **MSB First** (Quy định bắt buộc của chip SSD1306).
     - Clock Polarity (CPOL): **Low** (Mode 0).
     - Clock Phase (CPHA): **1 Edge** (Mode 0).
     - Baud Rate Prescaler: **/8** $\implies$ Baud Rate = $84\text{ MHz} / 8 = 10.5\text{ Mbits/s}$.
3. **Cấu hình GPIO điều khiển:**
   - `PB0 (CS)`: Output Push-Pull, High-Speed, Mặc định mức 1 (High).
   - `PB1 (DC)`: Output Push-Pull, High-Speed, Mặc định mức 0 (Command).
   - `PB2 (RST)`: Output Push-Pull, High-Speed, Mặc định mức 1 (High).

---

## 5. MÃ NGUỒN DRIVER OLED SSD1306 VÀ ĐỒ HỌA THỜI GIAN THỰC

### 5.1. Cấu trúc Bộ đệm khung hình (Frame Buffer)
Màn hình có độ phân giải $128 \times 64$ điểm ảnh:
- Chia thành 8 trang (Page 0 $\div$ Page 7), mỗi trang cao 8 pixels và rộng 128 cột.
- Kích thước bộ đệm RAM: $128 \times 8 = 1024\text{ bytes} = 1\text{ KB}$.
- Để chấm một điểm $(x, y)$ với $0 \le x < 128$ và $0 \le y < 64$:
  $$\text{Vị trí byte trong đệm} = x + \left(\lfloor \frac{y}{8} \rfloor \times 128\right)$$
  $$\text{Vị trí bit trong byte} = y \pmod 8$$

### 5.2. Mã nguồn Driver Đồ họa Căn bản
Tệp `Core/Src/ssd1306_spi.c`:

```c
#include "stm32f4xx_hal.h"
#include <string.h>

extern SPI_HandleTypeDef hspi1;

#define OLED_WIDTH      128
#define OLED_HEIGHT     64
#define FRAME_BUFFER_SZ (OLED_WIDTH * OLED_HEIGHT / 8)

static uint8_t frameBuffer[FRAME_BUFFER_SZ];

#define CS_LOW()    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET)
#define CS_HIGH()   HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET)
#define DC_CMD()    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_RESET)
#define DC_DATA()   HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET)

/* Gửi 1 byte lệnh xuống SSD1306 */
void SSD1306_WriteCommand(uint8_t cmd)
{
    DC_CMD();
    CS_LOW();
    HAL_SPI_Transmit(&hspi1, &cmd, 1, 10);
    CS_HIGH();
}

/* Khởi tạo màn hình OLED SSD1306 SPI */
void SSD1306_Init(void)
{
    /* Reset cứng */
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_RESET);
    HAL_Delay(10);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_SET);
    HAL_Delay(10);

    /* Chuỗi lệnh khởi tạo chuẩn SSD1306 */
    SSD1306_WriteCommand(0xAE); /* Tắt màn hình */
    SSD1306_WriteCommand(0xD5); SSD1306_WriteCommand(0x80); /* Tần số xung quét */
    SSD1306_WriteCommand(0xA8); SSD1306_WriteCommand(0x3F); /* Mux ratio 64 */
    SSD1306_WriteCommand(0xD3); SSD1306_WriteCommand(0x00); /* Display offset 0 */
    SSD1306_WriteCommand(0x40); /* Start line 0 */
    SSD1306_WriteCommand(0x8D); SSD1306_WriteCommand(0x14); /* Bật bơm điện áp Charge Pump */
    SSD1306_WriteCommand(0x20); SSD1306_WriteCommand(0x00); /* Chế độ đánh địa chỉ ngang */
    SSD1306_WriteCommand(0xA1); /* Đảo gương cột Segment remap */
    SSD1306_WriteCommand(0xC8); /* Đảo chiều quét hàng COM scan */
    SSD1306_WriteCommand(0xDA); SSD1306_WriteCommand(0x12); /* Cấu hình chân COM */
    SSD1306_WriteCommand(0x81); SSD1306_WriteCommand(0xCF); /* Độ tương phản */
    SSD1306_WriteCommand(0xD9); SSD1306_WriteCommand(0xF1); /* Thời gian nạp trước */
    SSD1306_WriteCommand(0xDB); SSD1306_WriteCommand(0x40); /* Ngưỡng VCOMH */
    SSD1306_WriteCommand(0xA4); /* Hiển thị theo nội dung RAM */
    SSD1306_WriteCommand(0xA6); /* Màn hình bình thường (không nghịch đảo màu) */
    SSD1306_WriteCommand(0xAF); /* BẬT MÀN HÌNH */

    memset(frameBuffer, 0x00, FRAME_BUFFER_SZ);
}

/* Chấm 1 điểm ảnh lên bộ đệm RAM */
void SSD1306_DrawPixel(int16_t x, int16_t y, uint8_t color)
{
    if (x < 0 || x >= OLED_WIDTH || y < 0 || y >= OLED_HEIGHT) return;

    if (color) {
        frameBuffer[x + (y / 8) * OLED_WIDTH] |= (1U << (y % 8));
    } else {
        frameBuffer[x + (y / 8) * OLED_WIDTH] &= ~(1U << (y % 8));
    }
}

/* Đẩy toàn bộ 1024 bytes từ RAM vi điều khiển ra màn hình OLED qua SPI */
void SSD1306_UpdateScreen(void)
{
    for (uint8_t page = 0; page < 8; page++) {
        SSD1306_WriteCommand(0xB0 + page); /* Chọn trang */
        SSD1306_WriteCommand(0x00);        /* Cột thấp = 0 */
        SSD1306_WriteCommand(0x10);        /* Cột cao = 0 */

        DC_DATA();
        CS_LOW();
        HAL_SPI_Transmit(&hspi1, &frameBuffer[page * OLED_WIDTH], OLED_WIDTH, 100);
        CS_HIGH();
    }
}
```

### 5.3. Ứng dụng: Vẽ biểu đồ nhiệt độ thời gian thực (Ex 6.4, 6.5)
```c
/* USER CODE BEGIN WHILE */
#define GRAPH_Y_MIN 15.0f
#define GRAPH_Y_MAX 45.0f

static uint8_t tempHistory[128] = {0};

while (1)
{
    /* Giả lập lấy mẫu nhiệt độ từ cảm biến DS18B20 */
    float currentTemp = 25.0f + (float)(rand() % 100) / 10.0f;

    /* Dịch toàn bộ biểu đồ sang trái 1 pixel để tạo hiệu ứng cuốn */
    for (int i = 0; i < 127; i++) {
        tempHistory[i] = tempHistory[i + 1];
    }
    /* Ánh xạ nhiệt độ mới sang tọa độ Y (0 ở trên đỉnh, 63 ở đáy) */
    int yPixel = 63 - (int)((currentTemp - GRAPH_Y_MIN) / (GRAPH_Y_MAX - GRAPH_Y_MIN) * 45.0f);
    if (yPixel < 16) yPixel = 16;
    if (yPixel > 63) yPixel = 63;
    tempHistory[127] = yPixel;

    /* Xóa bộ đệm và vẽ lại */
    memset(frameBuffer, 0x00, sizeof(frameBuffer));

    /* Vẽ các điểm trên đồ thị */
    for (int x = 0; x < 128; x++) {
        SSD1306_DrawPixel(x, tempHistory[x], 1);
    }

    /* Đẩy khung hình ra màn hình */
    SSD1306_UpdateScreen();
    HAL_Delay(100);
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & PHÂN TÍCH TỐC ĐỘ LÀM TƯƠI TRÊN LOGIC ANALYZER

### 6.1. Thiết lập đo lường
1. Kênh `CH0`: Nối `PA5` (SCK).
2. Kênh `CH1`: Nối `PA7` (MOSI).
3. Kênh `CH2`: Nối `PB0` (CS).
4. Thiết lập tần số lấy mẫu Logic Analyzer tối thiểu: $24\text{ MHz}$.

### 6.2. Đo thời gian cập nhật toàn màn hình (Frame Update Time)
1. Quan sát tín hiệu chân `CS`: Chân `CS` tích cực mức thấp trong suốt quá trình đẩy 128 bytes của mỗi trang.
2. Với tốc độ truyền SPI $10.5\text{ Mbps}$, thời gian truyền thuần túy cho $1024\text{ bytes} = 8192\text{ bits}$ là:
   $$t_{transfer} = \frac{8192\text{ bits}}{10{,}500{,}000\text{ bps}} \approx 0.78\text{ ms} = 780\mu s$$
3. Tốc độ làm tươi lý thuyết đạt được:
   $$\text{Frame Rate} \approx \frac{1}{0.001\text{ s}} \approx 1000\text{ FPS}$$
   *(Nhanh gấp hơn 50 lần so với bus I2C 100kHz, đáp ứng mượt mà mọi đồ họa chuyển động thời gian thực).*

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Ảnh chụp màn hình OLED hiển thị đồ thị nhiệt độ chạy mượt mà theo thời gian.
- Bảng so sánh 3 chuẩn giao tiếp: **UART vs $I^2C$ vs SPI** (Số dây, Tốc độ tối đa, Cấu trúc Master/Slave, Chi phí phần cứng).

### 7.2. Câu hỏi đánh giá
1. *Tại sao giao tiếp SPI lại đạt được tốc độ truyền vượt trội hơn nhiều so với $I^2C$ (hàng chục MHz so với vài trăm kHz)? Nêu 2 nguyên nhân cốt lõi ở tầng vật lý.*
2. *Nếu ta muốn gắn đồng thời 3 màn hình OLED trên cùng một bus SPI thì cần thêm bao nhiêu chân GPIO từ vi điều khiển STM32F407? Cách kết nối như thế nào?*
3. *Bộ đệm khung hình (Frame Buffer) chiếm $1024\text{ bytes}$. Nếu tăng độ phân giải màn hình lên chuẩn màu TFT $320 \times 240$ RGB565 (16-bit màu), bộ nhớ RAM cần thiết là bao nhiêu? Chip STM32F407VGT6 có đủ RAM chứa toàn bộ khung hình này không?*