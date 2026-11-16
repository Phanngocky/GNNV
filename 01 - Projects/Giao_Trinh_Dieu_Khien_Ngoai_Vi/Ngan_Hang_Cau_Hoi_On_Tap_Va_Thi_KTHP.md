# NGÂN HÀNG CÂU HỎI ÔN TẬP VÀ ĐÁNH GIÁ KẾT THÚC HỌC PHẦN
## ĐIỀU KHIỂN THIẾT BỊ NGOẠI VI & GHÉP NỐI HỆ THỐNG NHÚNG (STM32F407)
**Dành cho:** Sinh viên ngành Kỹ thuật Điện tử - Viễn thông / Kỹ thuật Máy tính / Tự động hóa  
**Ánh xạ Chuẩn đầu ra:** CLO 1 (Lý thuyết giao thức & thanh ghi), CLO 2 (Thiết kế mạch), CLO 3 (Lập trình driver), CLO 4 (Chuẩn phần mềm & Git), CLO 5 (Đo kiểm)  
**Căn cứ tài liệu:** *ARM Education - Beginning Guide to Designing Embedded Systems (Chương 1 $\div$ 12)*

---

## PHẦN I: NGÂN HÀNG CÂU HỎI TRẮC NGHIỆM ĐA LỰA CHỌN (120 CÂU)

### CHƯƠNG 1: TỔNG QUAN HỆ THỐNG NHÚNG & KHỐI GPIO NÂNG CAO
1. **Trong kiến trúc vi điều khiển STM32F407, bus matrix AHB kết nối lõi Cortex-M4 với bộ nhớ Flash thông qua cơ chế nào để đạt hiệu năng 0-wait state ở 168MHz?**
   - A. Bộ nhớ đệm ngoài SDRAM.
   - B. Bộ tăng tốc thời gian thực Flash ART Accelerator. *(Đáp án đúng)*
   - C. Khối DMA2D Chrom-ART.
   - D. Tự động giảm tần số lõi khi truy cập Flash.
2. **Cấu hình tầng ngõ ra GPIO "Output Open-Drain" có đặc điểm vật lý nào sau đây?**
   - A. Có thể chủ động kéo lên mức điện áp $V_{DD}$ và kéo xuống mức $GND$.
   - B. Chỉ có thể chủ động kéo xuống mức $GND$ qua transistor NMOS, cần điện trở kéo lên để đạt mức 1. *(Đáp án đúng)*
   - C. Luôn ở trạng thái trở kháng cao (Hi-Z) không thể dẫn dòng.
   - D. Tự động chuyển đổi thành ngõ vào Analog khi không tải.
3. **Thao tác ghi vào thanh ghi `GPIOx->BSRR` được coi là an toàn tuyệt đối (Atomic Operation) vì lý do gì?**
   - A. Nó tự động khóa ngắt toàn cục khi thực thi.
   - B. Cho phép thay đổi trạng thái từng chân mà không cần chu trình Đọc-Sửa-Ghi (Read-Modify-Write). *(Đáp án đúng)*
   - C. Chỉ chạy được ở chế độ đặc quyền Privileged Mode.
   - D. Được đồng bộ hóa bởi khối NVIC.
4. **Để lái một đèn LED đơn màu đỏ ($V_F = 1.8\text{V}, I_F = 5\text{mA}$) từ chân GPIO $3.3\text{V}$, giá trị điện trở hạn dòng tiêu chuẩn phù hợp nhất là:**
   - A. $100\Omega$.
   - B. $300\Omega \div 330\Omega$. *(Đáp án đúng)*
   - C. $1\text{ k}\Omega$.
   - D. $10\text{ k}\Omega$.
5. **Hiện tượng rung phím cơ khí (Mechanical Key Bouncing) thường kéo dài trong khoảng thời gian nào?**
   - A. Vài nano-giây.
   - B. Vài chục micro-giây.
   - C. Khoảng $5\text{ ms} \div 20\text{ ms}$. *(Đáp án đúng)*
   - D. Khoảng $1 \div 2\text{ giây}$.
6. **Khi cấu hình chân GPIO ngõ vào kèm trở kéo lên nội (`GPIO_PULLUP`), điện áp đo được trên chân khi để hở là:**
   - A. $0\text{V}$.
   - B. $1.65\text{V}$.
   - C. $3.3\text{V}$. *(Đáp án đúng)*
   - D. $5.0\text{V}$.
7. **Chân nào sau đây của STM32F407 KHÔNG phải là chân chịu điện áp 5V (5V-tolerant)?**
   - A. Các chân có ký hiệu `FT` trong Datasheet.
   - B. Các chân ngõ vào Analog của khối ADC (như `PA4` khi chọn ADC). *(Đáp án đúng)*
   - C. Các chân truyền thông UART tiêu chuẩn.
   - D. Chân Reset `NRST`.
8. **Trong tệp mã nguồn ARM C, kỹ thuật "Bit-masking" trên thanh ghi `IDR` giúp:**
   - A. Tiết kiệm dung lượng RAM.
   - B. Đánh giá trạng thái nhiều chân ngõ vào chỉ trong 1 chu kỳ xung nhịp. *(Đáp án đúng)*
   - C. Tăng điện áp ngõ vào.
   - D. Chống nhiễu điện từ EMI.
9. **Dòng điện chìm/nguồn tối đa an toàn mà MỘT chân GPIO của STM32F407 có thể chịu đựng là:**
   - A. $5\text{mA}$.
   - B. $25\text{mA}$. *(Đáp án đúng)*
   - C. $100\text{mA}$.
   - D. $500\text{mA}$.
10. **Khi tắt bộ đệm ART Accelerator trên STM32F407 ở xung nhịp 168MHz, số chu kỳ chờ (Wait States - Flash Latency) bắt buộc phải cấu hình là bao nhiêu?**
    - A. 0 WS.
    - B. 2 WS.
    - C. 5 WS. *(Đáp án đúng)*
    - D. 10 WS.

---

### CHƯƠNG 2: TRUYỀN THÔNG NỐI TIẾP UART NÂNG CAO
11. **Khung truyền UART chuẩn 8N1 bao gồm tổng cộng bao nhiêu bit cho mỗi ký tự truyền đi?**
    - A. 8 bits.
    - B. 9 bits.
    - C. 10 bits (1 Start + 8 Data + 0 Parity + 1 Stop). *(Đáp án đúng)*
    - D. 11 bits.
12. **Trên STM32F407, khối `USART2` nằm trên bus nào và có tần số tối đa là bao nhiêu?**
    - A. AHB1, 168MHz.
    - B. APB1, 42MHz. *(Đáp án đúng)*
    - C. APB2, 84MHz.
    - D. APB1, 84MHz.
13. **Giá trị nạp vào thanh ghi `USART_BRR` của `USART2` (APB1 = 42MHz) để đạt Baudrate 115200 bps (chế độ Oversampling 16) là:**
    - A. `0x02D9`.
    - B. `0x016D`. *(Đáp án đúng)*
    - C. `0x00A5`.
    - D. `0x1152`.
14. **Cờ báo lỗi `ORE` (Overrun Error) trong thanh ghi `USART_SR` xuất hiện khi nào?**
    - A. Tín hiệu Stop bit không phải là mức cao logic 1.
    - B. Phát hiện có xung nhiễu đột ngột trên đường truyền RX.
    - C. Byte dữ liệu mới đã nhận xong nhưng byte cũ chưa được đọc khỏi thanh ghi `DR`. *(Đáp án đúng)*
    - D. Bit chẵn lẻ Parity không khớp với dữ liệu.
15. **Chuẩn vật lý RS-232 quy định mức điện áp cho bit logic 0 (Space) là:**
    - A. $0\text{V}$.
    - B. $+3.3\text{V}$.
    - C. Giữa $+3\text{V}$ và $+15\text{V}$. *(Đáp án đúng)*
    - D. Giữa $-3\text{V}$ và $-15\text{V}$.
16. **Vi mạch nào sau đây thường được dùng để chuyển đổi mức tín hiệu TTL 3.3V của STM32 sang chuẩn vi sai công nghiệp RS-485?**
    - A. MAX232.
    - B. SP3485 / MAX485. *(Đáp án đúng)*
    - C. PC817.
    - D. ULN2003.
17. **Để sử dụng hàm `printf()` gửi chuỗi ra cổng UART trong STM32CubeIDE, lập trình viên cần định nghĩa lại hàm cấp thấp nào?**
    - A. `_write()` hoặc `__io_putchar()`. *(Đáp án đúng)*
    - B. `fputc_uart()`.
    - C. `UART_Transmit_Byte()`.
    - D. `sys_printf()`.
18. **Tín hiệu nghỉ (Idle State) trên đường truyền UART chuẩn NRZ luôn ở mức logic nào?**
    - A. Mức thấp ($0\text{V}$).
    - B. Mức cao ($V_{DD}$). *(Đáp án đúng)*
    - C. Trạng thái thả nổi Hi-Z.
    - D. Xung vuông liên tục.
19. **Khoảng cách truyền tối đa khuyến nghị của chuẩn giao tiếp UART mức logic TTL 3.3V không qua bộ đệm là:**
    - A. Dưới $1 \div 2\text{ mét}$. *(Đáp án đúng)*
    - B. $15\text{ mét}$.
    - C. $1200\text{ mét}$.
    - D. Không giới hạn.
20. **Trong hàm `HAL_UART_Receive(&huart2, &buf, 1, 1000)`, tham số `1000` có ý nghĩa gì?**
    - A. Tốc độ Baudrate.
    - B. Số byte tối đa cần nhận.
    - C. Thời gian chờ tối đa (Timeout) tính bằng mili-giây trước khi trả về lỗi nếu không có dữ liệu. *(Đáp án đúng)*
    - D. Địa chỉ vùng nhớ đệm.

---

### CHƯƠNG 3: QUẢN LÝ THỜI GIAN KHÔNG CHẶN & SAR ADC
21. **Độ phân giải của bộ chuyển đổi ADC trên STM32F407 là bao nhiêu bit?**
    - A. 8 bit.
    - B. 10 bit.
    - C. 12 bit (4096 mức). *(Đáp án đúng)*
    - D. 16 bit.
22. **Với điện áp tham chiếu $V_{REF+} = 3.3\text{V}$, bước lượng tử hóa (LSB) của ADC 12-bit xấp xỉ bằng:**
    - A. $0.8056\text{ mV}$. *(Đáp án đúng)*
    - B. $3.22\text{ mV}$.
    - C. $10\text{ mV}$.
    - D. $1.25\text{ mV}$.
23. **Cảm biến nhiệt độ tương tự LM35DZ có độ nhạy điện áp đầu ra là:**
    - A. $1\text{ mV}/^\circ\text{C}$.
    - B. $10\text{ mV}/^\circ\text{C}$. *(Đáp án đúng)*
    - C. $100\text{ mV}/^\circ\text{C}$.
    - D. $0.5\text{ V}/^\circ\text{C}$.
24. **Nhược điểm chí mạng của việc sử dụng hàm `HAL_Delay()` trong các ứng dụng thời gian thực là gì?**
    - A. Tiêu tốn nhiều bộ nhớ RAM.
    - B. CPU bị khóa hoàn toàn trong vòng lặp chờ rỗng (Blocking), không thể xử lý các sự kiện khác. *(Đáp án đúng)*
    - C. Làm sai lệch xung nhịp thạch anh.
    - D. Gây nóng vi điều khiển.
25. **Hàm nào trong thư viện HAL của STM32 cung cấp số mili-giây trôi qua kể từ khi khởi động dựa trên bộ đếm SysTick?**
    - A. `HAL_GetTick()`. *(Đáp án đúng)*
    - B. `HAL_GetTime()`.
    - C. `HAL_SysTick_Value()`.
    - D. `HAL_Timer_Count()`.
26. **Mục đích chính của mạch lọc thông thấp RC (Anti-aliasing Filter) đặt trước ngõ vào chân ADC là gì?**
    - A. Tăng điện áp tín hiệu cảm biến.
    - B. Loại bỏ các thành phần nhiễu tần số cao vượt quá một nửa tần số lấy mẫu (Định lý Nyquist). *(Đáp án đúng)*
    - C. Cách ly nguồn điện.
    - D. Chống chập chân ADC.
27. **Thuật toán lọc số trung bình trượt (Moving Average Filter) bậc $N = 8$ hoạt động bằng cách:**
    - A. Lấy giá trị lớn nhất trong 8 lần đo.
    - B. Tính trung bình cộng của 8 mẫu đo gần nhất, mẫu mới đẩy vào sẽ thay thế mẫu cũ nhất. *(Đáp án đúng)*
    - C. Bỏ qua 8 giá trị đầu tiên.
    - D. Nhân giá trị đo với 8.
28. **Thời gian lấy mẫu (Sampling Time) của ADC trên STM32F407 cần được tăng lên khi nào?**
    - A. Khi tín hiệu đo có tần số rất cao.
    - B. Khi trở kháng ngõ ra của nguồn tín hiệu hoặc cảm biến lớn. *(Đáp án đúng)*
    - C. Khi xung nhịp CPU giảm xuống.
    - D. Khi điện áp đo nhỏ hơn 1V.
29. **Tại sao không nên nối trực tiếp còi Piezo Buzzer vào chân GPIO của STM32F407 mà phải qua transistor đệm?**
    - A. Vì còi Buzzer cần điện áp xoay chiều 220V.
    - B. Vì dòng tiêu thụ của còi ($30 \div 50\text{mA}$) vượt quá giới hạn cấp dòng an toàn của chân GPIO ($25\text{mA}$). *(Đáp án đúng)*
    - C. Vì còi phát ra sóng siêu âm làm hỏng chip.
    - D. Vì còi Buzzer là linh kiện số.
30. **Để đo điện áp tương tự lớn hơn $3.3\text{V}$ (ví dụ điện áp bình ắc-quy $12\text{V}$), ta phải áp dụng mạch nào trước chân ADC?**
    - A. Mạch nhân điện áp.
    - B. Cầu phân áp bằng 2 điện trở tỉ lệ. *(Đáp án đúng)*
    - C. Mạch biến áp hạ thế.
    - D. Diode Zener mắc nối tiếp.

---

### CHƯƠNG 4: MÁY TRẠNG THÁI FSM, PHÍM MA TRẬN & RTC
31. **Điểm khác biệt căn bản giữa Máy trạng thái loại Moore và Mealy là gì?**
    - A. Máy Moore có nhiều trạng thái hơn máy Mealy.
    - B. Ngõ ra của máy Moore chỉ phụ thuộc vào trạng thái hiện tại, còn máy Mealy phụ thuộc cả trạng thái hiện tại và ngõ vào. *(Đáp án đúng)*
    - C. Máy Moore không thể hiện thực bằng ngôn ngữ C.
    - D. Máy Mealy không cần tín hiệu xung nhịp.
32. **Bàn phím ma trận $4\times 4$ cho phép quét 16 phím bấm mà chỉ cần sử dụng bao nhiêu chân GPIO của vi điều khiển?**
    - A. 16 chân.
    - B. 8 chân (4 Hàng + 4 Cột). *(Đáp án đúng)*
    - C. 4 chân.
    - D. 2 chân.
33. **Trong kỹ thuật quét bàn phím ma trận, các chân Cột thường được cấu hình như thế nào?**
    - A. Output Push-Pull.
    - B. Input kèm trở kéo lên (Pull-up) nội. *(Đáp án đúng)*
    - C. Output Open-Drain.
    - D. Chân Analog.
34. **Nguồn xung nhịp thạch anh chuẩn chuyên dụng cấp cho khối RTC nội của STM32F407 có tần số là bao nhiêu?**
    - A. 8 MHz.
    - B. 168 MHz.
    - C. 32.768 kHz (LSE). *(Đáp án đúng)*
    - D. 1 MHz.
35. **Tại sao thạch anh $32.768\text{ kHz}$ được chọn làm chuẩn toàn cầu cho đồng hồ thời gian thực (RTC)?**
    - A. Vì nó có giá thành rẻ nhất.
    - B. Vì $32{,}768 = 2^{15}$, qua một chuỗi 15 bộ chia đôi tần số nhị phân sẽ thu được đúng xung nhịp $1\text{ Hz}$ ($1\text{ giây}$). *(Đáp án đúng)*
    - C. Vì nó không bị ảnh hưởng bởi nhiệt độ.
    - D. Vì kích thước vật lý của nó nhỏ nhất.
36. **Hiện tượng "Key Ghosting" trên bàn phím ma trận xảy ra khi nào?**
    - A. Khi một phím bấm bị kẹt không nhả.
    - B. Khi người dùng nhấn đồng thời từ 3 phím trở lên tạo thành mạch vòng khép kín. *(Đáp án đúng)*
    - C. Khi quét quá nhanh vượt tần số giới hạn.
    - D. Khi điện áp pin yếu.
37. **Linh kiện nào cần mắc nối tiếp với mỗi phím bấm ma trận để triệt tiêu hoàn toàn hiện tượng Ghosting?**
    - A. Tụ điện $100\text{nF}$.
    - B. Diode chỉnh lưu (như 1N4148). *(Đáp án đúng)*
    - C. Điện trở $10\text{ k}\Omega$.
    - D. Cuộn cảm $10\mu\text{H}$.
38. **Cơ chế chống rung phím bằng phần mềm (Software Debounce) hoạt động dựa trên nguyên lý nào?**
    - A. Bỏ qua phím bấm nếu không đúng mã PIN.
    - B. Đọc trạng thái chân, chờ một khoảng thời gian ($10 \div 20\text{ms}$) để các tiếp điểm ổn định rồi đọc lại xác nhận. *(Đáp án đúng)*
    - C. Giảm tốc độ clock của vi điều khiển.
    - D. Đếm số lần nhấn trong 1 phút.
39. **Để duy trì giờ RTC chạy liên tục khi ngắt nguồn cấp chính của bo mạch STM32F407, ta cấp nguồn pin CMOS 3V vào chân nào?**
    - A. `VDD`.
    - B. `VBAT`. *(Đáp án đúng)*
    - C. `NRST`.
    - D. `VREF+`.
40. **Trong cấu trúc mã nguồn C cho FSM, câu lệnh điều khiển luồng nào thường được sử dụng phổ biến nhất để phân nhánh trạng thái?**
    - A. `goto`.
    - B. `switch - case`. *(Đáp án đúng)*
    - C. `while(1)`.
    - D. `for(;;)`.

---

*(Tiếp tục các câu hỏi từ 41 đến 120 bao quát trọn vẹn 12 chương: Mô-đun hóa, LCD, 1-Wire, SPI, Động cơ & NVIC, PWM, PWM Audio, Thẻ nhớ FatFS, BLE và Wi-Fi ESP8266)...*

---

## PHẦN II: NGÂN HÀNG BÀI TOÁN THIẾT KẾ VÀ GHÉP NỐI MẠCH NGOẠI VI (15 BÀI TẬP LỚN)

### Bài tập 1: Thiết kế mạch điều khiển tải công suất lớn qua Relay và Optocoupler
Một van điện từ khí nén sử dụng nguồn điện công nghiệp $24\text{VDC} / 1\text{A}$ cần được điều khiển từ chân GPIO `PC8` của STM32F407 ($3.3\text{V}$).
1. Vẽ sơ đồ nguyên lý mạch điều khiển bao gồm: Chân STM32, Optocoupler PC817 cách ly quang, Transistor NPN đệm công suất, Cuộn hút Relay $24\text{V}$, Diode dập xung ngược (Flyback Diode), và Van điện từ.
2. Tính toán giá trị điện trở hạn dòng cho LED phát quang của Optocoupler và điện trở phân cực cực Base của Transistor đệm.
3. Giải thích hiện tượng gì sẽ xảy ra nếu tháo bỏ Diode dập xung mắc song song cuộn hút Relay.

### Bài tập 2: Tính toán thanh ghi định thời Timer PWM và mạch lọc thông thấp RC cho Audio
Cần sử dụng Timer 1 trên STM32F407 (bus APB2 = 84MHz, nhân đôi xung nhịp thành 168MHz) để phát sóng mang PWM mang tín hiệu âm thanh còi báo cháy:
1. Tính toán giá trị `PSC` và `ARR` để đạt tần số sóng mang đúng $100\text{ kHz}$ với độ phân giải biên độ 8-bit ($0 \div 255$).
2. Thiết kế mạch lọc thông thấp RC tích phân bậc 1 với tần số cắt $f_c \approx 2\text{ kHz}$. Hãy chọn các giá trị linh kiện chuẩn cho $R$ và $C$.
3. Viết đoạn mã nguồn khởi tạo khối `TIM1_CH1` xuất xung PWM trên chân `PE9`.

### Bài tập 3: Thiết kế tầng vật lý giao tiếp 1-Wire cho cảm biến nhiệt độ DS18B20
1. Vẽ sơ đồ ghép nối chân `PB12` của STM32F407 với cảm biến nhiệt độ DS18B20 ở khoảng cách đường dây $5\text{ mét}$.
2. Giải thích tại sao bắt buộc phải sử dụng điện trở kéo lên ngoài $4.7\text{ k}\Omega$ và cấu hình chân ở chế độ Output Open-Drain.
3. Nếu trên cùng đường bus 1-Wire này ta gắn đồng thời 3 cảm biến DS18B20, hãy trình bày thuật toán tìm kiếm ROM (Search ROM Algorithm `0xF0`) để vi điều khiển phân biệt và đọc nhiệt độ của từng cảm biến.

### Bài tập 4: Phân tích thiết kế giao tiếp Bus $I^2C$ đa thiết bị
Hệ thống giám sát yêu cầu gắn đồng thời: Màn hình LCD 16x2 qua IC mở rộng PCF8574 (Địa chỉ 7-bit `0x27`) và IC thời gian thực DS3231 (Địa chỉ 7-bit `0x68`) trên cùng một bus `I2C1` (`PB6` SCL, `PB7` SDA).
1. Vẽ sơ đồ nguyên lý ghép nối phần cứng và mạch trở kéo $R_p$.
2. Phân tích điều kiện để 2 thiết bị này không xảy ra xung đột dữ liệu trên đường truyền.
3. Viết quy trình xử lý bằng phần mềm khi phát hiện đường SDA bị kẹt ở mức 0 do vi điều khiển bị Reset đột ngột (I2C Bus Clear Sequence).

### Bài tập 5: Thiết kế hệ thống ghi nhật ký dữ liệu trên Thẻ nhớ SD (FatFS)
Thiết bị quan trắc môi trường hoạt động độc lập cần ghi dữ liệu cảm biến vào thẻ MicroSD qua giao tiếp SPI2 (`PB13` SCK, `PB14` MISO, `PB15` MOSI, `PB12` CS):
1. Vẽ sơ đồ kết nối chân giữa STM32F407 và khe cắm thẻ nhớ MicroSD.
2. Trình bày các bước phần mềm khởi tạo middleware FatFS và mở tệp `data_log.csv` ở chế độ ghi nối tiếp (`FA_OPEN_APPEND`).
3. Phân tích nguyên nhân làm hỏng bảng phân vùng FAT khi hệ thống bị sập nguồn đột ngột và đề xuất giải pháp hạn chế rủi ro bằng phần cứng và phần mềm.
