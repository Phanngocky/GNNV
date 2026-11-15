# Patterns & Lessons Learned (Bài học kinh nghiệm)

Tài liệu này lưu trữ các nguyên tắc thiết kế, phương pháp sư phạm, quy chuẩn biên soạn giáo trình và các bài học kỹ thuật đúc kết được trong quá trình phát triển hệ thống bài giảng ngành Kỹ thuật Điện tử.

## 1. Nguyên Tắc Sư Phạm (Pedagogical Principles)
- **Học thông qua hành (Learn-by-doing):** Mọi khái niệm lý thuyết về ngoại vi đều cần đi kèm sơ đồ ghép nối phần cứng và ví dụ code mẫu kiểm chứng ngay lập tức.
- **Từ thanh ghi (Bare-metal) đến Thư viện chuẩn (HAL / LL):** Sinh viên ngành Điện tử cần hiểu rõ bản chất hoạt động của thanh ghi và xung nhịp (Clock Tree), sau đó mới nâng lên sử dụng HAL để tối ưu tốc độ phát triển dự án.
- **Tiêu chuẩn công nghiệp & An toàn phần cứng:** Chú trọng phân tích bảo vệ mạch (Optocoupler cách ly, diode dập xung, điện trở pull-up/pull-down, tụ lọc nhiễu).

## 2. Quy Chuẩn Trình Bày Code & Sơ Đồ
- Mã nguồn C/C++ tuân thủ chuẩn MISRA-C và CMSIS.
- Sơ đồ mạch nguyên lý rõ ràng, chú thích đầy đủ mức điện áp 3.3V / 5V (tránh xung đột logic level làm hỏng chân MCU).
- Biểu đồ thời gian (Timing diagrams) cho các chuẩn giao tiếp nối tiếp (UART, SPI, I2C).
