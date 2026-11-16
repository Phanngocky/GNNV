# Lịch Sử Hoạt Động Hệ Thống (System Log)

## [2026-11-15] setup | Khởi tạo hệ thống Second Brain và chuẩn bị biên soạn giáo trình Điều khiển Thiết bị Ngoại vi
- Khởi tạo kiến trúc 3 lớp (Inbox, Knowledge, CLAUDE.md)
- Khởi tạo 6 bộ kỹ năng (Skills) cốt lõi
- Xây dựng đề cương chi tiết học phần và khung giáo trình 11 chương
- Lập kế hoạch biên soạn và hệ thống bài thí nghiệm thực hành STM32F4

## [2026-11-15] ingest | Phân tích phân bổ lộ trình 45 giờ học phần Điều khiển Ngoại vi từ sách ARM Education
- Trích xuất chi tiết cấu trúc 12 chương từ Beg.GuidetoDesigingEmbeddedSystems_ch1-6 và ch7-12
- Xây dựng lộ trình 15 buổi tích hợp Studio (50 phút Lý thuyết + 95 phút Thực hành + 35 phút Đo kiểm/Under the Hood)
- Lập bảng ma trận phân bổ 45 giờ, danh mục linh kiện BOM và bộ tiêu chí đánh giá Rubric cho sinh viên Điện tử

## [2026-11-15] ingest | Hoàn thành biên soạn đồng bộ 11 Chương Giáo trình Điều khiển Ngoại vi từ 2 tập sách ARM Education
- Biên soạn chi tiết toàn bộ 11 chương giáo trình kết hợp giữa lý thuyết vi xử lý 32-bit, tính toán mạch nguyên lý điện tử (Schematic), phân tích thanh ghi và mã nguồn 3 tầng (Mbed C++, STM32 HAL C, CMSIS Bare-metal)
- Chuyển ngữ và nâng cấp toàn bộ hệ thống ví dụ thực chứng từ Chapter 1 đến Chapter 12 của sách giáo trình ARM (Nucleo-F429ZI Smart Home System)
- Thiết lập đầy đủ bài tập tính toán mạch và câu hỏi kiểm tra đánh giá theo chuẩn CDIO

## [2026-11-15] ingest | Tái cấu trúc chuẩn hóa tuyệt đối 12 Chương Giáo trình theo đúng 2 tập sách ARM Education dành riêng cho STM32F407
- Định vị học phần học sau học phần Vi điều khiển cơ bản: Bỏ qua các kiến thức nhập môn chung chung, tập trung trực tiếp vào kiến trúc Cortex-M4 168MHz và ghép nối ngoại vi chuyên sâu
- Chuyển đổi và thiết lập toàn bộ sơ đồ chân, tính toán phần cứng và mã nguồn 12 chương tương thích 100% với dòng vi điều khiển STM32F407 (LQFP100)
- Hoàn thiện toàn bộ 12 chương giáo trình theo đúng thứ tự và các ví dụ thực chứng từ Chapter 1 đến Chapter 12 của ARM Education

## [2026-11-16] edit | Rà soát toàn diện giáo trình Điều khiển Ngoại vi (Kế thừa sau học phần Vi điều khiển đến UART)
- Định vị rõ ranh giới học phần: Kế thừa nền tảng Vi điều khiển STM32F407 (đã học đến UART), tập trung vào làm chủ thế giới ngoại vi thực tế xung quanh để hoàn thiện hệ thống nhúng
- Bổ sung chuyên đề sâu về chuẩn giao tiếp 1-Wire và cảm biến nhiệt độ số chính xác cao DS18B20 trên chân PB12 (định thời micro-giây DWT, bảng đối sánh Song song vs 1-Wire vs I2C vs SPI)
- Cập nhật đề cương chi tiết học phần, bảng phân bổ 45 giờ 15 buổi tích hợp Studio, hệ thống index tri thức và đồng bộ lên GitHub

## [2026-11-16] ingest | Hoàn thiện trọn bộ Hệ thống 15 Bài Thí nghiệm Thực hành (Lab Manual) và Ngân hàng Câu hỏi Ôn tập Thi KTHP
- Biên soạn trọn vẹn 15 bài Lab chi tiết (Lab 01 -> Lab 15) theo mô hình lớp học Studio: Mục tiêu CLO, BOM linh kiện, Sơ đồ nguyên lý & Bảng chân, Cấu hình CubeMX, Mã nguồn C chuẩn, Quy trình đo kiểm Oscilloscope / Logic Analyzer, Troubleshooting và Rubric chấm điểm
- Tích hợp chuyên đề thực hành 1-Wire DS18B20 (DWT us delay), I2C LCD PCF8574, SPI OLED SSD1306, Động cơ Relay NVIC EXTI, Timer PWM Audio 100kHz lọc RC, Thẻ nhớ SD FatFS, BLE không dây và Embedded Web Server Wi-Fi ESP8266
- Xây dựng bài Lab 15 Capstone Project tích hợp đồng thời 10 khối ngoại vi trên STM32F407 kèm Ma trận kiểm thử chấp nhận (Acceptance Test Matrix)
- Biên soạn Ngân hàng câu hỏi ôn tập và thi kết thúc học phần gồm 120 câu hỏi trắc nghiệm chuyên sâu phân bổ 12 chương và 15 bài toán thiết kế mạch ngoại vi thực tế

## [2026-11-16] edit | Nâng cấp toàn diện Cơ sở lý thuyết, Sơ đồ nguyên lý Schematics và Bảng đấu nối chân chi tiết cho 12 Chương Giáo trình
- Mở rộng chuyên sâu Cơ sở lý thuyết: Toán học băm xung PWM và hiệu suất công suất, vòng điều khiển kín hồi tiếp âm (P controller, error, anti-windup), quang trở LDR & phối màu LED RGB, giải điều chế PWM Audio 100kHz qua lọc tích phân RC (Bode, hàm truyền Laplace, tụ chặn DC); hiện tượng cảm kháng Back-EMF và diode dập xung, mạch cầu 2 Relay SPDT, NVIC 12 clock cycles latency & tail-chaining; cấu trúc vật lý SD SPI và middleware FatFS (bộ đệm 512B); kiến trúc vô tuyến BLE GAP/GATT & Duty cycling tiết kiệm điện; kiến trúc mạng TCP/IP socket cổng 80 & bản tin HTTP/1.1; quy trình kỹ thuật V-Model Capstone 10 bước và thiết kế nguồn 3 tầng
- Bổ sung hệ thống sơ đồ mạch nguyên lý điện tử (Schematics) chi tiết bằng ASCII art trực quan và bảng phân bổ chân Master Pinout Table không xung đột cho từng chương
- Đồng bộ mã nguồn và cấu trúc tài liệu lên GitHub kho lưu trữ GNNV
