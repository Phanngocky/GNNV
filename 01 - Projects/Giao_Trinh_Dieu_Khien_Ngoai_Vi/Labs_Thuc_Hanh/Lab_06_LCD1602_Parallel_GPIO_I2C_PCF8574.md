# BÀI THỰC HÀNH SỐ 06: MÀN HÌNH KÝ TỰ LCD QUA BUS SONG SONG VÀ BUS NỐI TIẾP I2C (PCF8574)
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết giao thức $\rightarrow$ Mạch điện trở kéo $\rightarrow$ Lập trình Driver $\rightarrow$ Bắt dạng sóng)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Giao thức I2C), CLO 2 (Tính điện trở kéo Rp), CLO 3 (Lập trình I2C Driver), CLO 5 (Phân tích gói tin I2C trên Logic Analyzer)  
**Căn cứ giáo trình:** *ARM Education - Chương 6: Liquid Crystal Displays and I2C Bus (Ex 6.1 $\div$ 6.3)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Giao tiếp Ngoại vi:**
   - Phân tích cấu trúc phần cứng vi điều khiển màn hình HD44780: Chế độ song song 8-bit vs 4-bit, thanh ghi lệnh (IR) và thanh ghi dữ liệu (DR).
   - Nắm vững cơ chế vật lý của chuẩn giao tiếp nối tiếp $I^2C$: Tầng ngõ ra cực thu hở (Open-Drain), vai trò của điện trở kéo lên ($R_p$), điều kiện Start, Stop, truyền địa chỉ 7-bit + bit R/W, bit phản hồi ACK/NACK, và cơ chế giữ xung nhịp (Clock Stretching).
   - Phân tích kiến trúc IC mở rộng 8-bit I/O PCF8574 điều khiển màn hình LCD chỉ với 2 dây dẫn.
2. **Kỹ thuật Lập trình:**
   - Điều khiển màn hình LCD 16x2 trực tiếp qua 6 chân GPIO ở chế độ 4-bit (Ex 6.1, 6.2).
   - Cấu hình khối phần cứng `I2C1` (`PB6` SCL, `PB7` SDA) trên STM32F407 ở tốc độ Standard Mode ($100\text{ kHz}$).
   - Xây dựng thư viện Driver điều khiển LCD qua module PCF8574: Khởi tạo màn hình, gửi lệnh, gửi ký tự, đặt vị trí con trỏ `gotoxy(x, y)`, và tạo ký tự tự định nghĩa (Custom CGRAM character) (Ex 6.3).
3. **Đo kiểm & Bắt dạng sóng:**
   - Dùng Logic Analyzer bắt trọn vẹn khung truyền $I^2C$ của một byte dữ liệu, giải mã địa chỉ slave ($0x27$ hoặc $0x3F$) và kiểm tra bit phản hồi ACK thực tế.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Màn hình ký tự LCD 16x2 | Loại 16 cột $\times$ 2 dòng, tích hợp HD44780 | 1 | Kèm chiết áp chỉnh độ tương phản |
| 3 | Module mở rộng $I^2C$ LCD | Chip PCF8574T (địa chỉ mặc định `0x27`) | 1 | Đã hàn sẵn sau lưng LCD |
| 4 | Điện trở kim loại 0.25W | $4.7\text{ k}\Omega$ (Điện trở kéo $R_p$) | 2 | Dùng khi test chân I2C rời |
| 5 | Thiết bị đo | Logic Analyzer 8CH 24MHz (PulseView) | 1 | Bắt dạng sóng SDA và SCL |

---

## 3. CƠ SỞ TÍNH TOÁN VÀ SƠ ĐỒ NGUYÊN LÝ GHÉP NỐI

### 3.1. Tính toán điện trở kéo lên ($R_p$) cho bus $I^2C$
Vì các chân $I^2C$ có tầng ngõ ra là cực thu hở (Open-Drain), đường truyền không thể tự kéo lên mức cao nếu thiếu điện trở kéo lên $R_p$.
- **Giới hạn dưới của $R_p$ (Dòng chìm tối đa của chân GPIO):**
  $$R_{p,\min} = \frac{V_{DD} - V_{OL,\max}}{I_{OL}} = \frac{3.3\text{V} - 0.4\text{V}}{3\text{mA}} \approx 967\Omega$$
- **Giới hạn trên của $R_p$ (Thời gian tăng sườn lên $t_r \le 1000\text{ ns}$ do điện dung ký sinh đường dây $C_b$):**
  $$R_{p,\max} = \frac{t_r}{0.8473 \times C_b} \approx \frac{1000\text{ ns}}{0.8473 \times 200\text{ pF}} \approx 5.9\text{ k}\Omega$$
- $\implies$ **Chọn giá trị chuẩn công nghiệp:** $R_p = 4.7\text{ k}\Omega$ (Module PCF8574 thông thường đã tích hợp sẵn cặp trở $4.7\text{k}\Omega$ kéo lên nguồn nuôi).

### 3.2. Bảng đấu nối chân hệ thống
#### Chế độ $I^2C$ (Tiết kiệm chân - Sử dụng Module PCF8574):
| Chân STM32F407 | Chân Module I2C PCF8574 | Chức năng tín hiệu | Lưu ý điện áp |
| :---: | :---: | :--- | :--- |
| `PB6` (I2C1_SCL) | `SCL` | Xung nhịp đồng bộ I2C (100kHz) | Chân STM32 chịu áp 5V (FT) |
| `PB7` (I2C1_SDA) | `SDA` | Dữ liệu hai chiều I2C | Chân STM32 chịu áp 5V (FT) |
| `5V` | `VCC` | Nguồn cấp cho màn hình LCD và đèn nền | LCD 1602 cần 5V để hiển thị rõ |
| `GND` | `GND` | Mass chung hệ thống | Bắt buộc nối chung mass |

---

## 4. QUY TRÌNH CẤU HÌNH TRÊN STM32CubeMX
1. **Clock Configuration:** Cấu hình chuẩn HSE 8MHz $\rightarrow$ SYSCLK 168MHz $\rightarrow$ APB1 42MHz.
2. **Cấu hình khối I2C1:**
   - Vào mục *Connectivity* $\rightarrow$ *I2C1*.
   - Mục *I2C*: Chọn **I2C**.
   - Thẻ *Parameter Settings*:
     - I2C Speed Mode: **Standard Mode**.
     - I2C Clock Speed (Hz): **100000** ($100\text{ kHz}$).
     - Clock No Stretch Mode: **Disabled** (Cho phép Clock Stretching).
     - Primary Address Length: **7-bit**.
   - Kiểm tra cấu hình chân GPIO của `PB6` và `PB7`:
     - GPIO mode: **Alternate Function Open Drain**.
     - Pull-up / Pull-down: **Pull-up** (hoặc No pull-up nếu module ngoài đã có trở).
     - Maximum output speed: **Very High**.

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### 5.1. Thư viện Driver LCD I2C (`i2c_lcd.h` & `i2c_lcd.c`)
Tệp `i2c_lcd.h`:
```c
#ifndef I2C_LCD_H_
#define I2C_LCD_H_

#include "stm32f4xx_hal.h"

#define LCD_ADDR (0x27 << 1) /* Dịch trái 1 bit cho định dạng 8-bit HAL (0x4E) */

void LCD_Init(I2C_HandleTypeDef *hi2c);
void LCD_SendCommand(uint8_t cmd);
void LCD_SendData(uint8_t data);
void LCD_SendString(char *str);
void LCD_SetCursor(uint8_t row, uint8_t col);
void LCD_Clear(void);

#endif
```

Tệp `i2c_lcd.c`:
```c
#include "i2c_lcd.h"

static I2C_HandleTypeDef *lcd_i2c;

/* Gửi 4-bit (Nibble) dữ liệu qua PCF8574 */
static void LCD_WriteNibble(uint8_t nibble, uint8_t rs)
{
    /* PCF8574 Bit map: P7..P4 = Data, P3 = Backlight, P2 = E, P1 = RW, P0 = RS */
    uint8_t data = (nibble & 0xF0) | (rs ? 0x01 : 0x00) | 0x08; /* Bật đèn nền (0x08) */

    /* Tạo sườn xung kích hoạt chân Enable (E): Mức 1 -> Mức 0 */
    uint8_t packet[2];
    packet[0] = data | 0x04; /* E = 1 */
    packet[1] = data & ~0x04;/* E = 0 */

    HAL_I2C_Master_Transmit(lcd_i2c, LCD_ADDR, packet, 2, 100);
}

void LCD_SendCommand(uint8_t cmd)
{
    LCD_WriteNibble(cmd & 0xF0, 0);        /* Gửi 4 bit cao */
    LCD_WriteNibble((cmd << 4) & 0xF0, 0); /* Gửi 4 bit thấp */
    HAL_Delay(2);
}

void LCD_SendData(uint8_t data)
{
    LCD_WriteNibble(data & 0xF0, 1);        /* RS = 1 (Dữ liệu) */
    LCD_WriteNibble((data << 4) & 0xF0, 1);
    HAL_Delay(1);
}

void LCD_Init(I2C_HandleTypeDef *hi2c)
{
    lcd_i2c = hi2c;
    HAL_Delay(50); /* Chờ nguồn cấp ổn định > 40ms */

    /* Khởi tạo chế độ 4-bit theo đúng quy chuẩn HD44780 */
    LCD_WriteNibble(0x30, 0);
    HAL_Delay(5);
    LCD_WriteNibble(0x30, 0);
    HAL_Delay(1);
    LCD_WriteNibble(0x30, 0);
    HAL_Delay(1);
    LCD_WriteNibble(0x20, 0); /* Chuyển sang giao diện 4-bit */
    HAL_Delay(1);

    /* Cấu hình hiển thị */
    LCD_SendCommand(0x28); /* 2 dòng, font 5x8 */
    LCD_SendCommand(0x0C); /* Bật màn hình, tắt con trỏ nhấp nháy */
    LCD_SendCommand(0x06); /* Tự động tăng con trỏ sang phải */
    LCD_Clear();
}

void LCD_SetCursor(uint8_t row, uint8_t col)
{
    uint8_t address = (row == 0) ? (0x80 + col) : (0xC0 + col);
    LCD_SendCommand(address);
}

void LCD_Clear(void)
{
    LCD_SendCommand(0x01);
    HAL_Delay(2);
}

void LCD_SendString(char *str)
{
    while (*str) {
        LCD_SendData((uint8_t)*str++);
    }
}
```

### 5.2. Hiển thị thông số giám sát hệ thống Smart Home (Ex 6.3)
Trong file `Core/Src/main.c`:

```c
/* USER CODE BEGIN 2 */
LCD_Init(&hi2c1);
LCD_SetCursor(0, 0);
LCD_SendString("SMART HOME F407");
LCD_SetCursor(1, 0);
LCD_SendString("STATUS: SAFE");
HAL_Delay(2000);
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    static uint32_t lastLcdUpdate = 0;
    if (HAL_GetTick() - lastLcdUpdate >= 500)
    {
        lastLcdUpdate = HAL_GetTick();

        char buffer[17];
        float temp = 28.5f; /* Giả lập nhiệt độ đọc được từ bài Lab 3 */

        LCD_SetCursor(0, 0);
        snprintf(buffer, sizeof(buffer), "TEMP: %.1f C   ", temp);
        LCD_SendString(buffer);

        LCD_SetCursor(1, 0);
        snprintf(buffer, sizeof(buffer), "SYSTEM: ARMED   ");
        LCD_SendString(buffer);
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & BẮT DẠNG SÓNG GIAO THỨC I2C BẰNG LOGIC ANALYZER

### 6.1. Thiết lập đo lường
1. Kênh `CH0` của Logic Analyzer nối vào chân `PB6` (SCL).
2. Kênh `CH1` của Logic Analyzer nối vào chân `PB7` (SDA).
3. Kẹp mass nối vào `GND`.
4. Trên phần mềm PulseView:
   - Thêm Protocol Decoder: **I2C**.
   - Cấu hình: SCL = **CH0**, SDA = **CH1**, Address format = **7-bit**.

### 6.2. Phân tích gói tin I2C bắt được
Khi gửi dữ liệu ra màn hình LCD:
1. **Start Condition:** Đường SDA sụt từ mức cao xuống thấp trong khi đường SCL đang giữ ở mức cao.
2. **Byte 1 (Địa chỉ Slave + Write):**
   - Vi điều khiển phát 7 bit địa chỉ: `0 1 0 0 1 1 1` ($0x27$).
   - Bit thứ 8: `0` (Lệnh Write).
   - Bit thứ 9: Slave PCF8574 kéo đường SDA xuống thấp tạo xung **ACK (Acknowledge)**.
3. **Byte 2 & 3 (Dữ liệu):**
   - Truyền dữ liệu 8-bit điều khiển các chân của PCF8574.
   - Sau mỗi byte, PCF8574 luôn phản hồi bằng bit ACK.
4. **Stop Condition:** Đường SDA chuyển từ mức thấp lên cao trong khi SCL đang giữ ở mức cao.

```
       Start  Addr=0x27  W  ACK    Data=0x0C     ACK  Stop
SDA: ‾‾\___/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾‾
SCL: ‾‾‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾‾‾
```

---

## 7. XỬ LÝ SỰ CỐ: GỠ KẸT BUS I2C (BUS CLEAR SEQUENCE)
Nếu vi điều khiển bị Reset đột ngột khi đang nhận dở dữ liệu từ một ngoại vi I2C (ngoại vi đang giữ SDA ở mức 0):
- **Hiện tượng:** Bus I2C bị treo cứng vĩnh viễn (I2C Bus Stuck), STM32 khởi động lại báo lỗi `HAL_BUSY`.
- **Trình tự giải phóng bus phần mềm (I2C Bus Clear):**
  1. Cấu hình chân SCL sang GPIO Output.
  2. Phát liên tiếp 9 xung nhịp xung Clock trên chân SCL bằng phần mềm để ép ngoại vi nhả đường SDA.
  3. Tạo điều kiện STOP condition trên bus.
  4. Cấu hình lại chân SCL về ngoại vi I2C phần cứng.

---

## 8. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 8.1. Báo cáo thí nghiệm
- Ảnh chụp màn hình LCD 16x2 hiển thị đầy đủ thông tin nhiệt độ và trạng thái.
- Ảnh chụp màn hình Logic Analyzer giải mã khung truyền I2C có đầy đủ Start, Địa chỉ, Data và ACK.

### 8.2. Câu hỏi đánh giá
1. *Tại sao đường truyền I2C bắt buộc phải dùng ngõ ra Open-Drain kết hợp điện trở kéo lên mà không được dùng ngõ ra Push-Pull?*
2. *Hiện tượng "Clock Stretching" trong giao thức I2C là gì? Khi nào thiết bị Slave cần kích hoạt cơ chế này?*
3. *Nếu trên cùng bus I2C ta gắn thêm một IC đồng hồ thời gian thực DS3231 (Địa chỉ `0x68`), hai linh kiện này có thể xung đột nhau không? Vi điều khiển phân biệt chúng như thế nào?*