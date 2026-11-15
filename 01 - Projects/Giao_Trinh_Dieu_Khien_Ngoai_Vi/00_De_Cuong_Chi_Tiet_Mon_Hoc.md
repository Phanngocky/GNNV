# ĐỀ CƯƠNG CHI TIẾT HỌC PHẦN: ĐIỀU KHIỂN THIẾT BỊ NGOẠI VI (GNNV)
**Chuyên ngành:** Kỹ thuật Điện tử / Kỹ thuật Điện tử - Viễn thông / Kỹ thuật Máy tính / Cơ điện tử  
**Trình độ đào tạo:** Đại học chính quy  
**Nền tảng thực hành:** Vi điều khiển 32-bit ARM Cortex-M4 (STM32F4 Series - STM32F401/F407/F429)  
**Môi trường phát triển:** STM32CubeIDE / Keil MDK ARM + STM32CubeMX  

---

## I. THÔNG TIN CHUNG VỀ HỌC PHẦN
- **Mã học phần:** EE3042 (Dự kiến)
- **Tên học phần tiếng Việt:** Điều khiển Thiết bị Ngoại vi & Ghép nối Vi xử lý
- **Tên học phần tiếng Anh:** Microcontroller Peripherals and Interfacing Techniques
- **Số tín chỉ:** 3 tín chỉ (2 tín chỉ Lý thuyết: 30 tiết + 1 tín chỉ Thực hành/Thí nghiệm: 30 tiết)
- **Học phần tiên quyết:** 
  - Kỹ thuật Lập trình C/C++
  - Mạch điện tử 1 & 2
  - Kỹ thuật Vi xử lý - Vi điều khiển cơ bản (8051 / AVR / PIC hoặc kiến trúc máy tính)

---

## II. MỤC TIÊU HỌC PHẦN (COURSE OBJECTIVES - COs)
Học phần trang bị cho sinh viên ngành Điện tử:
1. **CO1 (Kiến thức):** Hiểu rõ bản chất phần cứng, sơ đồ khối, cơ chế quản lý bus (AHB/APB), không gian bộ nhớ và nguyên lý điều khiển của các khối ngoại vi tích hợp trên dòng vi điều khiển 32-bit hiện đại (ARM Cortex-M4).
2. **CO2 (Kỹ năng thiết kế phần cứng):** Có khả năng đọc hiểu tài liệu kỹ thuật (Datasheet, Reference Manual, Application Notes), tính toán thông số ghép nối mạch giao tiếp ngoại vi (điện trở hạn dòng, mạch đệm, mạch cách ly quang, phối hợp mức điện áp 3.3V - 5V, bảo vệ chống xung ngược).
3. **CO3 (Kỹ năng lập trình nhúng):** Thành thạo lập trình điều khiển thiết bị ngoại vi bằng ngôn ngữ C thông qua việc thao tác trực tiếp trên thanh ghi (Register-level / Bare-metal) và sử dụng thư viện chuẩn của hãng (STM32 HAL / LL Drivers).
4. **CO4 (Kỹ năng đo kiểm & Debug):** Sử dụng thành thạo các thiết bị đo kiểm trong phòng lab điện tử (Máy hiện sóng Oscilloscope, Bộ phân tích logic Logic Analyzer, Đồng hồ vạn năng DMM, Mạch nạp/gỡ lỗi ST-Link) để bắt xung, đo tín hiệu và phân tích lỗi giao thức ngoại vi.
5. **CO5 (Tích hợp hệ thống):** Năng lực phân tích, thiết kế và phát triển một hệ thống nhúng hoàn chỉnh ứng dụng trong đo lường, giám sát môi trường, điều khiển công nghiệp hoặc Internet of Things (IoT).

---

## III. CHUẨN ĐẦU RA HỌC PHẦN (COURSE LEARNING OUTCOMES - CLOs)
Theo thang đo Bloom và chuẩn kiểm định quốc tế (CDIO / ABET):

| Mã CLO | Mô tả Chuẩn đầu ra | Mức Bloom | Chuẩn CDIO |
| :--- | :--- | :---: | :---: |
| **CLO 1** | Trình bày được kiến trúc lõi Cortex-M4, cơ chế phân bổ bus, hệ thống xung nhịp Clock Tree và bản đồ bộ nhớ ngoại vi STM32F4. | C2 (Hiểu) | 1.2, 1.3 |
| **CLO 2** | Tính toán, cấu hình và lập trình thành thạo các cổng xuất nhập GPIO để điều khiển LED, relay, còi báo, ma trận phím, nút nhấn chống rung. | C3 (Áp dụng) | 2.1, 3.1 |
| **CLO 3** | Thiết kế giải pháp xử lý ngắt theo thời gian thực sử dụng khối quản lý ngắt lồng nhau NVIC và ngắt ngoài EXTI. | C3 (Áp dụng) | 2.1, 3.2 |
| **CLO 4** | Lập trình các bộ định thời (Timer/Counter) để định thời chính xác, đo tần số, đo độ rộng xung (Input Capture) và phát xung PWM điều khiển động cơ, công suất. | C4 (Phân tích) | 2.2, 3.2 |
| **CLO 5** | Cấu hình và lập trình bộ biến đổi ADC (Single, Continuous, Scan mode) và DAC để thu thập dữ liệu từ cảm biến tương tự và tạo tín hiệu analog. | C4 (Phân tích) | 2.1, 3.2 |
| **CLO 6** | Làm chủ các chuẩn truyền thông công nghiệp và ghép nối vi mạch: UART, SPI, I2C, CAN Bus; phân tích được dạng sóng tín hiệu trên máy hiện sóng / Logic Analyzer. | C4 (Phân tích) | 2.2, 3.3 |
| **CLO 7** | Ứng dụng bộ truy cập bộ nhớ trực tiếp (DMA) để tăng tốc độ truyền nhận dữ liệu ngoại vi và giải phóng CPU. | C4 (Phân tích) | 2.3, 3.2 |
| **CLO 8** | Thiết kế và tích hợp thành công một sản phẩm nhúng điều khiển ngoại vi đa chức năng (Mini Capstone Project) theo nhóm, báo cáo kỹ thuật chuẩn mực. | C5 (Đánh giá) & C6 (Sáng tạo) | 4.3, 4.4, 4.5 |

---

## IV. CẤU TRÚC CHI TIẾT NỘI DUNG GIẢNG DẠY (11 CHƯƠNG)

### CHƯƠNG 1: KIẾN TRÚC VI ĐIỀU KHIỂN ARM CORTEX-M4 VÀ HỆ THỐNG GHÉP NỐI NGOẠI VI (STM32F4)
- 1.1. Giới thiệu kiến trúc ARM Cortex-M4 và các đặc tính vi điều khiển 32-bit.
- 1.2. Sơ đồ khối tổng thể họ vi điều khiển STM32F4 (STM32F401, STM32F407, STM32F429).
- 1.3. Hệ thống ma trận Bus (Multi-AHB Bus Matrix): Cấu trúc I-Bus, D-Bus, S-Bus, AHB1, AHB2, APB1, APB2.
- 1.4. Không gian địa chỉ (Memory Mapping) và ánh xạ thanh ghi ngoại vi.
- 1.5. Cây phân phối xung nhịp (Clock Tree & RCC - Reset and Clock Control): Nguồn dao động HSI, HSE, PLL, phân chia tần số bus.
- 1.6. Môi trường phát triển phần mềm: Keil MDK, STM32CubeIDE, công cụ STM32CubeMX, thư viện CMSIS, HAL và LL.

### CHƯƠNG 2: ĐIỀU KHIỂN CỔNG XUẤT NHẬP ĐA DỤNG (GPIO) & GHÉP NỐI THIẾT BỊ CƠ BẢN
- 2.1. Cấu trúc phần cứng một chân I/O trên STM32F4 (Input Floating, Pull-up, Pull-down, Output Push-Pull, Open-Drain, Alternate Function, Analog).
- 2.2. Các thanh ghi điều khiển GPIO: `MODER`, `OTYPER`, `OSPEEDR`, `PUPDR`, `IDR`, `ODR`, `BSRR`, `AFR[2]`.
- 2.3. Lập trình xuất ngõ ra: Điều khiển LED đơn, LED 7 đoạn, Relay điều khiển tải 220VAC (Mạch cách ly quang Opto PC817, Transistor đệm dòng, Diode dập xung ngược).
- 2.4. Lập trình nhập ngõ vào: Đọc trạng thái nút nhấn đơn, bài toán chống rung phím (Key Debounce) bằng phần cứng (mạch RC) và phần mềm (lập trình timer/delay).
- 2.5. Kỹ thuật quét ma trận phím $4 \times 4$: Nguyên lý quét hàng/cột, giải mã phím nhấn.
- 2.6. Khái niệm chức năng luân phiên (Alternate Function - AF) và bảng ánh xạ AF.

### CHƯƠNG 3: HỆ THỐNG QUẢN LÝ NGẮT (NVIC) VÀ XỬ LÝ SỰ KIỆN NGOẠI VI NGẮT NGOÀI (EXTI)
- 3.1. Khái niệm ngắt (Interrupt) và cơ chế chuyển đổi ngữ cảnh (Context Switching).
- 3.2. Bộ điều khiển ngắt lồng nhau NVIC (Nested Vectored Interrupt Controller):
  - Bảng vector ngắt (Interrupt Vector Table).
  - Cấu hình mức ưu tiên ngắt: Nhóm ưu tiên (Priority Grouping), Preemption Priority và Sub-Priority.
- 3.3. Khối quản lý ngắt ngoài EXTI (Extended Interrupts and Events Controller):
  - Kiến trúc đường dây EXTI0 đến EXTI15 và bộ chọn đường ngắt `SYSCFG_EXTICR`.
  - Cấu hình bắt cạnh lên (Rising Edge), cạnh xuống (Falling Edge) hoặc cả hai.
- 3.4. Viết hàm phục vụ ngắt (ISR - Interrupt Service Routine) an toàn và tối ưu:
  - Khái niệm từ khóa `volatile` trong ngôn ngữ C nhúng.
  - Xóa cờ ngắt đúng quy chuẩn.
  - Phân chia công việc giữa ISR và vòng lặp chính (Flag polling & Non-blocking paradigm).

### CHƯƠNG 4: KHỐI ĐỊNH THỜI (TIMER/COUNTER), ĐIỀU CHẾ ĐỘ RỘNG XUNG (PWM) & ĐIỀU KHIỂN ĐỘNG CƠ
- 4.1. Phân loại các khối Timer trên STM32F4: Basic Timers (TIM6/7), General-Purpose Timers (TIM2..5, 9..14), Advanced Timers (TIM1/8).
- 4.2. Cấu trúc khối Timer cơ bản: Bộ chia trước Prescaler (`PSC`), Thanh ghi tự động nạp lại Auto-Reload Register (`ARR`), Bộ đếm Counter (`CNT`).
- 4.3. Công thức tính thời gian tràn (Update Event Time) và lập trình ngắt Timer chu kỳ chính xác.
- 4.4. Chế độ Input Capture: Đo tần số và đo độ rộng xung tín hiệu ngoại vi (Ứng dụng cảm biến siêu âm HC-SR04, đọc tín hiệu encoder).
- 4.5. Chế độ Output Compare & Điều chế độ rộng xung (PWM):
  - Thanh ghi `CCR` (Capture/Compare Register) và công thức tính chu kỳ, hệ số công tác Duty Cycle ($D = \frac{CCR}{ARR+1}$).
  - Cấu hình kênh PWM (Edge-aligned vs Center-aligned).
- 4.6. Ứng dụng điều khiển ngoại vi công suất:
  - Điều khiển độ sáng đèn LED, còi chíp phát giai điệu âm thanh.
  - Điều khiển tốc độ động cơ DC qua mạch cầu H (L298, BTS7960).
  - Điều khiển vị trí góc quay động cơ RC Servo (SG90, MG996R) bằng xung PWM 50Hz (độ rộng xung $1\text{ ms} \div 2\text{ ms}$).

### CHƯƠNG 5: BỘ CHUYỂN ĐỔI TƯƠNG TỰ - SỐ (ADC) VÀ SỐ - TƯƠNG TỰ (DAC)
- 5.1. Nguyên lý lấy mẫu và biến đổi ADC: Tần số lấy mẫu Nyquist, độ phân giải (12-bit), điện áp tham chiếu $V_{REF+}$.
- 5.2. Khối ADC trên STM32F4:
  - Cấu hình kênh ngõ vào (Regular Channels vs Injected Channels).
  - Các chế độ hoạt động: Đơn kênh (Single conversion), Chuyển đổi liên tục (Continuous mode), Quét nhiều kênh (Scan mode).
  - Cơ chế kích hoạt chuyển đổi (Trigger sources): Bằng phần mềm, bằng ngắt ngoài, bằng tín hiệu Timer (TRGO).
- 5.3. Kỹ thuật đọc cảm biến công nghiệp & điện tử:
  - Ghép nối biến trở xoay, cảm biến ánh sáng (LDR), cảm biến nhiệt độ tương tự LM35/NTC.
  - Kỹ thuật lọc nhiễu số (Digital Filtering): Lọc trung bình cộng (Moving Average Filter), lọc thông thấp số (Exponential Filter).
- 5.4. Khối chuyển đổi Số - Tương tự (DAC):
  - Cấu hình 12-bit DAC trên STM32F4.
  - Tạo điện áp DC tùy chỉnh.
  - Ứng dụng phát dạng sóng (Sin, Tam giác, Răng cưa) kết hợp bảng tra (Lookup Table) và Timer TRGO.

### CHƯƠNG 6: GIAO THỨC TRUYỀN THÔNG NỐI TIẾP DỊ BỘ UART / USART
- 6.1. Nguyên lý truyền thông UART: Khung dữ liệu (Start bit, Data bits, Parity bit, Stop bit), tốc độ Baud (Baud rate).
- 6.2. Ghép nối vật lý và mạch giao tiếp điện áp:
  - Chuẩn TTL 3.3V / 5V.
  - Chuẩn RS-232 (IC chuyển đổi mức MAX232/MAX3232).
  - Chuẩn công nghiệp RS-485 bán song công (Half-duplex) với IC MAX485/SP3485 (Điều khiển chân DE/RE).
- 6.3. Cấu hình khối USART trên STM32F4:
  - Thanh ghi `SR`, `DR`, `BRR`, `CR1`, `CR2`, `CR3`.
  - Tính toán giá trị Baud Rate Register (`USART_BRR`).
- 6.4. Các phương pháp lập trình truyền nhận UART:
  - Polling (Thăm dò cờ `TXE`, `RXNE`).
  - Interrupt-driven (Ngắt nhận ký tự, xử lý bộ đệm vòng Ring Buffer).
  - Tối ưu truyền nhận chuỗi dữ liệu với DMA và ngắt IDLE Line Detection.
- 6.5. Ứng dụng thực tế:
  - Giao tiếp máy tính PC (Virtual COM Port qua USB-TTL / Debug Console `printf`).
  - Ghép nối module không dây Bluetooth HC-05 / ESP8266 Wi-Fi / GPS NEO-6M / SIM4G.

### CHƯƠNG 7: GIAO THỨC TRUYỀN THÔNG NỐI TIẾP ĐỒNG BỘ SPI & GHÉP NỐI BỘ NHỚ / MÀN HÌNH
- 7.1. Nguyên lý giao thức SPI: Cấu trúc 4 dây (SCK, MOSI, MISO, CS/NSS), mô hình Master - Multi Slave.
- 7.2. Các chế độ hoạt động SPI (SPI Modes 0, 1, 2, 3) dựa trên cực tính xung nhịp (`CPOL`) và pha xung nhịp (`CPHA`).
- 7.3. Cấu hình thanh ghi SPI trên STM32F4: `CR1`, `CR2`, `SR`, `DR`.
- 7.4. Ghép nối IC ghi dịch xuất mở rộng ngõ ra 74HC595 và nhập ngõ vào 74HC165.
- 7.5. Giao tiếp bộ nhớ Flash ngoài SPI (Winbond W25Qxx): Đọc ID chip, xóa sector, ghi và đọc dữ liệu.
- 7.6. Giao tiếp thẻ nhớ MicroSD qua chuẩn SPI và hệ thống tập tin FatFS.
- 7.7. Điều khiển màn hình hiển thị đồ họa SPI: TFT LCD ST7735 / ST7789 hoặc OLED SSD1306 SPI.

### CHƯƠNG 8: GIAO THỨC TRUYỀN THÔNG HAI DÂY I2C & GHÉP NỐI CÁC IC NGOẠI VI CHUYÊN DỤNG
- 8.1. Nguyên lý giao thức $I^2C$: Cấu trúc 2 dây mở máng (Open-Drain) với điện trở kéo lên Pull-up ($SDA, SCL$), cấu trúc khung truyền (Start, Slave Address, R/W, ACK/NACK, Data, Stop).
- 8.2. Các chế độ tốc độ I2C: Standard-mode (100 kbps), Fast-mode (400 kbps).
- 8.3. Khối phần cứng I2C trên STM32F4 và giải quyết lỗi kẹt bus (Bus Lockup / Clock Stretching).
- 8.4. Lập trình giao tiếp IC thời gian thực RTC (DS1307 / DS3231): Đọc giờ, phút, giây, ngày, tháng, năm dạng mã BCD.
- 8.5. Ghép nối IC mở rộng I/O PCF8574 điều khiển màn hình Text LCD 16x2 / 20x4.
- 8.6. Ghép nối cảm biến chuyển động 6 trục IMU (Gia tốc kế & Con quay hồi chuyển MPU6050): Khởi tạo thanh ghi cấu hình, đọc dữ liệu thô và chuyển đổi sang đơn vị vật lý ($g, ^\circ/s$).
- 8.7. Giao tiếp cảm biến môi trường I2C (BME280 / SHT30 đo nhiệt độ, độ ẩm, áp suất khí quyển).

### CHƯƠNG 9: BỘ ĐIỀU KHIỂN TRUY CẬP BỘ NHỚ TRỰC TIẾP (DMA) & TỐI ƯU HÓA HỆ THỐNG
- 9.1. Khái niệm và vai trò của DMA trong vi điều khiển: Giảm tải cho CPU, truyền tải khối lượng dữ liệu lớn trực tiếp giữa ngoại vi và bộ nhớ SRAM.
- 9.2. Kiến trúc DMA trên STM32F4: 2 bộ điều khiển DMA1 và DMA2, mỗi bộ có 8 luồng (Streams) và 8 kênh (Channels) trên mỗi stream.
- 9.3. Các chế độ hoạt động của DMA:
  - Memory-to-Memory (M2M).
  - Peripheral-to-Memory (P2M).
  - Memory-to-Peripheral (M2P).
  - Chế độ bộ đệm vòng (Circular mode) và bộ đệm đôi (Double Buffer mode).
- 9.4. Ứng dụng thực tế của DMA:
  - Đọc đa kênh ADC liên tục không chiếm dụng CPU (ADC Multi-channel Scan with DMA Circular Buffer).
  - Truyền dữ liệu tốc độ cao qua UART-DMA (truyền mảng dữ liệu lớn lên máy tính mà không làm chậm hệ thống).
  - Đẩy dữ liệu điểm ảnh ra màn hình TFT LCD SPI bằng DMA (tối ưu hóa tốc độ làm tươi màn hình Frame Rate).

### CHƯƠNG 10: GIAO THỨC TRUYỀN THÔNG CÔNG NGHIỆP CAN BUS TRONG HỆ THỐNG NHÚNG
- 10.1. Giới thiệu mạng CAN (Controller Area Network): Lịch sử phát triển, cấu trúc vi sai 2 dây (CAN_H, CAN_L), khả năng chống nhiễu vượt trội trong môi trường công nghiệp và ô tô (Automotive).
- 10.2. Cấu trúc khung truyền CAN (CAN 2.0A 11-bit ID & CAN 2.0B 29-bit ID): Cực tính Recessive/Dominant, trường trọng tài (Arbitration), CRC kiểm tra lỗi.
- 10.3. Khối bxCAN trên STM32F4 và vi mạch thu phát CAN Transceiver (TJA1050 / MCP2551).
- 10.4. Cấu hình bộ lọc tin nhắn (Identifier Filtering): Mask mode và List mode.
- 10.5. Thực hành thiết lập mạng truyền thông 2 nút STM32 giao tiếp dữ liệu điều khiển qua CAN bus.

### CHƯƠNG 11: THIẾT KẾ HỆ THỐNG NHÚNG ĐIỀU KHIỂN NGOẠI VI TỔNG HỢP (CAPSTONE MINI-PROJECT)
- 11.1. Phương pháp luận thiết kế hệ thống nhúng điều khiển ngoại vi:
  - Phân tích yêu cầu phần cứng và lập bảng phân bổ chân I/O (Pinout Allocation & Conflict Resolution).
  - Thiết kế sơ đồ nguyên lý mạch (Schematic Design) và layout PCB theo chuẩn công nghiệp (lọc nguồn, chống nhiễu, cách ly).
  - Thiết kế kiến trúc phần mềm: Phân lớp Driver, Hardware Abstraction Layer (HAL), Middleware và Application Layer.
- 11.2. Mô hình máy trạng thái hữu hạn (FSM) và lập trình điều khiển đa nhiệm hợp tác (Cooperative Multitasking / Non-blocking Super-loop).
- 11.3. Các đề tài dự án mẫu:
  - Đề tài 1: Trạm quan trắc môi trường thông minh (Cảm biến SHT30/BME280 qua I2C + Lưu trữ thẻ nhớ SD qua SPI + Gửi dữ liệu IoT qua Wi-Fi/4G UART + Hiển thị LCD/OLED).
  - Đề tài 2: Hệ thống điều khiển tốc độ và góc quay động cơ khép kín (Đọc Encoder qua Timer Input Capture + Điều khiển cầu H bằng PWM + Báo động qua Relay/Buzzer + Giám sát qua PC).
  - Đề tài 3: Hệ thống khóa cửa bảo mật tích hợp (Đọc ma trận phím $4\times 4$ + Thẻ từ RFID SPI + Mở chốt điện tử Solenoid + Hiển thị TFT LCD + Còi báo động ngắt ngoài).
- 11.4. Quy chuẩn viết báo cáo kỹ thuật, tài liệu hướng dẫn sử dụng và bảo vệ dự án.

---

## V. HỆ THỐNG 10 BÀI THỰC HÀNH - THÍ NGHIỆM TẠI LAB (1 TÍN CHỈ = 30 TIẾT)

| Bài Lab | Tên bài thực hành | Nội dung trọng tâm | Thiết bị phần cứng |
| :---: | :--- | :--- | :--- |
| **Lab 01** | Làm quen Toolchain & Điều khiển GPIO cơ bản | Cài đặt STM32CubeIDE, tạo project CubeMX, nạp code nhấp nháy LED, điều khiển chuỗi LED hiệu ứng, đọc nút nhấn đơn chống rung. | Kit STM32F4, LED đơn, nút nhấn, điện trở. |
| **Lab 02** | Ghép nối ngoại vi hiển thị & Tải công suất | Điều khiển LED 7 đoạn (trực tiếp & quét multiplexing), đóng cắt Relay tải 220V qua transistor/opto, phát còi Buzzer. | Kit STM32F4, Module LED 7 đoạn, Module Relay 5V, Còi Buzzer. |
| **Lab 03** | Hệ thống ngắt ngoài EXTI & Ma trận phím | Lập trình ngắt ngoài EXTI từ nút bấm khẩn cấp (Emergency Button), giải mã ma trận phím $4\times 4$ kết hợp ngắt đánh thức. | Kit STM32F4, Ma trận phím $4\times 4$, Nút nhấn công nghiệp. |
| **Lab 04** | Bộ định thời Timer, Tạo xung nhịp & Đo thời gian | Cấu hình Timer ngắt chu kỳ chính xác 1ms/10ms/1s; Đo độ rộng xung cảm biến siêu âm HC-SR04 bằng chế độ Input Capture. | Kit STM32F4, Cảm biến siêu âm HC-SR04, Máy hiện sóng. |
| **Lab 05** | Băm xung PWM & Điều khiển công suất / Động cơ | Điều chế PWM thay đổi độ sáng LED; Điều khiển góc quay động cơ RC Servo; Điều khiển tốc độ động cơ DC qua module cầu H L298. | Kit STM32F4, Động cơ Servo SG90, Động cơ DC + Cầu H L298, Nguồn rời 12V. |
| **Lab 06** | Thu thập dữ liệu ADC & Điều khiển ngõ ra DAC | Cấu hình ADC đa kênh đọc biến trở và cảm biến nhiệt độ LM35; Kỹ thuật lọc số trung bình; Lập trình DAC phát sóng hình sin ra loa. | Kit STM32F4, Biến trở, Cảm biến LM35, Loa mini/Tai nghe, Dao động ký. |
| **Lab 07** | Giao tiếp máy tính và thiết bị qua UART | Lập trình gửi nhận ký tự, chuỗi ký tự qua Virtual COM; Thiết lập giao thức lệnh điều khiển thiết bị từ PC; Giao tiếp Bluetooth HC-05. | Kit STM32F4, Mạch chuyển USB-TTL (CP2102/CH340), Module HC-05. |
| **Lab 08** | Giao tiếp chuẩn I2C với IC chuyên dụng | Ghép nối và cấu hình IC thời gian thực RTC DS3231; Điều khiển màn hình Text LCD 16x2 qua IC PCF8574; Đọc cảm biến góc nghiêng MPU6050. | Kit STM32F4, Module RTC DS3231, LCD 1602 I2C, Cảm biến MPU6050. |
| **Lab 09** | Giao tiếp chuẩn SPI với IC nhớ Flash & Màn hình | Giao tiếp vi mạch ghi dịch 74HC595; Đọc ghi dữ liệu trên bộ nhớ Flash SPI W25Qxx; Hiển thị đồ họa trên màn hình OLED SSD1306/TFT. | Kit STM32F4, Module Flash W25Q64, Màn hình OLED SSD1306 SPI. |
| **Lab 10** | Tối ưu hóa hiệu năng với DMA & Tích hợp hệ thống | Lập trình ADC đọc liên tục qua DMA Circular buffer; Truyền dữ liệu cảm biến tốc độ cao lên PC qua UART-DMA; Xây dựng Mini Project. | Kit STM32F4, Logic Analyzer, Toàn bộ module cảm biến tổng hợp. |

---

## VI. PHƯƠNG PHÁP ĐÁNH GIÁ HỌC PHẦN
1. **Điểm chuyên cần và thái độ học tập (10%):** Tham gia đầy đủ các buổi lên lớp, chuẩn bị bài trước giờ học.
2. **Điểm đánh giá thực hành tại phòng Lab (30%):** 
   - Kiểm tra kỹ năng thực hành từng bài lab (10 bài).
   - Đánh giá khả năng debug lỗi phần cứng và phân tích tín hiệu trên Oscilloscope/Logic Analyzer.
3. **Điểm bài tập lớn / Mini Capstone Project (20%):**
   - Đánh giá sản phẩm phần cứng, mã nguồn, báo cáo kỹ thuật và phần thuyết trình phản biện nhóm.
4. **Điểm thi kết thúc học phần (40%):** 
   - Thi viết tự luận hoặc thi thực hành lập trình trực tiếp trên máy tính trong thời gian 90 phút (Thiết kế sơ đồ nguyên lý ghép nối + Viết chương trình điều khiển một tổ hợp thiết bị ngoại vi cụ thể).
