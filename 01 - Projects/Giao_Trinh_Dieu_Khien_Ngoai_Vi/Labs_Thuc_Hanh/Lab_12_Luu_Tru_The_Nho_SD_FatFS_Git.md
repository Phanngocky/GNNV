# BÀI THỰC HÀNH SỐ 12: LƯU TRỮ HỆ THỐNG TẬP TIN TRÊN THẺ NHỚ SD (FATFS) VÀ QUẢN LÝ MÃ NGUỒN BẰNG GIT
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết SPI SD/FAT32 $\rightarrow$ Cấu hình FatFS $\rightarrow$ Lập trình Logging CSV $\rightarrow$ Thực hành Git CLI)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Kiến trúc FatFS & Sector SD), CLO 2 (Mạch SPI thẻ nhớ), CLO 3 (Lập trình I/O File), CLO 4 (Quản lý dự án với Git)  
**Căn cứ giáo trình:** *ARM Education - Chương 9: Data Storage and Git Version Control (Ex 9.1 $\div$ 9.4)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Lưu trữ Dữ liệu & Hệ thống Tập tin Nhúng:**
   - Phân tích nguyên lý giao tiếp thẻ MicroSD ở chế độ SPI Mode: Cấu trúc sector vật lý $512\text{ bytes}$, trình tự gửi lệnh khởi tạo (Lệnh `CMD0`, `CMD8`, `ACMD41`, `CMD58`).
   - Nắm vững kiến trúc hệ thống tập tin **FAT (File Allocation Table - FAT16 / FAT32)**: Bảng phân vùng (Partition Table), Master Boot Record (MBR), Bảng chỉ mục FAT, và Thư mục gốc (Root Directory).
   - Nắm vững thư viện Middleware **FatFS** (Tác giả ChaN): Các tầng trừu tượng hóa Disk I/O Driver (`diskio.c`), API tập tin (`ff.c`), và tích hợp vào STM32Cube.
2. **Kỹ thuật Lập trình:**
   - Cấu hình khối `SPI2` (`PB13` SCK, `PB14` MISO, `PB15` MOSI, `PB12` CS) kết nối khe cắm thẻ MicroSD.
   - Sử dụng các API FatFS: `f_mount()` gắn kết phân vùng, `f_open()` mở/tạo tệp, `f_printf()` ghi dữ liệu định dạng, `f_sync()` đẩy bộ đệm Flash, và `f_close()`.
   - Lập trình tính năng **Nhật ký Sự kiện Nhúng (Data Logger)**: Tự động ghi nối các dòng dữ liệu định dạng CSV (`alarm_log.csv`) chứa mốc thời gian RTC, tên cảm biến, nhiệt độ và mã lỗi hệ thống khi có báo động (Ex 9.1 $\div$ 9.4).
3. **Quản lý Phiên bản Mã nguồn Dự án Nhúng (Git CLI):**
   - Thiết lập kho lưu trữ Git cục bộ (`git init`) cho project STM32CubeIDE.
   - Cấu hình tệp `.gitignore` chuẩn nhúng (loại trừ các thư mục sinh mã trung gian `Debug/`, `Release/`).
   - Thực hiện quy trình commit, tạo nhánh thử nghiệm (`feature-branch`), hợp nhất (`merge`) và đồng bộ lên kho lưu trữ GitHub từ dòng lệnh.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Module khe cắm thẻ MicroSD SPI | Tích hợp IC hạ áp 3.3V và đệm trở kháng | 1 | Giao tiếp qua chuẩn SPI |
| 3 | Thẻ nhớ MicroSD | Dung lượng $2\text{GB} \div 32\text{GB}$, định dạng FAT32 | 1 | Đã format sẵn FAT32 trên máy tính |
| 4 | Đầu đọc thẻ nhớ USB | Cắm vào cổng USB máy tính | 1 | Kiểm tra trực tiếp tệp CSV |
| 5 | Phần mềm máy tính | STM32CubeIDE, Git CLI (Git Bash) | 1 bộ | Thực hành code & quản lý repo |

---

## 3. THIẾT KẾ MẠCH NGUYÊN LÝ VÀ ĐẤU NỐI CHÂN (PINOUT)

### Bảng phân bổ chân kết nối Module MicroSD SPI vào STM32F407:
| Chân Module MicroSD | Chân STM32F407 | Chức năng phần cứng | Lưu ý cấu hình |
| :---: | :---: | :--- | :--- |
| `CS` | `PB12` | Chọn chip thẻ nhớ (Active-LOW) | GPIO_Output Push-Pull |
| `SCK` / `CLK` | `PB13` | Xung nhịp SPI2_SCK | Alternate Function SPI2 |
| `MISO` / `DO` | `PB14` | Dữ liệu thẻ nhớ gửi về vi điều khiển | Alternate Function SPI2 (Pull-up) |
| `MOSI` / `DI` | `PB15` | Dữ liệu vi điều khiển gửi xuống thẻ | Alternate Function SPI2 |
| `VCC` | `3V3` (hoặc `5V`) | Nguồn cấp cho module thẻ | Nối `3.3V` nếu module không có IC 3V3 |
| `GND` | `GND` | Mass chung hệ thống | Bắt buộc nối mass |

> [!IMPORTANT]
> Thẻ MicroSD chạy ở mức logic chuẩn **$3.3\text{V}$**. Đường `MISO` (`PB14`) nên được bật điện trở kéo lên nội (`Pull-up`) trong STM32CubeMX để tránh xung nhiễu khi thẻ nhớ ở trạng thái ngắt kết nối (High-Z).

---

## 4. QUY TRÌNH CẤU HÌNH TRÊN STM32CubeMX
1. **Cấu hình SPI2 (Khối giao tiếp thẻ nhớ):**
   - Vào *Connectivity* $\rightarrow$ *SPI2*.
   - Mode: **Full-Duplex Master**.
   - Parameter Settings:
     - Data Size: **8 Bits**.
     - First Bit: **MSB First**.
     - Clock Polarity (CPOL): **Low**.
     - Clock Phase (CPHA): **1 Edge**.
     - Baud Rate Prescaler: Ban đầu chọn **/32** hoặc **/64** (Tốc độ khởi tạo ban đầu của thẻ nhớ phải $< 400\text{ kHz}$). Sau khi khởi tạo xong có thể nâng lên **/4** ($10.5\text{ Mbps}$).
2. **Kích hoạt Middleware FATFS:**
   - Vào mục *Middleware* $\rightarrow$ *FATFS*.
   - Tích chọn **User-defined** (hoặc SD Card nếu dùng thư viện có sẵn).
   - Trong tab *Configuration*:
     - `_USE_MKFS`: Chọn **Enabled** (Cho phép format thẻ nhớ bằng code nếu cần).
     - `_CODE_PAGE`: Chọn **Standard English (437)**.
     - `_MAX_SS` (Sector Size): Chọn **512**.
3. **Cấu hình Chân CS (`PB12`):**
   - Cấu hình GPIO_Output, Speed = Very High, Mức ban đầu = High (1).

---

## 5. MÃ NGUỒN GHI NHẬT KÝ FATFS TỪNG BƯỚC

### 5.1. Khởi tạo và ghi tệp nhật ký `alarm_log.csv`
Trong file `Core/Src/main.c`:

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
#include <string.h>
#include "fatfs.h"
/* USER CODE END Includes */

/* USER CODE BEGIN 0 */
FATFS fs;         /* Khối điều khiển hệ thống tập tin FatFS */
FIL fil;          /* Đối tượng tệp tin */
FRESULT fresult;  /* Biến lưu mã kết quả trả về của API FatFS */

/* Ghi một bản ghi nhật ký sự kiện vào tệp CSV trên thẻ nhớ */
uint8_t SD_LogEvent(const char *timestamp, const char *eventType, float tempVal)
{
    /* 1. Gắn kết phân vùng thẻ nhớ (Mount SD Card) */
    fresult = f_mount(&fs, "", 1);
    if (fresult != FR_OK) {
        printf("[FATFS ERROR] Mount that bai! Ma loi: %d\r\n", fresult);
        return 0;
    }

    /* 2. Mở tệp tin ở chế độ Ghi nối đuôi (FA_OPEN_APPEND | FA_WRITE) */
    fresult = f_open(&fil, "alarm_log.csv", FA_OPEN_APPEND | FA_WRITE);
    if (fresult != FR_OK) {
        printf("[FATFS ERROR] Mo tep that bai! Ma loi: %d\r\n", fresult);
        f_mount(NULL, "", 0); /* Unmount */
        return 0;
    }

    /* 3. Nếu tệp mới tạo (kích thước = 0), ghi dòng tiêu đề CSV Header */
    if (f_size(&fil) == 0) {
        f_printf(&fil, "Timestamp,Event,Temperature_C,Status\r\n");
    }

    /* 4. Ghi dòng dữ liệu nhật ký mới */
    f_printf(&fil, "%s,%s,%.2f,ALARM_ACTIVE\r\n", timestamp, eventType, tempVal);

    /* 5. Đồng bộ và đóng tệp tin an toàn */
    f_close(&fil);

    /* 6. Hủy gắn kết phân vùng */
    f_mount(NULL, "", 0);

    printf("[SD LOG] Ghi nhan su kien thanh cong vao alarm_log.csv!\r\n");
    return 1;
}

/* Đọc toàn bộ nội dung tệp CSV truyền ngược lên PC qua UART */
void SD_DumpLogToConsole(void)
{
    char lineBuffer[128];

    if (f_mount(&fs, "", 1) == FR_OK)
    {
        if (f_open(&fil, "alarm_log.csv", FA_READ) == FR_OK)
        {
            printf("\r\n--- NOI DUNG TEP NHAT KY TREN THE NHO SD ---\r\n");
            while (f_gets(lineBuffer, sizeof(lineBuffer), &fil) != NULL)
            {
                printf("%s", lineBuffer);
            }
            printf("--------------------------------------------\r\n");
            f_close(&fil);
        }
        f_mount(NULL, "", 0);
    }
}
/* USER CODE END 0 */
```

### 5.2. Tích hợp ghi thẻ nhớ khi xảy ra sự cố trong Smart Home
```c
/* USER CODE BEGIN WHILE */
while (1)
{
    /* Giả lập khi cảm biến phát hiện sự cố báo động */
    if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) == GPIO_PIN_SET) /* Gas rò rỉ */
    {
        printf("[CANH BAO] Phat hien ro ri Gas! Ghi nhat ky SD...\r\n");
        SD_LogEvent("2026-11-16 12:30:15", "GAS_LEAK_DETECTED", 32.5f);
        HAL_Delay(5000); /* Chờ 5 giây tránh ghi lặp liên tục */
    }

    /* Khi người dùng gõ lệnh 'd' trên bàn phím máy tính PC */
    uint8_t cmd;
    if (HAL_UART_Receive(&huart2, &cmd, 1, 10) == HAL_OK)
    {
        if (cmd == 'd' || cmd == 'D')
        {
            SD_DumpLogToConsole();
        }
    }
}
/* USER CODE END WHILE */
```

---

## 6. THỰC HÀNH QUẢN LÝ MÃ NGUỒN NHÚNG BẰNG GIT CLI

### 6.1. Khởi tạo kho Git và cấu hình `.gitignore`
Sinh viên mở cửa sổ **Git Bash** tại thư mục gốc dự án:
```bash
# 1. Khởi tạo kho Git cục bộ
git init

# 2. Tạo tệp .gitignore để loại bỏ các tệp nhị phân rác do Compiler sinh ra
cat << 'EOF' > .gitignore
# STM32CubeIDE Build Artifacts
Debug/
Release/
*.o
*.d
*.su
*.elf
*.hex
*.bin
*.map

# OS & IDE Configuration files
.settings/
.project
.cproject
Thumbs.db
.DS_Store
EOF
```

### 6.2. Các lệnh thao tác phiên bản chuẩn mực
```bash
# 3. Kiểm tra trạng thái kho lưu trữ
git status

# 4. Thêm toàn bộ mã nguồn sạch vào vùng theo dõi (Staging Area)
git add .

# 5. Tạo Commit phiên bản đầu tiên
git commit -m "feat: complete SD Card FatFS data logging implementation"

# 6. Tạo một nhánh mới để phát triển tính năng không dây
git branch feature-ble-driver
git checkout feature-ble-driver

# 7. Kiểm tra lịch sử commit
git log --oneline --graph
```

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Rút thẻ nhớ cắm vào máy tính, mở tệp `alarm_log.csv` bằng Microsoft Excel hoặc Notepad chụp ảnh kết quả bảng tính có đầy đủ các cột dữ liệu.
- Chụp ảnh màn hình Terminal gõ lệnh `git log` và `git status` chứng minh kho mã nguồn sạch sẽ.

### 7.2. Câu hỏi đánh giá
1. *Tại sao khi kết thúc ghi dữ liệu bắt buộc phải gọi hàm `f_close()` hoặc `f_sync()`? Nếu rút thẻ nhớ đột ngột khi chưa gọi hàm này thì điều gì sẽ xảy ra với dữ liệu trên thẻ?*
2. *Tại sao ở giai đoạn khởi động (Initialization) của thẻ nhớ SD, tần số xung nhịp SPI bắt buộc phải duy trì dưới $400\text{ kHz}$?*
3. *Tại sao các tệp nhị phân như `.elf`, `.hex`, `.o` không nên commit vào kho Git của dự án nhúng?*