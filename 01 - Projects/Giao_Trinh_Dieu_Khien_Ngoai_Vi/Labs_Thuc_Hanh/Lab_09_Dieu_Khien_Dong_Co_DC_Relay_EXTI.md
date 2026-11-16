# BÀI THỰC HÀNH SỐ 09: ĐIỀU KHIỂN ĐỘNG CƠ DC ĐẢO CHIỀU QUA RELAY VÀ CƠ CHẾ NGẮT NGOÀI EXTI (NVIC)
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết mạch cảm kháng $\rightarrow$ Cấu hình NVIC $\rightarrow$ Lập trình ISR $\rightarrow$ Đo kiểm bắt ngắt)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Kiến trúc NVIC Cortex-M4), CLO 2 (Mạch Relay & Diode dập xung), CLO 3 (Lập trình ISR & Biến volatile), CLO 5 (Đo kiểm xung ngắt)  
**Căn cứ giáo trình:** *ARM Education - Chương 7: Direct Current Motors and Interrupts (Ex 7.1 $\div$ 7.4)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Điều khiển Công suất & Cảm kháng:**
   - Phân tích nguyên lý cảm ứng điện từ trong cuộn dây động cơ DC và cuộn hút Relay: Hiện tượng quá áp do sức điện động ngược sinh ra khi ngắt dòng đột ngột ($V = -L \frac{di}{dt}$).
   - Thiết kế mạch bảo vệ sử dụng **Diode dập xung (Flyback Diode)** và cách ly quang qua **Optocoupler PC817** để bảo vệ vi điều khiển khỏi xung nhiễu công nghiệp.
   - Phân tích nguyên lý mạch cầu H đảo chiều động cơ DC bằng 2 Relay SPDT (Single Pole Double Throw).
2. **Kiến trúc Ngắt NVIC (Nested Vectored Interrupt Controller):**
   - Nắm vững cơ chế hoạt động của khối NVIC trên lõi ARM Cortex-M4: Bảng vector ngắt (Vector Table), quá trình tự động cất/phục hồi ngữ cảnh phần cứng (Hardware Stacking/Unstacking), phân cấp mức độ ưu tiên cắt ngắt (Preemption Priority) và ưu tiên phụ (Subpriority).
   - Nắm vững quy tắc cấu hình bộ điều khiển ngắt ngoài **EXTI (Extended Interrupts and Events Controller)**.
   - Hiểu rõ bản chất từ khóa `volatile` trong C để ngăn chặn trình biên dịch tối ưu hóa sai lệch giá trị biến chia sẻ giữa ngắt và chương trình chính.
3. **Kỹ thuật Lập trình:**
   - Ghép nối cảm biến chuyển động hồng ngoại PIR (`PC4`) kích hoạt ngắt `EXTI4` phát hiện có người đến gần cổng.
   - Lập trình chu trình tự động hóa cửa cổng thông minh: Mở cổng (Relay 1 ON) $\rightarrow$ Dừng chờ $\rightarrow$ Đóng cổng (Relay 2 ON).
   - Thiết lập ngắt bảo vệ khẩn cấp quá dòng / kẹt cổng (`PC5`) kích hoạt ngắt ưu tiên cao nhất `EXTI9_5_IRQHandler` dừng tức thời động cơ trong vòng micro-giây (Ex 7.1 $\div$ 7.4).

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | Module 2 Relay cách ly Opto | 5VDC, tiếp điểm 10A/250VAC, kích mức thấp | 1 | Điều khiển động cơ DC |
| 3 | Động cơ DC kèm hộp giảm tốc | Động cơ giảm tốc vàng 3V - 6V DC | 1 | Giả lập cổng trượt tự động |
| 4 | Cảm biến chuyển động PIR | Module HC-SR501 | 1 | Tín hiệu ngõ ra mức logic 3.3V |
| 5 | Nút bấm ngắt khẩn cấp | Nút nhấn nhả có nắp bảo vệ | 1 | Giả lập cảm biến kẹt cửa |
| 6 | Nguồn ngoài cấp cho Động cơ | Bộ nguồn $5\text{V} / 2\text{A}$ độc lập | 1 | Không dùng nguồn 3.3V của STM32 |
| 7 | Thiết bị đo | Oscilloscope 2CH & Logic Analyzer | 1 | Bắt xung kích ngắt và đo độ trễ |

---

## 3. THIẾT KẾ MẠCH NGUYÊN LÝ VÀ ĐẤU NỐI LINH KIỆN

### 3.1. Sơ đồ mạch cầu đảo chiều bằng 2 Relay SPDT
Mỗi Relay có 3 tiếp điểm: Thường hở (NO - Normally Open), Thường đóng (NC - Normally Closed), và Chung (COM - Common).
- Tiếp điểm **NC** của cả 2 Relay nối vào cực âm nguồn ngoài ($GND_{EXT}$).
- Tiếp điểm **NO** của cả 2 Relay nối vào cực dương nguồn ngoài ($+5V_{EXT}$).
- Tiếp điểm **COM** của Relay 1 nối vào Cực 1 của động cơ DC.
- Tiếp điểm **COM** của Relay 2 nối vào Cực 2 của động cơ DC.

**Bảng trạng thái điều khiển động cơ:**
| Relay 1 (PC8) | Relay 2 (PC9) | Điện áp Cực 1 | Điện áp Cực 2 | Trạng thái Động cơ |
| :---: | :---: | :---: | :---: | :--- |
| TẮT (0) | TẮT (0) | $GND$ | $GND$ | **DỪNG (Phanh điện từ)** |
| **BẬT (1)** | TẮT (0) | $+5\text{V}$ | $GND$ | **QUAY THUẬN (MỞ CỔNG)** |
| TẮT (0) | **BẬT (1)** | $GND$ | $+5\text{V}$ | **QUAY NGHỊCH (ĐÓNG CỔNG)** |
| BẬT (1) | BẬT (1) | $+5\text{V}$ | $+5\text{V}$ | **DỪNG (Phanh an toàn - Tuyệt đối không chập mạch)** |

### 3.2. Bảng đấu nối chân hệ thống
| Chân STM32F407 | Ngoại vi / Module | Chức năng | Chế độ cấu hình CubeMX |
| :---: | :---: | :--- | :--- |
| `PC8` | In 1 Module Relay | Lệnh mở cổng (Quay thuận) | GPIO_Output Push-Pull |
| `PC9` | In 2 Module Relay | Lệnh đóng cổng (Quay nghịch) | GPIO_Output Push-Pull |
| `PC4` | Tín hiệu PIR (HC-SR501) | Bắt chuyển động người đến | **GPIO_EXTI4** (Sườn lên) |
| `PC5` | Nút ngắt khẩn cấp / Cảm biến kẹt | Bắt sự cố kẹt cửa | **GPIO_EXTI5** (Sườn xuống / Pull-up) |
| `GND` | Module Relay & PIR | Mass chung tín hiệu logic | Chung mass với STM32 |

> [!CAUTION]
> Dòng khởi động và xung nhiễu của động cơ DC có thể làm sập nguồn vi điều khiển. Bắt buộc phải cấp nguồn động cơ từ bộ đổi nguồn ngoài $5\text{V}$ riêng biệt, chỉ nối chung cực âm mass ($GND$) với bo mạch STM32F407.

---

## 4. CẤU HÌNH TRÊN STM32CubeMX
1. **Cấu hình GPIO ngắt ngoài:**
   - Chân `PC4`: Chọn **GPIO_EXTI4**.
     - GPIO mode: **External Interrupt Mode with Rising edge trigger detection** (Kích hoạt ngắt khi có chuyển động - sườn lên).
     - Pull-up/Pull-down: **No pull-up and no pull-down** (PIR đã có tầng đệm ngõ ra).
   - Chân `PC5`: Chọn **GPIO_EXTI5**.
     - GPIO mode: **External Interrupt Mode with Falling edge trigger detection** (Kích hoạt ngắt khi nhấn nút khẩn cấp - sườn xuống).
     - Pull-up/Pull-down: **Pull-up** (Kéo lên mức cao nội).
2. **Cấu hình NVIC (Mức độ ưu tiên ngắt):**
   - Vào mục *System Core* $\rightarrow$ *NVIC*.
   - Priority Group: **4 bits for pre-emption priority, 0 bits for sub-priority**.
   - Tìm dòng `EXTI line4 interrupt`:
     - Enabled: **Tích chọn**.
     - Preemption Priority: **2** (Mức ưu tiên bình thường).
   - Tìm dòng `EXTI line[9:5] interrupts`:
     - Enabled: **Tích chọn**.
     - Preemption Priority: **0** (**Mức ưu tiên KHẨN CẤP CAO NHẤT - Được phép cắt ngang ngắt PIR**).

---

## 5. MÃ NGUỒN XỬ LÝ NGẮT VÀ ĐIỀU KHIỂN HOÀN CHỈNH

### 5.1. Khai báo biến chia sẻ `volatile` và State Machine Động cơ
Trong file `Core/Src/main.c`:

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>
#include <stdbool.h>

/* Khai báo biến toàn cục chia sẻ với trình phục vụ ngắt BẮT BUỘC có volatile */
volatile bool pirMotionDetected = false;
volatile bool emergencyStopFlag = false;

typedef enum {
    GATE_IDLE_CLOSED,
    GATE_OPENING,
    GATE_WAIT_OPEN,
    GATE_CLOSING,
    GATE_EMERGENCY_STOP
} Gate_State_t;

static Gate_State_t gateState = GATE_IDLE_CLOSED;

/* Các hàm phụ trợ điều khiển tầng công suất */
void Motor_Stop(void) {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_RESET);
}

void Motor_OpenGate(void) {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_SET);
}

void Motor_CloseGate(void) {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_SET);
}
/* USER CODE END 0 */
```

### 5.2. Trình phục vụ ngắt ngoài (Callback ISR)
```c
/* USER CODE BEGIN 4 */
/* Hàm callback do thư viện HAL gọi khi có sự kiện ngắt trên chân EXTI */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    /* 1. NGẮT KHẨN CẤP TRÊN CHÂN PC5 (Ưu tiên mức 0 - Cao nhất) */
    if (GPIO_Pin == GPIO_PIN_5)
    {
        /* Ngắt ngay lập tức tầng công suất động cơ */
        Motor_Stop();
        emergencyStopFlag = true;
        gateState = GATE_EMERGENCY_STOP;
    }
    /* 2. NGẮT PHÁT HIỆN CHUYỂN ĐỘNG PIR TRÊN CHÂN PC4 (Ưu tiên mức 2) */
    else if (GPIO_Pin == GPIO_PIN_4)
    {
        if (!emergencyStopFlag && gateState == GATE_IDLE_CLOSED)
        {
            pirMotionDetected = true;
        }
    }
}
/* USER CODE END 4 */
```

### 5.3. Vòng lặp điều phối chính (Non-blocking State Machine)
```c
/* USER CODE BEGIN WHILE */
uint32_t stateTimer = 0;

while (1)
{
    /* Kiểm tra trạng thái khẩn cấp */
    if (emergencyStopFlag)
    {
        Motor_Stop();
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET); /* Bật LED đỏ báo nguy */
        /* Chờ người dùng nhấn nút giải trừ an toàn để tiếp tục */
        continue;
    }

    switch (gateState)
    {
        case GATE_IDLE_CLOSED:
            Motor_Stop();
            if (pirMotionDetected)
            {
                pirMotionDetected = false;
                gateState = GATE_OPENING;
                stateTimer = HAL_GetTick();
                Motor_OpenGate();
                printf("[GATE] Phat hien nguoi -> DANG MO CONG...\r\n");
            }
            break;

        case GATE_OPENING:
            /* Giả lập hành trình mở cổng kéo dài 3000ms */
            if (HAL_GetTick() - stateTimer >= 3000)
            {
                Motor_Stop();
                gateState = GATE_WAIT_OPEN;
                stateTimer = HAL_GetTick();
                printf("[GATE] Cong da mo hoan toan. Dung cho 5 giay...\r\n");
            }
            break;

        case GATE_WAIT_OPEN:
            /* Dừng chờ xe đi qua trong 5000ms */
            if (HAL_GetTick() - stateTimer >= 5000)
            {
                gateState = GATE_CLOSING;
                stateTimer = HAL_GetTick();
                Motor_CloseGate();
                printf("[GATE] Het thoi gian cho -> DANG DONG CONG...\r\n");
            }
            break;

        case GATE_CLOSING:
            /* Nếu trong khi đang đóng mà có người bước tới (PIR kích hoạt lại) */
            if (pirMotionDetected)
            {
                pirMotionDetected = false;
                printf("[CANH BAO] Co nguoi chan cua! Dao chieu mo lai ngay!\r\n");
                gateState = GATE_OPENING;
                stateTimer = HAL_GetTick();
                Motor_OpenGate();
            }
            /* Hành trình đóng hoàn tất sau 3000ms */
            else if (HAL_GetTick() - stateTimer >= 3000)
            {
                Motor_Stop();
                gateState = GATE_IDLE_CLOSED;
                printf("[GATE] Cong da dong kin an toan.\r\n");
            }
            break;

        case GATE_EMERGENCY_STOP:
            Motor_Stop();
            break;
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & PHÂN TÍCH ĐỘ TRỄ NGẮT (INTERRUPT LATENCY)

### 6.1. Thiết lập đo lường trên Oscilloscope
1. Kênh `CH1`: Cắm vào chân nút bấm khẩn cấp `PC5` (Tín hiệu kích hoạt ngắt).
2. Kênh `CH2`: Cắm vào chân điều khiển Relay `PC8` (Tín hiệu ngắt công suất).
3. Thiết lập Trigger: Falling Edge trên CH1, Single trigger.

### 6.2. Đo thời gian trễ đáp ứng của NVIC
1. Bấm nút dừng khẩn cấp: CH1 sụt từ $3.3\text{V}$ xuống $0\text{V}$.
2. Quan sát khoảng thời gian cho đến khi CH2 sụt từ $3.3\text{V}$ về $0\text{V}$:
   - Thời gian đáp ứng phần cứng NVIC của Cortex-M4 (Stacking 8 thanh ghi $R0 \div R3, R12, LR, PC, xPSR$): Đúng **12 chu kỳ xung nhịp** ($12 \times 5.95\text{ ns} \approx 71.4\text{ ns}$).
   - Thời gian thực thi hàm ngắt: Tổng độ trễ chưa đầy $1\mu s$.
   - Điều này chứng minh ngắt phần cứng bảo vệ an toàn gấp hàng trăm lần so với việc dùng vòng lặp `while` quét tuần tự.

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Sơ đồ nguyên lý đấu nối mạch cầu 2 Relay, Opto và nguồn cách ly.
- Video ngắn hoặc log Terminal ghi nhận quá trình tự động đảo chiều cửa khi gặp vật cản.

### 7.2. Câu hỏi đánh giá
1. *Tại sao các biến chia sẻ giữa chương trình phục vụ ngắt (ISR) và vòng lặp `main` bắt buộc phải được khai báo với từ khóa `volatile`? Nếu không có `volatile`, trình biên dịch có thể tối ưu hóa mã nguồn như thế nào gây ra lỗi logic?*
2. *Nguyên tắc "Vàng" khi viết hàm xử lý ngắt (ISR) là gì? Tại sao tuyệt đối không được gọi các hàm như `HAL_Delay()` hoặc các vòng lặp chờ dài bên trong ISR?*
3. *Sự khác nhau cơ bản giữa Preemption Priority (Ưu tiên cắt ngắt) và Sub-priority (Ưu tiên phụ) trong khối NVIC của ARM Cortex-M4 là gì?*