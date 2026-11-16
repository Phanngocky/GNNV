# BÀI THỰC HÀNH SỐ 07: CHUẨN GIAO TIẾP 1-WIRE VÀ CẢM BIẾN NHIỆT ĐỘ SỐ CHÍNH XÁC CAO DS18B20
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết giản đồ xung micro-giây $\rightarrow$ Cấu hình DWT $\rightarrow$ Lập trình Driver $\rightarrow$ Bắt sóng Oscilloscope)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Chuẩn 1-Wire), CLO 2 (Mạch trở kéo 4.7k), CLO 3 (Lập trình DWT delay & Scratchpad), CLO 5 (Bắt sóng micro-giây)  
**Căn cứ giáo trình:** *ARM Education - Mở rộng chuyên sâu Chương 6: Advanced Serial Protocols (1-Wire Interface)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Chuẩn Giao tiếp 1-Wire:**
   - Nắm vững nguyên lý hoạt động của bus 1-Wire độc quyền (Dallas / Maxim): Toàn bộ việc truyền thông hai chiều, đồng bộ xung nhịp và cấp nguồn đều có thể tích hợp trên một dây dẫn duy nhất ($DQ$).
   - Phân tích cấu trúc phần cứng cảm biến số DS18B20: Mã ROM 64-bit laser duy nhất trên toàn cầu (Family code `0x28`), bộ nhớ đệm Scratchpad 9 byte, độ phân giải cấu hình linh hoạt từ 9 đến 12 bit ($0.0625^\circ\text{C}$).
   - Phân tích chi tiết giản đồ thời gian cấp độ micro-giây ($\mu s$):
     - Xung khởi tạo (Reset Pulse): Master kéo bus thấp tối thiểu $480\mu s$.
     - Xung phản hồi hiện diện (Presence Pulse): Slave kéo thấp $60 \div 240\mu s$.
     - Khe ghi dữ liệu (Write Slot 0 và 1): Chu kỳ khe thời gian $60 \div 120\mu s$.
     - Khe đọc dữ liệu (Read Slot): Master kéo thấp $1 \div 2\mu s$, nhả bus và lấy mẫu dữ liệu chính xác tại thời điểm $15\mu s$.
2. **Kỹ thuật Lập trình:**
   - Ứng dụng bộ đếm xung chu kỳ phần cứng Cortex-M4 **DWT (Data Watchpoint and Trace)** để tạo hàm trễ chính xác tuyệt đối cấp micro-giây (`delay_us`) ở xung nhịp 168MHz.
   - Cấu hình chân `PB12` ở chế độ **GPIO Output Open-Drain** kết hợp điện trở kéo lên ngoài $4.7\text{ k}\Omega$, cho phép đảo chiều truyền nhận không cần chuyển đổi thanh ghi hướng liên tục.
   - Viết trọn vẹn Driver điều khiển DS18B20: Reset bus, gửi lệnh bỏ qua ROM (`0xCC`), lệnh khởi động đo nhiệt độ (`0x44`), lệnh đọc bộ nhớ nháp Scratchpad (`0xBE`) và tính toán nhiệt độ âm/dương 12-bit.
3. **Đo kiểm & Bắt dạng sóng:**
   - Dùng máy hiện sóng (Oscilloscope) bắt trọn vẹn chuỗi xung: Xung Reset $\rightarrow$ Xung nhả $\rightarrow$ Xung Presence của cảm biến.
   - Dùng Logic Analyzer giải mã từng bit trong quá trình chuyển đổi nhiệt độ.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Cảm biến nhiệt độ số DS18B20 | Dạng vỏ TO-92 hoặc đầu dò bọc kim loại chống nước | 1 | Dải đo $-55^\circ\text{C} \div +125^\circ\text{C}$ |
| 3 | Điện trở kim loại 0.25W | $4.7\text{ k}\Omega$ (Điện trở kéo lên đường $DQ$) | 1 | Bắt buộc phải có |
| 4 | Màn hình LCD I2C 16x2 | Đã cấu hình từ Bài Lab 6 | 1 | Hiển thị nhiệt độ chính xác |
| 5 | Máy đo | Oscilloscope 2CH & Logic Analyzer | 1 | Bắt xung thời gian thực |

---

## 3. THIẾT KẾ MẠCH NGUYÊN LÝ VÀ ĐẤU NỐI LINH KIỆN

### 3.1. Sơ đồ mạch điện tử chuẩn
Cảm biến DS18B20 có 3 chân nhìn từ mặt vát phẳng (Vỏ TO-92):
- **Chân 1 (GND):** Nối mass.
- **Chân 2 (DQ):** Đường truyền dữ liệu hai chiều. Nối vào chân `PB12` của STM32F407. Đồng thời mắc điện trở kéo lên $4.7\text{ k}\Omega$ lên nguồn `3.3V`.
- **Chân 3 (VDD):** Chân cấp nguồn dương $3.0\text{V} \div 5.5\text{V}$. Nối vào nguồn `3.3V` của STM32.

```
       STM32F407                      DS18B20
     +-----------+                +-------------+
     |      3.3V |----------------| VDD (Pin 3) |
     |           |        +-------|             |
     |           |        |  4.7k |             |
     |           |       [R]      |             |
     |           |        |       |             |
     |      PB12 |--------+-------| DQ  (Pin 2) |
     |           |                |             |
     |       GND |----------------| GND (Pin 1) |
     +-----------+                +-------------+
```

### 3.2. Bảng đấu nối chân hệ thống
| Chân STM32F407 | Chân Cảm biến DS18B20 | Màu dây chuẩn (Đầu dò) | Chức năng tín hiệu |
| :---: | :---: | :---: | :--- |
| `3V3` | Chân 3 (`VDD`) | Màu Đỏ (Red) | Nguồn cấp dương 3.3V |
| `GND` | Chân 1 (`GND`) | Màu Đen (Black) | Mass chung |
| `PB12` | Chân 2 (`DQ`) | Màu Vàng / Trắng | Tín hiệu 1-Wire hai chiều |

> [!WARNING]
> Cảm biến DS18B20 rất dễ bị đánh thủng nếu cắm ngược chân GND và VDD (thân cảm biến sẽ nóng ran trong vài giây). Sinh viên phải quan sát kỹ góc nhìn chân hoặc bảng màu dây trước khi bật nguồn.

---

## 4. BỘ ĐẾM DWT VÀ HÀM TRỄ MICRO-GIÂY Ở XUNG NHỊP 168MHz

Chuẩn 1-Wire yêu cầu các khoảng định thời rất ngắn: $1\mu s, 10\mu s, 60\mu s, 480\mu s$. Hàm `HAL_Delay()` có độ phân giải tối thiểu là $1\text{ ms} = 1000\mu s$ nên hoàn toàn không thể sử dụng.  
Trên lõi ARM Cortex-M4, ta sử dụng thanh ghi **DWT->CYCCNT** đếm chu kỳ xung nhịp hệ thống:
- Mỗi chu kỳ xung nhịp ở 168MHz tương ứng: $T_{clk} = \frac{1}{168{,}000{,}000\text{ Hz}} \approx 5.95\text{ ns}$.
- $1\mu s$ tương ứng đúng $168$ chu kỳ đếm của `CYCCNT`.

### Khởi tạo DWT và hàm `DWT_Delay_us()`:
```c
/* Khởi tạo thanh ghi DWT của Cortex-M4 */
void DWT_Init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk; /* Bật khối Trace */
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;             /* Bật bộ đếm CYCCNT */
}

/* Hàm tạo trễ micro-giây chính xác tuyệt đối */
void DWT_Delay_us(uint32_t us)
{
    uint32_t startTicks = DWT->CYCCNT;
    uint32_t delayTicks = us * (SystemCoreClock / 1000000); /* 168 ticks cho mỗi 1us */

    while ((DWT->CYCCNT - startTicks) < delayTicks);
}
```

---

## 5. MÃ NGUỒN DRIVER 1-WIRE VÀ ĐỌC DS18B20 HOÀN CHỈNH

### 5.1. Khởi tạo GPIO Open-Drain
Chân `PB12` được cấu hình là **GPIO Output Open-Drain**:
- Khi ghi mức 0: Transistor bên trong kéo chân xuống mass ($0\text{V}$).
- Khi ghi mức 1: Transistor ngắt, điện trở kéo ngoài $4.7\text{k}\Omega$ tự động kéo chân lên $3.3\text{V}$. Khi đó vi điều khiển có thể đọc trực tiếp trạng thái chân bằng lệnh `HAL_GPIO_ReadPin()` mà **không cần đổi chế độ chân (Direction)**!

```c
/* Cấu hình chân PB12 trên STM32CubeMX:
   - Mode: Output Open-Drain
   - Pull: No pull-up and no pull-down
   - Speed: Very High
*/
#define DS18B20_PORT    GPIOB
#define DS18B20_PIN     GPIO_PIN_12

#define DQ_LOW()        HAL_GPIO_WritePin(DS18B20_PORT, DS18B20_PIN, GPIO_PIN_RESET)
#define DQ_HIGH()       HAL_GPIO_WritePin(DS18B20_PORT, DS18B20_PIN, GPIO_PIN_SET)
#define DQ_READ()       HAL_GPIO_ReadPin(DS18B20_PORT, DS18B20_PIN)
```

### 5.2. Các hàm giao tiếp cấp thấp 1-Wire
```c
/* 1. Phát xung Reset và kiểm tra xung phản hồi Presence */
uint8_t DS18B20_Reset(void)
{
    uint8_t presence = 0;
    DQ_LOW();
    DWT_Delay_us(480); /* Kéo bus thấp 480us */

    DQ_HIGH();          /* Nhả bus cho trở kéo lên */
    DWT_Delay_us(80);  /* Chờ 80us rồi lấy mẫu xung Presence */

    if (DQ_READ() == GPIO_PIN_RESET) {
        presence = 1;   /* Cảm biến phản hồi có mặt */
    }

    DWT_Delay_us(400); /* Chờ hết chu kỳ Reset (tổng > 480us) */
    return presence;
}

/* 2. Ghi 1 bit dữ liệu xuống bus 1-Wire */
void DS18B20_WriteBit(uint8_t bit)
{
    if (bit) {
        /* Write 1 Slot */
        DQ_LOW();
        DWT_Delay_us(2);   /* Kéo thấp ngắn 2us */
        DQ_HIGH();          /* Nhả bus ngay lập tức */
        DWT_Delay_us(60);  /* Giữ mức cao đến hết khe thời gian */
    } else {
        /* Write 0 Slot */
        DQ_LOW();
        DWT_Delay_us(65);  /* Kéo thấp toàn bộ khe thời gian */
        DQ_HIGH();
        DWT_Delay_us(2);
    }
}

/* 3. Đọc 1 bit dữ liệu từ bus 1-Wire */
uint8_t DS18B20_ReadBit(void)
{
    uint8_t bit = 0;
    DQ_LOW();
    DWT_Delay_us(2);       /* Master kéo thấp 2us khởi động khe đọc */
    DQ_HIGH();              /* Nhả bus */
    DWT_Delay_us(10);      /* Chờ 10us trước thời điểm lấy mẫu 15us */

    if (DQ_READ() == GPIO_PIN_SET) {
        bit = 1;
    }
    DWT_Delay_us(50);      /* Chờ hết khe thời gian đọc */
    return bit;
}

/* 4. Ghi 1 byte (LSB trước) */
void DS18B20_WriteByte(uint8_t byte)
{
    for (int i = 0; i < 8; i++) {
        DS18B20_WriteBit(byte & 0x01);
        byte >>= 1;
    }
}

/* 5. Đọc 1 byte (LSB trước) */
uint8_t DS18B20_ReadByte(void)
{
    uint8_t byte = 0;
    for (int i = 0; i < 8; i++) {
        if (DS18B20_ReadBit()) {
            byte |= (1U << i);
        }
    }
    return byte;
}
```

### 5.3. Hàm khởi động đo và chuyển đổi nhiệt độ thực tế
```c
float DS18B20_ReadTemperature(void)
{
    /* Bước 1: Phát lệnh yêu cầu cảm biến chuyển đổi nhiệt độ */
    if (!DS18B20_Reset()) return -999.0f; /* Lỗi không thấy cảm biến */
    DS18B20_WriteByte(0xCC); /* Lệnh Skip ROM (Áp dụng khi chỉ có 1 cảm biến trên bus) */
    DS18B20_WriteByte(0x44); /* Lệnh Convert Temperature */

    /* Bước 2: Chờ cảm biến chuyển đổi (Độ phân giải 12-bit cần tối đa 750ms) */
    HAL_Delay(750);

    /* Bước 3: Đọc kết quả từ Scratchpad */
    if (!DS18B20_Reset()) return -999.0f;
    DS18B20_WriteByte(0xCC); /* Skip ROM */
    DS18B20_WriteByte(0xBE); /* Lệnh Read Scratchpad */

    uint8_t tempLSB = DS18B20_ReadByte();
    uint8_t tempMSB = DS18B20_ReadByte();

    /* Đọc bỏ qua 7 byte còn lại của Scratchpad */
    for (int i = 0; i < 7; i++) {
        DS18B20_ReadByte();
    }

    /* Ghép 2 byte thành số nguyên 16-bit có dấu */
    int16_t rawTemp = (int16_t)((tempMSB << 8) | tempLSB);

    /* Mỗi đơn vị LSB tương ứng 0.0625 độ C */
    float actualTemp = (float)rawTemp * 0.0625f;
    return actualTemp;
}
```

---

## 6. ĐO KIỂM & PHÂN TÍCH DẠNG SÓNG TRÊN MÁY HIỆN SÓNG

### 6.1. Thiết lập máy hiện sóng (Oscilloscope)
1. Que đo kết nối trực tiếp vào chân `PB12` (Đường $DQ$).
2. Kẹp mass nối vào chân `GND`.
3. Timebase: $100\mu s\text{/div}$, V/div: $1\text{ V/div}$.
4. Trigger: Sườn xuống (Falling Edge) tại mức $1.5\text{V}$, Mode Single.

### 6.2. Giải mã dạng sóng thực tế
Khi hàm `DS18B20_Reset()` chạy:
1. **Giai đoạn Master kéo thấp:** Đường tín hiệu $DQ$ tụt từ $3.3\text{V}$ xuống $0\text{V}$ và kéo dài đúng $480\mu s$ (gần 5 ô timebase).
2. **Giai đoạn nhả bus:** Master ngắt ngõ ra, trở kéo $4.7\text{k}\Omega$ kéo đường tín hiệu vọt lên $3.3\text{V}$.
3. **Giai đoạn Slave kéo Presence:** Sau khoảng $30\mu s$, cảm biến DS18B20 tự động ghì đường tín hiệu xuống $0\text{V}$ trong khoảng $120\mu s$ để thông báo "Tôi đang có mặt trên đường truyền".
4. **Giai đoạn kết thúc:** DS18B20 nhả bus, tín hiệu trở lại mức cao $3.3\text{V}$ sẵn sàng nhận byte lệnh.

```
       Master Reset (480us)     Release  Presence (120us)   Ready
3.3V: ‾\_______________________/‾‾‾‾‾‾‾\_______________/‾‾‾‾‾‾‾‾‾‾
        |<-      480 us      ->|  30us  |<-   120 us  ->|
```

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Ảnh chụp dạng sóng xung Reset và Presence đo được từ máy hiện sóng thực tế kèm thước đo thời gian con trỏ (Cursors).
- Bảng so sánh độ chính xác và độ ổn định giữa cảm biến nhiệt độ Analog LM35 (Bài Lab 3) và cảm biến số 1-Wire DS18B20 (Bài Lab 7).

### 7.2. Câu hỏi đánh giá
1. *Tại sao cấu hình Open-Drain lại bắt buộc phải có điện trở kéo lên ngoài? Nếu chọn giá trị điện trở kéo quá lớn ($100\text{ k}\Omega$) thì dạng sóng tín hiệu sẽ bị méo mó như thế nào?*
2. *Lệnh "Skip ROM (`0xCC`)" có ý nghĩa gì? Trong trường hợp trên cùng một đường dây 1-Wire ta gắn 5 cảm biến DS18B20 ở 5 phòng khác nhau, ta có dùng lệnh Skip ROM được không? Phải dùng lệnh nào để phân biệt từng cảm biến?*
3. *Độ phân giải nhiệt độ 12-bit có bước nhảy nhỏ nhất là bao nhiêu $^\circ\text{C}$? Mất bao lâu để cảm biến hoàn thành một lần chuyển đổi nhiệt độ ở độ phân giải này?*