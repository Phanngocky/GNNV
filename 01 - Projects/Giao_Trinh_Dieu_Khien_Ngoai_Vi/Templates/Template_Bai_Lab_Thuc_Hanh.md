# BÀI THỰC HÀNH SỐ [X]: [TÊN BÀI THÍ NGHIỆM]
**Thời lượng:** 3 tiết (135 phút)  
**Nền tảng:** Kit phát triển STM32F4 (Nucleo / Discovery / Custom STM32F407VGT6)  
**Phần mềm:** STM32CubeIDE, Logic Analyzer Software (Saleae Logic / PulseView)  

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
- Củng cố kiến thức lý thuyết về ngoại vi [Tên ngoại vi].
- Rèn luyện kỹ năng đấu nối mạch điện tử thực tế trên breadboard / module.
- Thực hành cấu hình ngoại vi trên STM32CubeMX và viết code trên STM32CubeIDE.
- Đo kiểm, bắt dạng sóng tín hiệu thực tế bằng máy hiện sóng / Logic Analyzer.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Tên thiết bị / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Kit phát triển STM32F4 | STM32F401RE / F407VET6 / F429ZI | 1 | Kèm cáp nạp ST-Link USB |
| 2 | Module ngoại vi chính | ... | 1 | Kèm dây cắm đực-cái |
| 3 | Máy tính cá nhân | Cài sẵn STM32CubeIDE | 1 | Phòng máy tính |
| 4 | Thiết bị đo kiểm | Logic Analyzer 8CH 24MHz / Oscilloscope | 1 | Đo tín hiệu |

---

## 3. SƠ ĐỒ ĐẤU NỐI MẠCH (PIN CONNECTION TABLE)
| Chân STM32F4 | Chân Ngoại vi / Module | Chức năng tín hiệu | Ghi chú an toàn điện áp |
| :---: | :---: | :---: | :--- |
| `PA5` | `CLK` / `SCK` | Xung nhịp đồng bộ | Mức logic 3.3V |
| `PA7` | `DIN` / `MOSI` | Dữ liệu Master Out | Mức logic 3.3V |
| `3V3` | `VCC` | Nguồn nuôi cảm biến | Tuyệt đối không cắm nhầm 5V |
| `GND` | `GND` | Nối chung mass | Bắt buộc chung mass |

---

## 4. NỘI DUNG THỰC HÀNH TỪNG BƯỚC (STEP-BY-STEP)

### Nhiệm vụ 1: Khởi tạo Project và Cấu hình Ngoại vi
- Mở STM32CubeMX, chọn dòng chip vi điều khiển tương ứng.
- Cấu hình nguồn xung Clock hệ thống (HSE thạch anh ngoài 8MHz/25MHz, nhân PLL lên tần số tối đa).
- Cấu hình các chân I/O và khối ngoại vi...
- Tạo mã nguồn (Generate Code) cho STM32CubeIDE.

### Nhiệm vụ 2: Viết mã nguồn điều khiển cơ bản
- Thêm mã nguồn vào các khối `/* USER CODE BEGIN ... */` và `/* USER CODE END ... */`.
- Nạp chương trình vào vi điều khiển và quan sát hiện tượng.

### Nhiệm vụ 3: Đo kiểm và Phân tích Dạng sóng Tín hiệu
- Kết nối các kênh của Logic Analyzer vào các đường truyền tín hiệu.
- Thiết lập tần số lấy mẫu (Sampling Rate >= 4 lần tần số tín hiệu).
- Bắt dạng sóng khi truyền dữ liệu. Chụp ảnh màn hình kết quả và giải thích khung truyền.

---

## 5. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC LAB)
### Báo cáo kết quả
- Chụp ảnh mạch thực tế đang hoạt động.
- Dán biểu đồ dạng sóng đo được từ Logic Analyzer / Oscilloscope kèm chú thích các byte truyền nhận.
- Đính kèm mã nguồn C đã chạy hoàn chỉnh.

### Câu hỏi đánh giá & Mở rộng (Bonus)
1. *Nếu tần số xung nhịp bus thay đổi từ 42MHz sang 84MHz thì tốc độ truyền sẽ thay đổi thế nào? Cần sửa thanh ghi nào?*
2. *Thực hiện bài toán nâng cao: Tích hợp thêm ngắt... để tối ưu hóa chương trình.*
