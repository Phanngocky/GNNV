# KẾ HOẠCH TỔNG THỂ BIÊN SOẠN GIÁO TRÌNH ĐIỀU KHIỂN THIẾT BỊ NGOẠI VI

---

## 1. MỤC ĐÍCH VÀ ĐỐI TƯỢNG PHỤC VỤ
- **Mục đích:** Xây dựng bộ giáo trình học tập và tài liệu hướng dẫn thực hành - thí nghiệm chuẩn mực, hiện đại, bám sát nhu cầu tuyển dụng của doanh nghiệp nhúng và vi mạch cho sinh viên ngành Điện tử / Điện tử - Viễn thông / Tự động hóa.
- **Đối tượng học tập:** Sinh viên đại học từ năm thứ 2 đến năm thứ 4 đã có kiến thức nền tảng về C/C++ và mạch điện tử cơ bản.
- **Tính đột phá của giáo trình:**
  - Kết hợp chặt chẽ giữa **Hiểu sâu bản chất phần cứng (Thanh ghi/Bare-metal)** và **Kỹ năng ứng dụng công nghiệp (STM32 HAL, CubeMX, DMA)**.
  - Mỗi chương lý thuyết đều có **Sơ đồ nguyên lý mạch điện tử (Schematic)**, **Công thức tính toán linh kiện ghép nối** và **Dạng sóng thực tế đo bằng máy hiện sóng/Logic Analyzer**.
  - Tích hợp 10 bài thí nghiệm thực tế đi kèm code mẫu đã kiểm chứng 100% trên phần cứng STM32F4.

---

## 2. KẾ HOẠCH TRIỂN KHAI THEO GIAI ĐOẠN (ROADMAP)

```mermaid
flowchart LR
    P1[Giai đoạn 1: Chuẩn bị & Khung tài liệu] --> P2[Giai đoạn 2: Biên soạn Lý thuyết 11 Chương]
    P2 --> P3[Giai đoạn 3: Phát triển Code mẫu & 10 Bài Lab]
    P3 --> P4[Giai đoạn 4: Thẩm định & Hoàn thiện ấn phẩm]
```

### Giai đoạn 1: Khởi tạo Kiến trúc và Tài nguyên (Tuần 1 - Tuần 2)
- [x] Khởi tạo môi trường Second Brain Antigravity và cấu trúc thư mục dự án.
- [x] Soạn thảo Đề cương chi tiết học phần (`00_De_Cuong_Chi_Tiet_Mon_Hoc.md`).
- [x] Xây dựng các mẫu chuẩn (Template chương giáo trình, Template bài Lab thực hành).
- [ ] Tập hợp các tài liệu kỹ thuật gốc (Reference Manual RM0090, Datasheets STM32F4, Sách giáo trình ARM Education).

### Giai đoạn 2: Biên soạn Nội dung 11 Chương Giáo trình (Tuần 3 - Tuần 8)
- **Đợt 1 (Tuần 3 - 4): Nền tảng kiến trúc & Giao tiếp cơ bản**
  - Chương 1: Kiến trúc ARM Cortex-M4 & Bus Matrix STM32F4.
  - Chương 2: Điều khiển Cổng GPIO & Mạch ghép nối cơ bản.
  - Chương 3: Hệ thống ngắt NVIC & Ngắt ngoài EXTI.
- **Đợt 2 (Tuần 5 - 6): Xử lý tín hiệu, Thời gian & Công suất**
  - Chương 4: Khối định thời Timer, Input Capture & Điều chế PWM.
  - Chương 5: Bộ biến đổi tương tự - số ADC & số - tương tự DAC.
- **Đợt 3 (Tuần 7 - 8): Các giao thức truyền thông nối tiếp & Ngoại vi nâng cao**
  - Chương 6: Truyền thông nối tiếp UART/USART & Mạng RS-485.
  - Chương 7: Giao thức truyền thông SPI, Flash W25Qxx & Màn hình hiển thị.
  - Chương 8: Giao thức truyền thông I2C, RTC DS3231 & Cảm biến MPU6050.
  - Chương 9: Bộ điều khiển truy cập bộ nhớ trực tiếp DMA.
  - Chương 10: Mạng truyền thông công nghiệp CAN Bus.
  - Chương 11: Thiết kế hệ thống nhúng tổng hợp (Capstone Mini-Project).

### Giai đoạn 3: Xây dựng Hệ thống 10 Bài Thí nghiệm & Code Mẫu (Tuần 9 - Tuần 11)
- Lập trình kiểm chứng toàn bộ mã nguồn ví dụ trên kit STM32F4 (F401/F407/F429).
- Chụp ảnh dạng sóng thực tế trên Logic Analyzer và Oscilloscope dán vào tài liệu Lab.
- Soạn thảo hướng dẫn từng bước (Step-by-step Lab Manual) và Rubric chấm điểm.

### Giai đoạn 4: Rà soát, Thẩm định & Xuất bản (Tuần 12 - Tuần 14)
- Kiểm tra chéo (Linting): Rà soát tính chính xác của thanh ghi, công thức toán và sơ đồ mạch.
- Đóng gói tài liệu: Xuất bản file Markdown, PDF, Slide bài giảng và kho mã nguồn mẫu (Git Repository).

---

## 3. TIÊU CHÍ CHẤT LƯỢNG NỘI DUNG (QUALITY GATES)
1. **Tính chính xác tuyệt đối về phần cứng:** Mọi tên thanh ghi, vị trí bit, công thức tính xung nhịp phải đối chiếu trực tiếp với STM32F4 Reference Manual (RM0090) của STMicroelectronics.
2. **Quy chuẩn mã nguồn:** Mã nguồn viết bằng ngôn ngữ C chuẩn C99, có chú thích chi tiết từng bước, định dạng thụt dòng rõ ràng, tuân thủ nguyên tắc lập trình an toàn MISRA-C.
3. **Tính trực quan sư phạm:** Mỗi chương phải có ít nhất:
   - 01 Sơ đồ khối nguyên lý hoặc lưu đồ thuật toán (Mermaid flowchart/sequence).
   - 01 Sơ đồ mạch điện tử ghép nối ngoại vi kèm giá trị linh kiện cụ thể.
   - 01 Bảng phân tích thanh ghi và giải thích các bit quan trọng.
   - Code mẫu so sánh giữa Bare-metal và HAL Driver.
