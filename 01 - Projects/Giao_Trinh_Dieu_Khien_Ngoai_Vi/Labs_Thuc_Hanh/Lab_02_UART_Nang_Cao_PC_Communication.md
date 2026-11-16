# BÀI THỰC HÀNH SỐ 02: TRUYỀN THÔNG UART NÂNG CAO VÀ GIAO TIẾP MÁY TÍNH PC
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết giao thức $\rightarrow$ Cấu hình $\rightarrow$ Lập trình $\rightarrow$ Đo kiểm)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Hiểu UART/BRR), CLO 2 (Tính Baudrate APB1), CLO 3 (Lập trình Console), CLO 5 (Bắt dạng sóng Logic Analyzer)  
**Căn cứ giáo trình:** *ARM Education - Chương 2: Serial Communication (Ex 2.1 $\div$ 2.5)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết & Tính toán Thanh ghi:**
   - Phân tích khung truyền UART chuẩn NRZ: Start bit, 8 Data bits (LSB first), No Parity, 1 Stop bit (8N1).
   - Tự tay tính toán giá trị nạp vào thanh ghi cấu hình tốc độ Baudrate `USART_BRR` trên bus APB1 ($f_{PCLK1} = 42\text{ MHz}$) ở tốc độ 115200 bps.
   - Nắm vững 4 cờ báo lỗi phần cứng của UART: Parity Error (`PE`), Framing Error (`FE`), Noise Error (`NE`), và Overrun Error (`ORE`).
2. **Kỹ thuật Lập trình:**
   - Cấu hình khối `USART2` (`PA2` TX, `PA3` RX) kết nối máy tính PC qua module chuyển đổi USB-to-UART.
   - Tái định tuyến hàm `printf()` chuẩn trong thư viện C `stdio.h` sang kênh UART của vi điều khiển STM32F407.
   - Xây dựng máy phân tích gói lệnh (Command Parser) nhận lệnh từ Terminal máy tính: Báo cáo trạng thái ('s'), kích hoạt báo động ('a'), tắt báo động ('c'), và đổi mật mã hệ thống gồm chuỗi 4 chữ số (Ex 2.1 $\div$ 2.5).
3. **Đo kiểm Thực tế:**
   - Dùng thiết bị phân tích logic (Logic Analyzer) bắt dạng sóng đường truyền TX và RX.
   - Giải mã (Decode) từng bit tín hiệu thực tế trên phần mềm và so sánh với lý thuyết.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Tên thiết bị / Module | Thông số kỹ thuật / Mã hiệu | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Mạch chuyển đổi USB-to-UART | Chip CH340G / CP2102 / FT232RL | 1 | Giao tiếp với cổng USB máy tính |
| 3 | LED chỉ thị | LED 5mm xanh/đỏ + điện trở $330\Omega$ | 2 | Chỉ thị trạng thái Báo động (`PD12`) |
| 4 | Phần mềm Terminal trên PC | Hercules Setup, Tera Term, hoặc PuTTY | 1 | Cấu hình 115200 bps, 8N1 |
| 5 | Thiết bị đo | Logic Analyzer 8CH 24MHz (PulseView) | 1 | Bắt dạng sóng và giải mã UART |

---

## 3. CƠ SỞ TÍNH TOÁN VÀ SƠ ĐỒ ĐẤU NỐI MẠCH

### 3.1. Tính toán giá trị thanh ghi `USART_BRR` (Baud Rate Register)
Trên STM32F407, khối `USART2` nằm trên bus ngoại vi **APB1** với tần số xung nhịp $f_{PCLK1} = 42\text{ MHz}$.  
Công thức tính hệ số chia `USARTDIV` trong chế độ lấy mẫu 16 lần (`OVER8 = 0`):
$$\text{USARTDIV} = \frac{f_{PCLK1}}{16 \times \text{BaudRate}} = \frac{42{,}000{,}000}{16 \times 115{,}200} = \frac{42{,}000{,}000}{1{,}843{,}200} \approx 22.786458$$

Tách phần nguyên và phần phân số của `USARTDIV`:
- **Phần nguyên (Mantissa):**
  $$\text{DIV\_Mantissa} = \lfloor 22.786458 \rfloor = 22 = 0x16$$
- **Phần phân số (Fraction):**
  $$\text{DIV\_Fraction} = \text{round}(0.786458 \times 16) = \text{round}(12.5833) = 13 = 0x0D$$
- **Giá trị ghi vào thanh ghi `USART2->BRR`:**
  $$\text{USART\_BRR} = (0x16 \ll 4) \mid 0x0D = 0x016D$$

**Tính sai số thực tế:**
$$\text{BaudRate}_{\text{Actual}} = \frac{42{,}000{,}000}{16 \times (22 + \frac{13}{16})} = \frac{42{,}000{,}000}{365} \approx 115{,}068.5\text{ bps}$$
$$\text{Sai số} = \frac{|115{,}068.5 - 115{,}200|}{115{,}200} \times 100\% = 0.114\%$$
*(Sai số $0.114\% \ll 2.5\%$ cho phép, đường truyền đạt độ tin cậy tuyệt đối).*

---

### 3.2. Bảng đấu nối dây chi tiết
| Chân STM32F407 | Chân Module USB-UART | Chức năng tín hiệu | Lưu ý an toàn |
| :---: | :---: | :--- | :--- |
| `PA2` (USART2_TX) | `RXD` | Vi điều khiển truyền dữ liệu lên PC | Đấu chéo TX $\rightarrow$ RX |
| `PA3` (USART2_RX) | `TXD` | Máy tính gửi dữ liệu xuống vi điều khiển | Đấu chéo RX $\leftarrow$ TX |
| `GND` | `GND` | Nối chung mass hệ thống | **Bắt buộc nối chung mass!** |
| `PD12` | Anode LED đỏ | Đèn chỉ thị trạng thái Báo động | Qua trở hạn dòng $330\Omega$ |

> [!WARNING]
> Tuyệt đối **KHÔNG NỐI** chân `5V` hoặc `VCC` của module USB-UART vào chân `3V3` của kit STM32F407 để tránh gây xung đột điện áp và phá hỏng vi điều khiển! Chỉ kết nối 3 đường tín hiệu: **TX, RX, và GND**.

---

## 4. QUY TRÌNH CẤU HÌNH TRÊN STM32CubeMX
1. **Clock Configuration:** Cấu hình HSE $8\text{ MHz}$, SYSCLK $168\text{ MHz}$, APB1 $42\text{ MHz}$.
2. **Cấu hình USART2:**
   - Vào mục *Connectivity* $\rightarrow$ *USART2*.
   - Mode: **Asynchronous** (Không đồng bộ).
   - Parameter Settings:
     - Baud Rate: **115200 Bits/s**.
     - Word Length: **8 Bits (including Parity)**.
     - Parity: **None**.
     - Stop Bits: **1**.
     - Over Sampling: **16 Samples**.
3. **Cấu hình GPIO:**
   - Chân `PA2` tự động là `USART2_TX` (Alternate Function).
   - Chân `PA3` tự động là `USART2_RX` (Alternate Function).
   - Chân `PD12`: **GPIO_Output** (LED Alarm).

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### 5.1. Tái định tuyến `printf` sang `USART2`
Thêm vào khối `/* USER CODE BEGIN 0 */` trong file `Core/Src/main.c`:

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>
#include <string.h>

extern UART_HandleTypeDef huart2;

/* Định nghĩa lại hàm gửi ký tự cấp thấp cho thư viện printf */
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 0 */
```

### 5.2. Lập trình máy phân tích lệnh (Command Parser - Ex 2.1 $\div$ 2.5)
Tại vòng lặp chính `main.c`:

```c
/* USER CODE BEGIN 2 */
/* Mật mã bảo mật mặc định ban đầu của Smart Home: 1 2 3 4 */
static char systemPIN[5] = "1234";
static uint8_t isAlarmActive = 0;

printf("\r\n==============================================\r\n");
printf("  SMART HOME SYSTEM - STM32F407 CONSOLE READY \r\n");
printf("==============================================\r\n");
printf("Menu lenh:\r\n");
printf(" [s] : Xem trang thai he thong (Status)\r\n");
printf(" [a] : Kich hoat bao dong khan cap (Activate Alarm)\r\n");
printf(" [c] : Tat bao dong (Clear Alarm)\r\n");
printf(" [p] : Thay doi ma PIN he thong (Change PIN)\r\n");
printf("Nhap lua chon cua ban: ");
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    uint8_t rxCommand;
    
    /* Chờ nhận 1 ký tự lệnh từ PC với thời gian chờ 500ms */
    if (HAL_UART_Receive(&huart2, &rxCommand, 1, 500) == HAL_OK)
    {
        /* Phản hồi lại ký tự vừa nhập (Echo) lên màn hình */
        printf("%c\r\n", rxCommand);

        switch (rxCommand)
        {
            case 's':
            case 'S':
                printf("[STATUS] Trang thai bao dong: %s\r\n", 
                       isAlarmActive ? "DANG BAO DONG (DANGER)" : "AN TOAN (SAFE)");
                printf("[STATUS] Ma PIN hien tai: %s\r\n", systemPIN);
                break;

            case 'a':
            case 'A':
                isAlarmActive = 1;
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
                printf("[ALERT] Da kich hoat bao dong! LED PD12 BAT.\r\n");
                break;

            case 'c':
            case 'C':
                isAlarmActive = 0;
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
                printf("[OK] Da tat bao dong! LED PD12 TAT.\r\n");
                break;

            case 'p':
            case 'P':
            {
                printf("Vui long nhap ma PIN moi (dung 4 chu so): ");
                char newPIN[5] = {0};
                /* Nhận liên tục 4 ký tự từ PC */
                if (HAL_UART_Receive(&huart2, (uint8_t *)newPIN, 4, 5000) == HAL_OK)
                {
                    newPIN[4] = '\0';
                    strncpy(systemPIN, newPIN, 4);
                    printf("\r\n[THANH CONG] Ma PIN da duoc cap nhat thanh: %s\r\n", systemPIN);
                }
                else
                {
                    printf("\r\n[LOI] Het thoi gian nhap (Timeout)!\r\n");
                }
                break;
            }

            default:
                printf("[LOI] Lenh khong hop le. Vui long nhap s, a, c, hoac p.\r\n");
                break;
        }
        printf("\r\nNhap lua chon tiep theo: ");
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & PHÂN TÍCH DẠNG SÓNG BẰNG LOGIC ANALYZER

### 6.1. Thiết lập Logic Analyzer (Saleae / PulseView)
1. Cắm kênh `CH0` của Logic Analyzer vào chân `PA2` (STM32 TX).
2. Cắm kênh `CH1` của Logic Analyzer vào chân `PA3` (STM32 RX).
3. Kẹp kẹp mass của Logic Analyzer vào chân `GND` của STM32F407.
4. Mở phần mềm PulseView:
   - Thêm bộ giải mã giao thức (Add Protocol Decoder) $\rightarrow$ Chọn **UART**.
   - Cấu hình decoder cho CH0: Baud rate = **115200**, Data bits = **8**, Parity = **None**, Stop bits = **1**, Bit order = **LSB-first**.

### 6.2. Phân tích khung truyền ký tự 'A' (Mã ASCII $0x41$)
1. Khi máy tính gửi ký tự 'A' ($0x41 = 01000001_2$):
   - Mức logic nghỉ (Idle) của đường truyền: Mức cao $3.3\text{V}$ (Logic 1).
   - **Start bit:** Sườn xuống về mức 0 trong đúng 1 chu kỳ bit ($T_{bit} = 1/115200 \approx 8.68\mu s$).
   - **Data bits (LSB trước):**
     - Bit 0: `1` ($8.68\mu s$)
     - Bit 1: `0`
     - Bit 2: `0`
     - Bit 3: `0`
     - Bit 4: `0`
     - Bit 5: `0`
     - Bit 6: `1`
     - Bit 7: `0`
   - **Stop bit:** Kéo lên mức cao $3.3\text{V}$ tối thiểu $8.68\mu s$.

```
         Start   B0  B1  B2  B3  B4  B5  B6  B7   Stop
Idle: ___     ___     ___ ___ ___ ___     ___     _________
     1   \___/ 1 \___/ 0   0   0   0 \___/ 1 \___/ 1 (Stop)
          0
         |<-------- 8.68 us x 10 bits = 86.8 us -------->|
```

---

## 7. XỬ LÝ SỰ CỐ & CƠ CHẾ "UNDER THE HOOD"

### Chẩn đoán lỗi phần cứng UART
Sinh viên quan sát thanh ghi trạng thái `USART2->SR` khi xảy ra sự cố:
1. **Lỗi Overrun Error (`ORE` bit 3 trong `USART2->SR`):**
   - *Hiện tượng:* Vi điều khiển bị mất dữ liệu khi máy tính gửi một chuỗi văn bản dài liên tục.
   - *Cơ chế:* Thanh ghi dịch `RDR` vừa nhận xong một byte mới trong khi CPU chưa kịp đọc byte trước đó ra khỏi bộ đệm.
   - *Cách khắc phục:* Đọc thanh ghi `USART2->SR` sau đó đọc `USART2->DR` để xóa cờ ORE; hoặc nâng cấp sang chế độ nhận bằng ngắt (Interrupt) / DMA.
2. **Ký tự bị biến dạng thành rác (Garbled text/Gibberish):**
   - *Nguyên nhân:* Lệch tốc độ Baudrate giữa máy tính và STM32 (thường do cấu hình sai xung nhịp APB1 hoặc thạch anh ngoài).

---

## 8. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 8.1. Yêu cầu sản phẩm nộp
- Bản in log giao tiếp Terminal giữa máy tính và STM32 thực hiện đầy đủ 4 lệnh.
- Ảnh chụp màn hình Logic Analyzer giải mã thành công chuỗi ký tự mật mã mới.
- Bảng tính giá trị thanh ghi `USART_BRR` nếu đổi sang Baudrate 9600 bps và 57600 bps.

### 8.2. Câu hỏi đánh giá
1. *Tại sao UART quy định truyền bit LSB (Least Significant Bit) trước thay vì MSB?*
2. *Nếu tần số xung nhịp bus APB1 thay đổi do thay đổi chế độ tiết kiệm năng lượng, tốc độ Baudrate của USART2 có bị ảnh hưởng không? Làm sao để phần mềm tự thích ứng?*
3. *Giải thích tại sao trong hàm `HAL_UART_Receive` kiểu Polling, nếu người dùng không gõ phím trên PC thì CPU sẽ bị "chặn" (Block) lại cho đến hết thời gian Timeout? Giải pháp khắc phục là gì?*