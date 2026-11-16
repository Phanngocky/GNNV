# HỆ THỐNG BÀI THỰC HÀNH THÍ NGHIỆM (LAB MANUAL)
## ĐIỀU KHIỂN THIẾT BỊ NGOẠI VI & GHÉP NỐI HỆ THỐNG NHÚNG (STM32F407)
**Thời lượng:** 45 giờ (15 buổi $\times$ 3 giờ/buổi theo mô hình Studio)  
**Nền tảng thực hành:** Kit vi điều khiển ARM Cortex-M4 (STM32F407VGT6)  
**Phần mềm:** STM32CubeIDE, STM32CubeMX, PulseView / Saleae Logic, Oscilloscope  
**Tài liệu tham chiếu:** *ARM Education - Beginning Guide to Designing Embedded Systems (Chương 1 $\div$ 12)*

---

## MỤC LỤC 15 BÀI THỰC HÀNH CHI TIẾT

| STT | Tên bài thực hành thí nghiệm | Chủ đề công nghệ ngoại vi | Tệp hướng dẫn chi tiết |
| :---: | :--- | :--- | :--- |
| **01** | **Khởi động Smart Home & Flash ART** | GPIO nâng cao, LED, Nút nhấn, Cảm biến Gas/Quá nhiệt, Prefetch & Flash ART Accelerator | [Lab_01_Khoi_Dong_Smart_Home_GPIO_ART.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_01_Khoi_Dong_Smart_Home_GPIO_ART.md) |
| **02** | **Truyền thông UART Nâng cao & PC** | USART2 APB1 42MHz, Baudrate `USART_BRR`, Tái định tuyến `printf`, Command Parser | [Lab_02_UART_Nang_Cao_PC_Communication.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_02_UART_Nang_Cao_PC_Communication.md) |
| **03** | **ADC Không Chặn & Còi Báo Động** | SAR ADC 12-bit, LM35, Lọc số Moving Average, Non-blocking Timing SysTick, Còi Buzzer | [Lab_03_ADC_Non_Blocking_Buzzer.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_03_ADC_Non_Blocking_Buzzer.md) |
| **04** | **Máy Trạng Thái FSM, Phím Ma Trận & RTC** | FSM Moore/Mealy, Quét phím ma trận 4x4, Bắt đúp phím, Khóa PIN bảo mật, LSE RTC 32.768kHz | [Lab_04_FSM_Matrix_Keypad_RTC.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_04_FSM_Matrix_Keypad_RTC.md) |
| **05** | **Mô-đun Hóa Mã Nguồn & MISRA-C** | Phân tầng kiến trúc, Header Guards, Đóng gói `static`/`extern`, Clean `main.c`, MISRA-C | [Lab_05_Mo_Dun_Hoa_Phan_Mem_MISRA_C.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_05_Mo_Dun_Hoa_Phan_Mem_MISRA_C.md) |
| **06** | **Màn hình LCD Bus Song song & $I^2C$** | LCD 16x2 GPIO 4-bit, Giao thức I2C Open-Drain, Trở kéo $R_p$, Driver PCF8574, Gỡ kẹt bus | [Lab_06_LCD1602_Parallel_GPIO_I2C_PCF8574.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_06_LCD1602_Parallel_GPIO_I2C_PCF8574.md) |
| **07** | **Chuẩn Giao tiếp 1-Wire & DS18B20** | Bus 1-Wire, Trễ micro-giây DWT CYCCNT, ROM 64-bit, Scratchpad, Cảm biến số DS18B20 | [Lab_07_Giao_Tiep_1Wire_Cam_Bien_DS18B20.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_07_Giao_Tiep_1Wire_Cam_Bien_DS18B20.md) |
| **08** | **Chuẩn Đồng bộ SPI & Màn hình OLED** | Giao thức SPI 4-wire, CPOL/CPHA, Frame Buffer 1024B, Driver SSD1306, Đồ thị nhiệt độ | [Lab_08_Giao_Tiep_SPI_Man_Hinh_OLED_SSD1306.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_08_Giao_Tiep_SPI_Man_Hinh_OLED_SSD1306.md) |
| **09** | **Động cơ DC, Relay & Ngắt NVIC** | Cầu 2 Relay SPDT, Opto PC817, Cảm kháng Diode dập xung, PIR EXTI4, Ngắt quá dòng EXTI5 | [Lab_09_Dieu_Khien_Dong_Co_DC_Relay_EXTI.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_09_Dieu_Khien_Dong_Co_DC_Relay_EXTI.md) |
| **10** | **Timer PWM Đa Kênh & Hồi Tiếp Âm** | TIM4 PWM 3 kênh RGB, Tính PSC/ARR/CCR, Vòng điều khiển kín Negative Feedback bù sáng LDR | [Lab_10_PWM_Timer_RGB_Hoi_Tiep_Quang_Tro_LDR.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_10_PWM_Timer_RGB_Hoi_Tiep_Quang_Tro_LDR.md) |
| **11** | **Phát Âm Thanh Thoại PWM & Lọc RC** | Sóng mang TIM1 100kHz, Định thời lấy mẫu TIM6 8kHz, Lọc thông thấp tích phân RC, PCM Flash | [Lab_11_Phat_Am_Thanh_Thoai_PWM_Loc_RC.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_11_Phat_Am_Thanh_Thoai_PWM_Loc_RC.md) |
| **12** | **Lưu trữ Thẻ Nhớ SD & Git CLI** | Thẻ nhớ MicroSD SPI2, Middleware FatFS (FAT32), Tệp `alarm_log.csv`, Quản lý Git CLI | [Lab_12_Luu_Tru_The_Nho_SD_FatFS_Git.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_12_Luu_Tru_The_Nho_SD_FatFS_Git.md) |
| **13** | **Không Dây BLE với Smartphone** | Kiến trúc BLE GAP/GATT, AT Commands, Module HC-08/HM-10 USART3, Điều khiển qua điện thoại | [Lab_13_Khong_Day_BLE_Smartphone_App.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_13_Khong_Day_BLE_Smartphone_App.md) |
| **14** | **Máy Chủ Web Nhúng Wi-Fi ESP8266** | TCP/IP Socket Port 80, HTTP Request/Response, USART6 ESP8266, HTML Dashboard điều khiển | [Lab_14_May_Chu_Web_Nhung_WiFi_ESP8266.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_14_May_Chu_Web_Nhung_WiFi_ESP8266.md) |
| **15** | **Tích hợp Capstone & Nghiệm thu** | Tích hợp toàn diện 10 khối ngoại vi trên STM32F407, Ma trận kiểm thử chấp nhận, Rubric CDIO | [Lab_15_Tich_Hop_Capstone_Smart_Home_System.md](file:///d:/tam/stm32f4/GNNV/01%20-%20Projects/Giao_Trinh_Dieu_Khien_Ngoai_Vi/Labs_Thuc_Hanh/Lab_15_Tich_Hop_Capstone_Smart_Home_System.md) |

---

## QUY ĐỊNH AN TOÀN PHÒNG THÍ NGHIỆM
1. **Quy tắc điện áp:** STM32F407 hoạt động ở mức logic **$3.3\text{V}$**. Tuyệt đối không cắm nguồn $5\text{V}$ vào các chân không chịu áp 5V (Non FT pins).
2. **Quy tắc nối đất:** Luôn nối chung dây mass ($GND$) giữa vi điều khiển, nguồn ngoài của động cơ, thiết bị đo kiểm (Oscilloscope, Logic Analyzer) và máy tính PC.
3. **Chống phóng tĩnh điện (ESD):** Không chạm tay trần vào chân linh kiện bán dẫn nhạy cảm khi đang mang tĩnh điện.
4. **Kiểm tra trước khi cấp nguồn:** Giảng viên hoặc Trợ giảng phải kiểm tra mạch dây đấu nối trước khi sinh viên cắm cáp USB cấp nguồn lần đầu tiên.
