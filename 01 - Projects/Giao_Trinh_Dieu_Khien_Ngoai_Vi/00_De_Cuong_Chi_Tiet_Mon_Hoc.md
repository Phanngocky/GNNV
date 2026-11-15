# ĐỀ CƯƠNG CHI TIẾT HỌC PHẦN: ĐIỀU KHIỂN THIẾT BỊ NGOẠI VI & GHÉP NỐI VI XỬ LÝ
**Chuyên ngành:** Kỹ thuật Điện tử / Kỹ thuật Điện tử - Viễn thông / Cơ điện tử  
**Trình độ đào tạo:** Đại học chính quy (Học phần học sau học phần Vi điều khiển cơ bản)  
**Phần cứng mục tiêu chuẩn hóa:** Vi điều khiển 32-bit ARM Cortex-M4 (**STM32F407VGT6 / STM32F407VET6**)  
**Tài liệu tham chiếu chuẩn:** *A Beginner's Guide to Designing Embedded System Applications on Arm Cortex-M Microcontrollers* (Arm Education Program, 12 Chapters)  
**Môi trường phát triển:** STM32CubeIDE / Keil MDK ARM + Logic Analyzer / Oscilloscope  

---

## I. THÔNG TIN CHUNG VỀ HỌC PHẦN
- **Tên học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Vi xử lý
- **Tên tiếng Anh:** Microcontroller Peripherals and Interfacing Techniques
- **Thời lượng:** 3 tín chỉ (45 giờ tiếp xúc trực tiếp = 15 buổi $\times$ 3 giờ/buổi theo mô hình Lớp học tích hợp Studio)
- **Vị trí học phần:** Học phần chuyên ngành cốt lõi, **học sau học phần Vi điều khiển cơ bản**. Sinh viên đã có kiến thức nền tảng về kiến trúc máy tính, thanh ghi và lập trình C nhúng.

---

## II. CẤU TRÚC CHƯƠNG TRÌNH 12 CHƯƠNG CHUẨN HÓA (THEO GIÁO TRÌNH GỐC ARM EDUCATION)

| Chương | Tên Chương Giáo trình | Khối ngoại vi & Chủ đề kỹ thuật | Ví dụ thực chứng trên STM32F407 |
| :---: | :--- | :--- | :--- |
| **Chương 1** | **Tổng quan Hệ thống Nhúng & Ứng dụng Báo động Đầu tiên** | Cấu trúc ARM Cortex-M4 (STM32F407 @ 168MHz), Bus Matrix, Memory Map, Cổng GPIO, LED, Nút bấm | Examples 1.1 $\div$ 1.5 (Báo động Gas, Quá nhiệt, Chốt trạng thái, Nhập mã giải mã, Khóa an toàn) |
| **Chương 2** | **Cơ sở Truyền thông Nối tiếp và Giao tiếp Máy tính (UART)** | Chuẩn UART TTL, RS-232, RS-485, Cấu hình Baudrate USART2/1, Phân tách hàm mô-đun, Mảng dữ liệu | Examples 2.1 $\div$ 2.5 (Giám sát trạng thái qua PC, lệnh switch-case, tắt báo động từ xa, đổi mã từ PC) |
| **Chương 3** | **Quản lý Thời gian và Đo lường Tín hiệu Tương tự (ADC)** | Bộ biến đổi SAR ADC 12-bit, Cảm biến nhiệt LM35, Khí gas MQ-2, Biến trở ngưỡng, Quản lý thời gian không chặn | Examples 3.1 $\div$ 3.5 (Đo đa kênh, Loại bỏ delay bằng non-blocking, Ngưỡng nhiệt, Tính độ C, Còi Buzzer) |
| **Chương 4** | **Máy Trạng thái Hữu hạn (FSM) và Đồng hồ Thời gian thực (RTC)** | Mô hình FSM (Mealy/Moore), Quét ma trận phím $4\times 4$, Khối RTC nội 32.768kHz và IC ngoài DS3231 | Examples 4.1 $\div$ 4.4 (Nhấn đúp phím Enter, Giải mã ma trận phím, Nhập mã PIN, Timestamp sự kiện) |
| **Chương 5** | **Mô-đun hóa Mã nguồn Phần mềm Hệ thống Nhúng** | Nguyên lý phân tách trách nhiệm, Tái cấu trúc (Refactoring) mã nguồn thành các module `.h`/`.c` riêng biệt | Examples 5.1 $\div$ 5.4 (inputs, outputs, alarm_system, pc_serial_com, matrix_keypad, rtc) |
| **Chương 6** | **Màn hình LCD và Truyền thông Vi mạch qua Bus I2C & SPI** | So sánh GPIO vs I2C vs SPI, LCD 16x2 4-bit, IC mở rộng PCF8574 I2C, Màn hình đồ họa SPI Graphical LCD | Examples 6.1 $\div$ 6.5 (LCD 4-bit, I2C LCD, OLED/TFT SPI, Vẽ đồ thị cảm biến, Under the Hood so sánh bus) |
| **Chương 7** | **Điều khiển Động cơ DC bằng Relay và Cơ chế Ngắt Thời gian thực** | Bộ điều khiển ngắt NVIC, Ngắt ngoài EXTI, Cảm biến chuyển động PIR, Cầu 2 Relay đảo chiều động cơ DC | Examples 7.1 $\div$ 7.4 (Ngắt PIR, Đóng mở cổng tự động, Bảo vệ quá dòng động cơ, Non-blocking gate timing) |
| **Chương 8** | **Quản lý Thời gian Nâng cao, PWM, Hồi tiếp Âm & Phát Âm thanh** | Khối Timer nâng cao TIM1/TIM4, Điều chế độ rộng xung PWM, Đổi màu LED RGB, Vòng hồi tiếp âm LDR, Lọc RC | Examples 8.1 $\div$ 8.6 (PWM đổi màu LED RGB, Điều khiển độ sáng đèn theo LDR, Phát giọng nói thoại từ Flash) |
| **Chương 9** | **Lưu trữ Tệp tin trên Thẻ nhớ SD & Quản lý Kho Phần mềm (Git)** | Giao diện SPI2 Thẻ nhớ MicroSD, Hệ thống tập tin FatFS (FAT32), Quản trị mã nguồn dự án nhúng bằng Git | Examples 9.1 $\div$ 9.4 (Tạo file log `.csv`, Ghi sự kiện mới, Đọc danh sách tệp tin, Truyền file lên PC qua UART) |
| **Chương 10** | **Truyền thông Không dây Bluetooth Low Energy (BLE) với Smartphone** | Kiến trúc giao thức BLE (GAP, GATT, Services, Characteristics), Module BLE HC-08/HM-10 qua USART3 | Examples 10.1 $\div$ 10.4 (Điều khiển mở cổng từ Smartphone, Báo cáo trạng thái, Thiết kế đối tượng, Trễ không chặn) |
| **Chương 11** | **Máy chủ Web Nhúng qua Kết nối Wi-Fi** | Chồng giao thức TCP/IP, Cổng 80 HTTP, Module Wi-Fi ESP8266 qua USART6, Tập lệnh AT, Trang Web Dashboard động | Examples 11.1 $\div$ 11.4 (Khởi tạo kết nối Wi-Fi, Khởi tạo TCP Web Server, Phục vụ HTML giám sát & điều khiển) |
| **Chương 12** | **Hướng dẫn Thiết kế và Hiện thực hóa Dự án Hệ thống Nhúng** | Quy trình 10 bước chuẩn công nghiệp: Khảo sát, Đặc tả, Phân bổ chân STM32F407, Nguồn cấp, FSM, Đo kiểm, Hồ sơ | Examples 12.1 $\div$ 12.10 (Capstone Project: Tích hợp hoàn chỉnh Hệ thống Nhà Thông minh Đa Năng) |
