# BÀI THỰC HÀNH SỐ 04: MÁY TRẠNG THÁI HỮU HẠN (FSM), BÀN PHÍM MA TRẬN 4x4 VÀ ĐỒNG HỒ THỜI GIAN THỰC (RTC)
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết FSM $\rightarrow$ Quét ma trận $\rightarrow$ Lập trình $\rightarrow$ Đo kiểm)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Mô hình FSM Mealy/Moore), CLO 2 (Mạch ma trận phím), CLO 3 (Lập trình FSM & RTC), CLO 5 (Đo kiểm xung quét)  
**Căn cứ giáo trình:** *ARM Education - Chương 4: Finite State Machines and Keypads (Ex 4.1 $\div$ 4.4)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Máy trạng thái (FSM) & Giao tiếp Phím ma trận:**
   - Nắm vững phương pháp luận thiết kế phần mềm nhúng theo mô hình **Máy trạng thái hữu hạn (Finite State Machine - FSM)** kiểu Moore và Mealy; loại bỏ hoàn toàn cấu trúc mã nguồn rườm rà (Spaghetti code).
   - Phân tích nguyên lý ghép nối và thuật toán quét ma trận phím $4\times 4$: 4 chân Hàng (Outputs) và 4 chân Cột (Inputs kèm trở kéo lên Pull-up).
   - Nắm vững cơ chế chống rung phím số (Software Debounce) và bắt sự kiện nhấn đúp (Double-press detection).
   - Tìm hiểu khối ngoại vi RTC (Real-Time Clock) tích hợp của STM32F407 chạy với thạch anh ngoài $32.768\text{ kHz}$ (LSE).
2. **Kỹ thuật Lập trình:**
   - Xây dựng FSM quản lý bàn phím nhận diện từng phím bấm mà không làm trễ CPU.
   - Hiện thực hóa FSM Quản lý Bảo mật Nhà Thông minh (Trạng thái: `DISARMED`, `ARMING`, `ARMED`, `ALARM`, `PIN_ENTRY`).
   - Ghi nhận và xuất nhật ký sự kiện có gắn tem thời gian thực RTC (Timestamp: Giờ:Phút:Giây) lên cổng nối tiếp PC (Ex 4.1 $\div$ 4.4).
3. **Đo kiểm & Bắt dạng sóng:**
   - Dùng Logic Analyzer bắt xung quét tuần tự trên 4 chân Hàng và phát hiện sườn tín hiệu trên các chân Cột khi một nút được bấm.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Bàn phím ma trận $4\times 4$ | Loại màng mỏng (Membrane) hoặc nút cơ | 1 | 16 phím: $0 \div 9, A, B, C, D, *, \#$ |
| 3 | Mạch USB-to-UART | CP2102 / CH340G | 1 | Kết nối PC xem nhật ký sự kiện |
| 4 | Còi Buzzer & LED | Buzzer 5V + Transistor NPN + LED đỏ/xanh | 1 bộ | Báo trạng thái FSM |
| 5 | Thiết bị đo | Logic Analyzer 8CH 24MHz | 1 | Bắt dạng sóng quét ma trận |

---

## 3. NGUYÊN LÝ HOẠT ĐỘNG VÀ SƠ ĐỒ ĐẤU NỐI MẠCH

### 3.1. Thuật toán quét bàn phím ma trận $4\times 4$
- **Cấu hình phần cứng:**
  - 4 chân Hàng (`ROW0..ROW3`): Cấu hình là **Output Open-Drain** (hoặc Output Push-Pull).
  - 4 chân Cột (`COL0..COL3`): Cấu hình là **Input Pull-Up** (bên trong STM32 kéo lên $3.3\text{V}$).
- **Trình tự quét tuần tự:**
  - Ở trạng thái bình thường, kéo cả 4 Hàng xuống mức 0. Nếu không có phím nào được bấm, cả 4 Cột đều đọc được mức 1 nhờ trở kéo lên.
  - Khi có một phím tại vị trí $(Hàng_i, Cột_j)$ được nhấn, chân $Cột_j$ sẽ bị kéo tụt xuống mức 0.
  - Để xác định chính xác phím nào được bấm, vi điều khiển kích hoạt lần lượt từng hàng: Đặt $Hàng_i = 0$ và 3 hàng còn lại $= 1$. Sau đó đọc trạng thái 4 cột. Cột nào có mức 0 chính là phím đang được tác động.

```
          COL0 (PE11)  COL1 (PE12)  COL2 (PE13)  COL3 (PE14)
               |            |            |            |
ROW0 (PE7)  --[1]----------[2]----------[3]----------[A]--
               |            |            |            |
ROW1 (PE8)  --[4]----------[5]----------[6]----------[B]--
               |            |            |            |
ROW2 (PE9)  --[7]----------[8]----------[9]----------[C]--
               |            |            |            |
ROW3 (PE10) --[*]----------[0]----------[#]----------[D]--
```

### 3.2. Bảng phân bổ chân kết nối STM32F407
| Chân STM32 | Chân Bàn phím | Chức năng | Cấu hình CubeMX |
| :---: | :---: | :--- | :--- |
| `PE7` | Row 0 | Quét Hàng 0 (Phím 1, 2, 3, A) | GPIO_Output, Push-Pull, High-Speed |
| `PE8` | Row 1 | Quét Hàng 1 (Phím 4, 5, 6, B) | GPIO_Output, Push-Pull, High-Speed |
| `PE9` | Row 2 | Quét Hàng 2 (Phím 7, 8, 9, C) | GPIO_Output, Push-Pull, High-Speed |
| `PE10` | Row 3 | Quét Hàng 3 (Phím *, 0, #, D) | GPIO_Output, Push-Pull, High-Speed |
| `PE11` | Col 0 | Đọc Cột 0 | GPIO_Input, Pull-Up |
| `PE12` | Col 1 | Đọc Cột 1 | GPIO_Input, Pull-Up |
| `PE13` | Col 2 | Đọc Cột 2 | GPIO_Input, Pull-Up |
| `PE14` | Col 3 | Đọc Cột 3 | GPIO_Input, Pull-Up |
| `PD12` | LED Xanh | Trạng thái DISARMED (An toàn) | GPIO_Output |
| `PD14` | LED Đỏ | Trạng thái ARMED / ALARM (Báo động) | GPIO_Output |

---

## 4. CẤU HÌNH TRÊN STM32CubeMX
1. **System Core & Clock Tree:** Cấu hình thạch anh ngoài HSE 8MHz lên 168MHz.
2. **Khối RTC (Real-Time Clock):**
   - Vào mục *Timers* $\rightarrow$ *RTC*.
   - Tích chọn **Activate Clock Source** và **Activate Calendar**.
   - Tại *Clock Configuration*: Chọn nguồn xung cho RTC là **LSE** ($32.768\text{ kHz}$) hoặc **LSI** ($32\text{ kHz}$).
   - Trong thẻ *Parameter Settings*: Đặt giờ khởi tạo ban đầu (ví dụ: `12:00:00`, Thứ 2, `16/11/2026`).
3. **Cấu hình GPIO Port E:**
   - Chân `PE7..PE10`: Cấu hình Output, Initial state = High.
   - Chân `PE11..PE14`: Cấu hình Input, Pull-up enabled.

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### 5.1. Mô-đun Quét bàn phím ma trận (Non-blocking Debounce)
Thêm vào `Core/Src/main.c`:

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>
#include <string.h>

extern RTC_HandleTypeDef hrtc;
extern UART_HandleTypeDef huart2;

int __io_putchar(int ch) {
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}

/* Ma trận ký tự tương ứng phím bấm */
const char KEYPAD_MAP[4][4] = {
    {'1', '2', '3', 'A'},
    {'4', '5', '6', 'B'},
    {'7', '8', '9', 'C'},
    {'*', '0', '#', 'D'}
};

/* Hàm quét bàn phím trả về ký tự được nhấn (hoặc 0 nếu không nhấn) */
char Keypad_Scan(void)
{
    const uint16_t rowPins[4] = {GPIO_PIN_7, GPIO_PIN_8, GPIO_PIN_9, GPIO_PIN_10};
    const uint16_t colPins[4] = {GPIO_PIN_11, GPIO_PIN_12, GPIO_PIN_13, GPIO_PIN_14};

    for (int r = 0; r < 4; r++)
    {
        /* Kéo tất cả các hàng lên mức cao 1 */
        HAL_GPIO_WritePin(GPIOE, GPIO_PIN_7 | GPIO_PIN_8 | GPIO_PIN_9 | GPIO_PIN_10, GPIO_PIN_SET);
        /* Kéo riêng hàng r xuống mức thấp 0 */
        HAL_GPIO_WritePin(GPIOE, rowPins[r], GPIO_PIN_RESET);

        /* Độ trễ nhỏ vài micro-giây cho điện áp ổn định */
        for(volatile int d = 0; d < 50; d++);

        /* Đọc 4 cột */
        for (int c = 0; c < 4; c++)
        {
            if (HAL_GPIO_ReadPin(GPIOE, colPins[c]) == GPIO_PIN_RESET)
            {
                /* Chờ nhả phím và chống rung cơ khí đơn giản */
                while (HAL_GPIO_ReadPin(GPIOE, colPins[c]) == GPIO_PIN_RESET);
                HAL_Delay(20);
                return KEYPAD_MAP[r][c];
            }
        }
    }
    return 0; /* Không có phím nào được bấm */
}
/* USER CODE END 0 */
```

### 5.2. Hiện thực hóa Máy trạng thái FSM Hệ thống Báo động (Ex 4.1 $\div$ 4.4)
Định nghĩa FSM và vòng lặp vận hành:

```c
/* USER CODE BEGIN 2 */
/* Định nghĩa các trạng thái FSM của Hệ thống Smart Home */
typedef enum {
    STATE_DISARMED,     /* Hệ thống giải trừ, an toàn hoàn toàn */
    STATE_ARMED,        /* Hệ thống đã kích hoạt phòng vệ */
    STATE_ENTRY_DELAY,  /* Đang đếm lùi thời gian cho phép nhập mã PIN */
    STATE_ALARM         /* Đang hú còi báo động khẩn cấp */
} SystemState_t;

SystemState_t currentState = STATE_DISARMED;
char enteredPIN[5] = {0};
uint8_t pinIndex = 0;
const char CORRECT_PIN[] = "1234";
uint32_t entryDelayStartTime = 0;

void Print_Timestamp(void)
{
    RTC_TimeTypeDef sTime = {0};
    RTC_DateTypeDef sDate = {0};
    HAL_RTC_GetTime(&hrtc, &sTime, RTC_FORMAT_BIN);
    HAL_RTC_GetDate(&hrtc, &sDate, RTC_FORMAT_BIN);
    printf("[%02d:%02d:%02d] ", sTime.Hours, sTime.Minutes, sTime.Seconds);
}
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    char key = Keypad_Scan();

    /* MÁY TRẠNG THÁI HỮU HẠN FSM */
    switch (currentState)
    {
        case STATE_DISARMED:
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);   /* LED xanh SÁNG */
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_RESET); /* LED đỏ TẮT */
            
            if (key == 'A') /* Nhấn phím 'A' để Kích hoạt hệ thống phòng vệ */
            {
                currentState = STATE_ARMED;
                Print_Timestamp();
                printf("HE THONG DA KICH HOAT PHONG VE (ARMED)!\r\n");
            }
            break;

        case STATE_ARMED:
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET);   /* LED đỏ SÁNG */

            /* Giả lập có người đột nhập hoặc nhấn phím bất kỳ để nhập pass */
            if (key != 0)
            {
                currentState = STATE_ENTRY_DELAY;
                entryDelayStartTime = HAL_GetTick();
                pinIndex = 0;
                memset(enteredPIN, 0, sizeof(enteredPIN));
                enteredPIN[pinIndex++] = key;
                Print_Timestamp();
                printf("PHAT HIEN THAM NHAP! Con 10 giay de nhap PIN: *\r\n");
            }
            break;

        case STATE_ENTRY_DELAY:
            /* Nhấp nháy LED đỏ cảnh báo thời gian đang cạn dần */
            if ((HAL_GetTick() / 250) % 2 == 0)
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET);
            else
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_RESET);

            if (key >= '0' && key <= '9')
            {
                enteredPIN[pinIndex++] = key;
                printf("*");
                if (pinIndex == 4)
                {
                    enteredPIN[4] = '\0';
                    printf("\r\n");
                    if (strcmp(enteredPIN, CORRECT_PIN) == 0)
                    {
                        currentState = STATE_DISARMED;
                        Print_Timestamp();
                        printf("DUNG MA PIN! GIAI TRU THANH CONG (DISARMED).\r\n");
                    }
                    else
                    {
                        currentState = STATE_ALARM;
                        Print_Timestamp();
                        printf("SAI MA PIN! KICH HOAT COI BAO DONG!\r\n");
                    }
                }
            }

            /* Hết 10 giây mà chưa giải mã xong -> Kích hoạt Báo động */
            if (HAL_GetTick() - entryDelayStartTime >= 10000)
            {
                currentState = STATE_ALARM;
                Print_Timestamp();
                printf("\r\nHET GIO NHAP PIN! BAO DONG KHAN CAP!\r\n");
            }
            break;

        case STATE_ALARM:
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET);
            /* Bật còi Buzzer hú liên tục */
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_11, GPIO_PIN_SET);

            /* Nhấn phím '#' để hủy báo động nếu nhập đúng PIN */
            if (key == '#')
            {
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_11, GPIO_PIN_RESET);
                currentState = STATE_DISARMED;
                Print_Timestamp();
                printf("DA TAT BAO DONG KHAN CAP BOI NGUOI DUNG.\r\n");
            }
            break;
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & PHÂN TÍCH DẠNG SÓNG QUÉT BÀN PHÍM

### 6.1. Kết nối Logic Analyzer
- Cắm kênh `CH0` vào `PE7` (Row 0), `CH1` vào `PE8` (Row 1).
- Cắm kênh `CH2` vào `PE11` (Col 0), `CH3` vào `PE12` (Col 1).
- Mức lấy mẫu: 1MHz hoặc 2MHz.

### 6.2. Phân tích kết quả đo
1. Khi không bấm phím:
   - Các chân Hàng liên tục nhận các xung mức thấp tuần tự xoay vòng (Scanning pulses).
   - Các chân Cột duy trì mức cao $3.3\text{V}$ ổn định không đổi.
2. Khi nhấn giữ phím số '1' (Giao điểm giữa `Row 0` và `Col 0`):
   - Chân `Col 0` xuất hiện các xung mức thấp đồng bộ hoàn hảo với các xung tích cực mức thấp của `Row 0`.
   - Các kênh Cột khác vẫn giữ nguyên mức cao.
   - Nhờ đó vi điều khiển định danh chính xác tuyệt đối tọa độ phím mà không bị nhiễu.

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Sơ đồ chuyển đổi trạng thái FSM (State Transition Diagram) vẽ bằng Mermaid hoặc tay.
- Log Terminal máy tính ghi nhận quá trình chuyển trạng thái và tem thời gian RTC.

### 7.2. Câu hỏi đánh giá
1. *FSM trong bài Lab thuộc loại Moore hay Mealy? Tại sao?*
2. *Nếu người dùng nhấn đồng thời 3 phím trên cùng một cột thì hiện tượng "bóng ma phím" (Key Ghosting) xảy ra như thế nào? Cần lắp thêm linh kiện gì vào bàn phím để triệt tiêu?*
3. *Tại sao thạch anh ngoài của khối RTC luôn có giá trị đặc thù là $32.768\text{ kHz}$?*