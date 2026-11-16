# CHƯƠNG 7: ĐIỀU KHIỂN ĐỘNG CƠ DC BẰNG RELAY VÀ CƠ CHẾ NGẮT THỜI GIAN THỰC TRÊN STM32F407
*(Biên soạn dựa trên Chapter 7: DC Motor Driving using Relays and Interrupts - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 7.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã biết cách điều khiển đóng ngắt tải điện tử cơ bản ở Chương 1; chương này nâng lên điều khiển tải cảm kháng công suất lớn (Động cơ DC đảo chiều) và làm chủ **Cơ chế ngắt ngoài EXTI kết hợp bộ điều khiển ngắt lồng nhau NVIC** trên vi điều khiển STM32F407.
- **Mục tiêu chi tiết:**
  1. **Lý thuyết Điện tử Công suất & Cảm kháng:** Nắm vững hiện tượng phóng điện áp cảm ứng ngược ($V = -L \frac{di}{dt}$) khi ngắt dòng qua cuộn hút Relay và cuộn dây động cơ DC. Thiết kế mạch bảo vệ dùng Diode dập xung (Flyback Diode) và cách ly quang qua Optocoupler PC817.
  2. **Lý thuyết Mạch Cầu Đảo Chiều Động cơ:** Thiết kế mạch cầu H sử dụng 2 Relay SPDT (Single Pole Double Throw), phân tích 4 trạng thái vận hành: Dừng phanh điện từ, Quay thuận (Mở cổng), Quay nghịch (Đóng cổng) và trạng thái dừng an toàn chống ngắn mạch.
  3. **Kiến trúc Ngắt NVIC trên Cortex-M4:** Hiểu rõ cơ chế phần cứng: Bảng vector ngắt (Vector Table), quy trình tự động cất/phục hồi ngữ cảnh 8 thanh ghi phần cứng (Hardware Stacking/Unstacking) trong 12 chu kỳ clock, phân cấp ưu tiên ngắt (Preemption Priority vs Sub-priority), và vai trò sống còn của từ khóa `volatile` trong C.
  4. **Ghép nối Ngoại vi & Cảm biến:**
     - Ghép nối cảm biến chuyển động hồng ngoại thụ động PIR (HC-SR501 trên chân `PC4`) kích hoạt ngắt ngoài `EXTI4`.
     - Ghép nối cảm biến phát hiện sự cố kẹt cửa / quá dòng động cơ (`PC5`) kích hoạt ngắt bảo vệ khẩn cấp ưu tiên cao nhất `EXTI9_5`.
  5. **Lập trình Hệ thống (Examples 7.1 $\div$ 7.4):** Hiện thực hóa chu trình đóng/dừng/mở cổng tự động thông minh bằng máy trạng thái FSM không chặn thời gian (Non-blocking timing).

---

## 7.2. CƠ SỞ LÝ THUYẾT CHUYÊN SÂU

### 7.2.1. Hiện Tượng Cảm Kháng và Mạch Bảo Vệ Tải Cuộn Dây (Inductive Kickback)
Cuộn hút của Relay và cuộn dây phần ứng của động cơ điện một chiều (DC Motor) là các phần tử tích lũy năng lượng dưới dạng từ trường theo định luật Faraday:
$$\mathcal{E} = -L \frac{di}{dt}$$
- Khi cấp điện: Dòng điện qua cuộn dây tăng dần đều theo thời gian, năng lượng từ trường tích lũy là $W = \frac{1}{2} L I^2$.
- Khi ngắt điện đột ngột: Dòng điện bị triệt tiêu tức thời ($dt \to 0 \implies \frac{di}{dt} \to -\infty$), cuộn cảm sinh ra một xung điện áp cảm ứng ngược cực tính (Back-EMF) có biên độ lên tới **hàng trăm đến hàng nghìn Volt**.
- Xung điện áp cao này phóng tia lửa điện phá hủy tiếp điểm cơ khí, đánh thủng mối nối bán dẫn của Transistor điều khiển và truyền xung gai nhiễu cao tần làm treo (Latch-up) vi điều khiển STM32F407.

```
                  +V_Supply (+5V / +12V)
                     |
                     +------------+
                     |            |
                    --- Diode     |
               1N4007^  Flyback  [L] Cuộn cảm Relay / Motor
                     |            |
                     +------------+
                     |
                     |  (Xung âm được ghim an toàn qua Diode)
                     v Collector
                 +-------+
       Base      |  NPN  |
  GPIO --->[R]---| 2N2222|
                 +-------+
                     | Emitter
                    GND
```

#### Vai trò của Diode Dập Xung (Flyback / Freewheeling Diode):
- Mắc song song ngược cực tính với cuộn cảm (Cathode nối vào cực dương nguồn, Anode nối vào cực góp Collector của Transistor).
- Ở trạng thái bình thường: Diode bị phân cực ngược, không dẫn điện.
- Khi Transistor ngắt: Xung điện áp cảm ứng ngược làm Diode phân cực thuận, tạo thành một mạch vòng kín (Closed loop) xả năng lượng từ trường an toàn qua điện trở nội của cuộn dây:
  $$i(t) = I_0 e^{-\frac{R}{L} t}$$
- Nhờ đó, điện áp trên cực Collector của Transistor được ghim an toàn ở mức:
  $$V_{C,\max} = V_{Supply} + V_{F,Diode} \approx V_{Supply} + 0.7\text{V}.$$

#### Vai trò của Mạch Cách Ly Quang (Optocoupler PC817):
- Để ngăn chặn hoàn toàn xung nhiễu công nghiệp và sự chênh lệch thế đất giữa mạch số $3.3\text{V}$ và mạch công suất động cơ $12\text{V} / 24\text{V}$, ta sử dụng bộ cách ly quang PC817:
  - Tầng phát: Diode hồng ngoại ($V_F \approx 1.2\text{V}, I_F \approx 5 \div 10\text{mA}$).
  - Tầng thu: Phototransistor có điện áp cách ly lên tới $V_{iso} = 5000\text{V}_{rms}$.
  - Không có bất kỳ kết nối dây dẫn điện trực tiếp nào giữa vi điều khiển và mạch động lực!

---

### 7.2.2. Mạch Cầu Đảo Chiều Động Cơ DC Bằng 2 Relay SPDT (Single Pole Double Throw)
Mỗi rơ-le SPDT gồm 3 tiếp điểm: Thường đóng (NC - Normally Closed), Thường hở (NO - Normally Open), và Cực chung (COM - Common):
- Nối cực **NC** của cả 2 Relay về cực âm nguồn công suất ($GND_{Motor}$).
- Nối cực **NO** của cả 2 Relay lên cực dương nguồn công suất ($+V_{Motor}$).
- Nối 2 cực của động cơ DC vào 2 tiếp điểm **COM** của Relay 1 và Relay 2.

```
         +V_Motor (+5V / +12V)
           |             |
       +--(NO)         (NO)--+
       |   o             o   |
       |    \           /    |
       |     o COM     o COM |
       |     |         |     |
       |     +--[ M ]--+     |  (Động cơ DC)
       |     |         |     |
       |    /           \    |
       |   o             o   |
       +--(NC)         (NC)--+
           |             |
         GND_Motor     GND_Motor
```

#### Bảng chân trị trạng thái điều khiển động cơ:
| Relay 1 (PC8) | Relay 2 (PC9) | Điện áp Cực 1 ($COM_1$) | Điện áp Cực 2 ($COM_2$) | Chênh lệch thế ($V_1 - V_2$) | Trạng thái Động cơ |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **0 (OFF)** | **0 (OFF)** | $GND$ | $GND$ | $0\text{V}$ | **DỪNG (Phanh điện từ ngắn mạch)** |
| **1 (ON)** | **0 (OFF)** | $+V$ | $GND$ | $+V$ | **QUAY THUẬN (MỞ CỔNG)** |
| **0 (OFF)** | **1 (ON)** | $GND$ | $+V$ | $-V$ | **QUAY NGHỊCH (ĐÓNG CỔNG)** |
| **1 (ON)** | **1 (ON)** | $+V$ | $+V$ | $0\text{V}$ | **DỪNG (Phanh an toàn - KHÔNG BAO GIỜ CHẬP NGUỒN)** |

> [!TIP]
> Ưu điểm vượt trội của mạch cầu 2 Relay SPDT so với mạch cầu H dùng Transistor/MOSFET là: **Không bao giờ xảy ra hiện tượng chập nguồn (Shoot-through fault)** ngay cả khi cả 2 tín hiệu điều khiển cùng ở mức cao do lỗi phần mềm, vì khi đó cả 2 đầu động cơ đều nối lên $+V$, động cơ dừng phanh an toàn tuyệt đối!

---

### 7.2.3. Khối Điều Khiển Ngắt Lồng Nhau NVIC & Ngắt Ngoài EXTI Trên Cortex-M4

#### 1. Bộ điều khiển NVIC (Nested Vectored Interrupt Controller):
- Tích hợp sâu bên trong lõi ARM Cortex-M4, hỗ trợ tới 82 kênh ngắt có thể cấu hình được trên STM32F407.
- **Tự động lưu ngữ cảnh phần cứng (Hardware Stacking):** Khi một ngắt được chấp nhận, phần cứng NVIC tự động đẩy 8 thanh ghi thiết yếu ($R0 \div R3, R12, LR, PC, xPSR$) vào ngăn xếp (Stack) chỉ trong đúng **12 chu kỳ xung nhịp** ($71.4\text{ ns}$ ở 168MHz) mà không cần can thiệp bằng phần mềm.
- **Kỹ thuật Tail-Chaining:** Khi một ngắt mới đến đúng lúc ngắt cũ vừa kết thúc, NVIC không phục hồi rồi cất lại ngữ cảnh mà nhảy trực tiếp sang ngắt mới, giảm độ trễ ngắt kế tiếp xuống chỉ còn **6 chu kỳ clock**!

#### 2. Phân cấp mức độ ưu tiên ngắt (Preemption Priority vs Sub-priority):
Trên STM32F407, thanh ghi ưu tiên ngắt sử dụng 4 bit để phân loại (16 mức độ ưu tiên):
- **Ưu tiên cắt ngắt (Preemption Priority):** Một ngắt có mức ưu tiên số nhỏ hơn (ưu tiên cao hơn) ĐƯỢC PHÉP cắt ngang ngắt có mức ưu tiên số lớn hơn đang thực thi.
- **Ưu tiên phụ (Sub-priority):** Khi có hai ngắt cùng mức Preemption xuất hiện đồng thời, ngắt nào có Sub-priority nhỏ hơn sẽ được CPU phục vụ trước; tuy nhiên ngắt này KHÔNG ĐƯỢC PHÉP cắt ngang ngắt kia nếu ngắt kia đã bắt đầu chạy.

```
Ví dụ cấu hình hệ thống:
- EXTI line[9:5] (Sự cố kẹt cửa khẩn cấp): Preemption = 0 (ƯU TIÊN CAO NHẤT)
- EXTI line4 (Phát hiện người PIR):        Preemption = 2 (Ưu tiên thấp hơn)

Khi CPU đang xử lý ngắt PIR (Preemption = 2), nếu cảm biến kẹt cửa kích hoạt (Preemption = 0), 
NVIC lập tức CẮT NGANG ngắt PIR để dừng khẩn cấp động cơ trong vòng micro-giây!
```

#### 3. Bộ điều khiển ngắt ngoài EXTI (Extended Interrupts and Events Controller):
- STM32F407 có 23 đường ngắt ngoài độc lập.
- Đường `EXTI4` có thể ánh xạ vào chân `Pin 4` của bất kỳ cổng nào: `PA4, PB4, PC4, PD4, PE4` thông qua bộ ghép kênh `SYSCFG->EXTICR`.
- Hỗ trợ bắt sườn xung: Sườn lên (Rising), Sườn xuống (Falling), hoặc cả hai sườn (Both edges).

#### 4. Bản chất từ khóa `volatile` trong C:
Khi một biến được chia sẻ giữa chương trình phục vụ ngắt (ISR) và vòng lặp `while(1)` chính:
```c
volatile uint8_t motionDetectedFlag = 0;
```
- Nếu **KHÔNG CÓ** `volatile`: Trình biên dịch GCC khi bật cờ tối ưu hóa (`-O2` / `-O3`) sẽ thấy trong vòng lặp `while(1)` không có lệnh nào ghi vào biến này, nên nó tải biến vào thanh ghi CPU $R4$ một lần duy nhất và đọc mãi giá trị cũ từ thanh ghi $R4$. Dù ngắt có xảy ra và ghi vào biến trong RAM, vòng lặp chính cũng không bao giờ nhận biết được $\rightarrow$ **Chương trình bị treo vĩnh viễn!**
- Khi **CÓ** `volatile`: Ép trình biên dịch bắt buộc phải sinh lệnh truy xuất trực tiếp vào ô nhớ RAM mỗi khi đọc/ghi biến này.

---

## 7.3. SƠ ĐỒ NGUYÊN LÝ & SƠ ĐỒ ĐẤU NỐI CHÂN CHI TIẾT

### 7.3.1. Sơ đồ mạch nguyên lý điện tử (Schematics)

```
=============================================================================================
1. MẠCH ĐIỀU KHIỂN CẦU 2 RELAY ĐẢO CHIỀU ĐỘNG CƠ QUA OPTOCOUPLER
=============================================================================================
STM32F407           R_in = 330 Ohm         +5V_EXT
[ PC8 ] -----------/\/\/\/\/---->|---+       |
(Mở cổng)                       Opto |     [===] R_pull = 1k
[ GND ] ------------------------|<---+       |
                                PC817        +---------> Base Transistor Q1 (2N2222)
                                             |           Collector Q1 -> Cuộn hút Relay 1
                                             v           Emitter Q1   -> GND_EXT
                                            GND_EXT
(Tương tự chân PC9 điều khiển qua Opto thứ 2 lái Cuộn hút Relay 2)


=============================================================================================
2. MẠCH CẢM BIẾN CHUYỂN ĐỘNG HỒNG NGOẠI PIR (HC-SR501)
=============================================================================================
      Module PIR HC-SR501                 STM32F407
     +-------------------+              +-----------+
     | VCC (Nguồn 5V)    |--------------| 5V        |
     |                   |              |           |
     | OUT (Tín hiệu 3.3V)|------------>| PC4 (EXTI4| (Ngắt bắt sườn lên Rising Edge)
     |                   |              |           |
     | GND (Mass)        |--------------| GND       |
     +-------------------+              +-----------+


=============================================================================================
3. MẠCH NÚT NGẮT DỪNG KHẨN CẤP / CẢM BIẾN KẸT CỬA (ACTIVE-LOW)
=============================================================================================
        +3.3V
          |
         [R] R_pullup = 10k Ohm
          |
  PC5 ----+-------------------o/ o---------------- GND
  (EXTI5 Ngắt sườn xuống)    Nút dừng khẩn cấp
```

---

### 7.3.2. Bảng Phân Bổ Chân Toàn Diện Trên STM32F407 (Master Pinout Table)

| Tên Chân STM32 | Tên Tín Hiệu | Chức Năng Phần Cứng / Ngắt | Cấu Hình Trên STM32CubeMX | Kết Nối Linh Kiện Ngoại Vi | Mức Điện Áp & Lưu Ý Kỹ Thuật |
| :---: | :---: | :---: | :---: | :--- | :--- |
| `PC4` | `PIR_TRIG` | `GPIO_EXTI4` | External Interrupt Mode with Rising edge trigger | Nối vào chân OUT của cảm biến chuyển động PIR HC-SR501 | Mức logic $3.3\text{V}$, không cần trở kéo ngoài |
| `PC5` | `EMG_STOP` | `GPIO_EXTI5` | External Interrupt Mode with Falling edge trigger | Nối vào nút dừng khẩn cấp / cảm biến dòng kẹt cửa | Mức logic $3.3\text{V}$, bật điện trở kéo lên nội (`PULLUP`) |
| `PC8` | `RELAY_FWD` | GPIO_Output | Output Push-Pull, High Speed | Nối vào chân IN1 của module 2 Relay (Lệnh mở cổng) | $3.3\text{V}$, cách ly quang qua Opto PC817 |
| `PC9` | `RELAY_REV` | GPIO_Output | Output Push-Pull, High Speed | Nối vào chân IN2 của module 2 Relay (Lệnh đóng cổng) | $3.3\text{V}$, cách ly quang qua Opto PC817 |
| `PD12` | `LED_GREEN` | GPIO_Output | Output Push-Pull | Đèn báo cổng đang mở an toàn | Mức logic 3.3V qua trở hạn dòng $330\Omega$ |
| `PD14` | `LED_RED` | GPIO_Output | Output Push-Pull | Đèn báo cổng đang đóng hoặc có sự cố khẩn cấp | Mức logic 3.3V qua trở hạn dòng $330\Omega$ |
| `5V` | `VCC_PIR` | Nguồn $5\text{V}$ | Nguồn nuôi cảm biến | Cấp cho chân VCC của module PIR HC-SR501 | Lấy từ chân 5V của mạch nạp USB |
| `GND` | `GND` | Mass hệ thống | Mass chung | Nối chung mass STM32 và mass PIR | **Nguồn ngoài động cơ chỉ nối chung mass** |

---

## 7.4. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 7.1 ĐẾN 7.4 TRÊN STM32F407

### 7.4.1. Cấu hình STM32CubeMX Cho Ngắt và GPIO
1. **Cấu hình chân ngắt:**
   - Chân `PC4`: Chọn **GPIO_EXTI4**. Mode: **Rising edge**. Pull-up/Pull-down: **No pull-up and no pull-down**.
   - Chân `PC5`: Chọn **GPIO_EXTI5**. Mode: **Falling edge**. Pull-up/Pull-down: **Pull-up**.
2. **Cấu hình chân ngõ ra Relay:**
   - Chân `PC8, PC9`: Chọn **GPIO_Output**, Output Level: **Low**, Speed: **High**.
3. **Cấu hình NVIC (Nested Vectored Interrupt Controller):**
   - Priority Group: **4 bits for pre-emption priority, 0 bits for sub-priority**.
   - `EXTI line[9:5] interrupts` (Chân `PC5` ngắt dừng khẩn cấp): Preemption Priority = **0** (Mức cao nhất).
   - `EXTI line4 interrupt` (Chân `PC4` cảm biến PIR): Preemption Priority = **2** (Mức trung bình).

---

### 7.4.2. Mã Nguồn Hoàn Chỉnh Của Chương

#### 1. Khai báo Biến Chia Sẻ Ngắt và Định Nghĩa Máy Trạng Thái FSM
```c
#include "main.h"
#include <stdbool.h>
#include <stdio.h>

/* Khai báo biến cờ hiệu ngắt bắt buộc có volatile */
volatile bool pirMotionFlag = false;
volatile bool emergencyStopFlag = false;

/* Các trạng thái của Cổng Tự Động */
typedef enum {
    GATE_IDLE_CLOSED,   /* Cổng đang đóng, chờ người tới */
    GATE_OPENING,       /* Đang quay thuận mở cổng */
    GATE_WAIT_OPEN,     /* Cổng đã mở toang, dừng chờ xe đi qua */
    GATE_CLOSING,       /* Đang quay nghịch đóng cổng */
    GATE_EMERGENCY      /* Sự cố khẩn cấp: Dừng tức thời */
} GateState_t;

static GateState_t currentGateState = GATE_IDLE_CLOSED;
static uint32_t stateTimer = 0;

/* Các hàm phụ trợ điều khiển tầng công suất động cơ */
void Motor_Stop(void)
{
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_RESET);
}

void Motor_OpenGate(void)
{
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_SET); /* Relay 1 ON */
}

void Motor_CloseGate(void)
{
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_8, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_9, GPIO_PIN_SET); /* Relay 2 ON */
}
```

---

#### 2. Trình Phục Vụ Ngắt Ngoài Callback ISR (Examples 7.1 & 7.3)
```c
/* Hàm callback do thư viện HAL gọi tự động khi xảy ra ngắt EXTI */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    /* 1. NGẮT KHẨN CẤP TRÊN CHÂN PC5 (Ưu tiên mức 0 - Cao nhất) */
    if (GPIO_Pin == GPIO_PIN_5)
    {
        /* Ngắt ngay lập tức điện áp cấp cho động cơ trong vòng micro-giây */
        Motor_Stop();
        emergencyStopFlag = true;
        currentGateState = GATE_EMERGENCY;
    }
    /* 2. NGẮT PHÁT HIỆN CHUYỂN ĐỘNG TRÊN CHÂN PC4 (Ưu tiên mức 2) */
    else if (GPIO_Pin == GPIO_PIN_4)
    {
        /* Chỉ ghi nhận chuyển động khi hệ thống không có sự cố khẩn cấp */
        if (!emergencyStopFlag && currentGateState == GATE_IDLE_CLOSED)
        {
            pirMotionFlag = true;
        }
    }
}
```

---

#### 3. Máy Trạng Thái FSM Điều Khiển Cổng Không Chặn Thời Gian (Examples 7.2 & 7.4)
```c
void Gate_StateMachine_Process(void)
{
    uint32_t currentMillis = HAL_GetTick();

    /* Xử lý trạng thái dừng khẩn cấp */
    if (emergencyStopFlag)
    {
        Motor_Stop();
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET); /* Bật LED đỏ báo nguy */
        return;
    }

    switch (currentGateState)
    {
        case GATE_IDLE_CLOSED:
            Motor_Stop();
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_RESET);

            /* Khi ngắt PIR báo có người */
            if (pirMotionFlag)
            {
                pirMotionFlag = false;
                currentGateState = GATE_OPENING;
                stateTimer = currentMillis;
                Motor_OpenGate();
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET); /* LED xanh sáng */
                printf("[GATE] Phat hien nguoi -> DANG MO CONG...\r\n");
            }
            break;

        case GATE_OPENING:
            /* Cổng mở trong 4 giây */
            if (currentMillis - stateTimer >= 4000)
            {
                Motor_Stop();
                currentGateState = GATE_WAIT_OPEN;
                stateTimer = currentMillis;
                printf("[GATE] Cong da mo hoan toan. Dung cho 5 giay...\r\n");
            }
            break;

        case GATE_WAIT_OPEN:
            /* Dừng chờ xe/người đi qua trong 5 giây */
            if (currentMillis - stateTimer >= 5000)
            {
                currentGateState = GATE_CLOSING;
                stateTimer = currentMillis;
                Motor_CloseGate();
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET); /* LED đỏ sáng */
                printf("[GATE] Het gio cho -> DANG DONG CONG...\r\n");
            }
            break;

        case GATE_CLOSING:
            /* TÍNH NĂNG AN TOÀN THÔNG MINH:
               Nếu đang đóng mà người khác bước vào (PIR kích hoạt lại) -> Đảo chiều mở lại ngay! */
            if (pirMotionFlag)
            {
                pirMotionFlag = false;
                printf("[CANH BAO] Co nguoi chan cua! Dao chieu mo cong ngay lap tuc!\r\n");
                currentGateState = GATE_OPENING;
                stateTimer = currentMillis;
                Motor_OpenGate();
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_RESET);
            }
            /* Hành trình đóng hoàn tất sau 4 giây */
            else if (currentMillis - stateTimer >= 4000)
            {
                Motor_Stop();
                currentGateState = GATE_IDLE_CLOSED;
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_RESET);
                printf("[GATE] Cong da dong kin an toan.\r\n");
            }
            break;

        case GATE_EMERGENCY:
            Motor_Stop();
            break;
    }
}
```

---

## 7.5. "UNDER THE HOOD" & CASE STUDY (THEO GIÁO TRÌNH ARM)

### 7.5.1. Under the Hood (7.3.1): Phân Tích Độ Trễ Ngắt (Interrupt Latency) Của NVIC
- **Độ trễ ngắt (Interrupt Latency):** Là khoảng thời gian tính từ khi tín hiệu ngắt xuất hiện trên chân vật lý cho đến khi câu lệnh đầu tiên trong hàm phục vụ ngắt (ISR) bắt đầu được thực thi.
- Trên lõi ARM Cortex-M4 của STM32F407:
  $$\text{Độ trễ phần cứng NVIC} = 12 \text{ chu kỳ xung nhịp}$$
  Với xung nhịp $f_{CPU} = 168\text{ MHz}$ ($T_{clk} \approx 5.95\text{ ns}$):
  $$t_{latency} = 12 \times 5.95\text{ ns} \approx 71.4\text{ ns}!$$
- So sánh với giải pháp Polling tuần tự (vòng lặp `while` quét chân): Nếu vòng lặp chính bận gọi một hàm mất $50\text{ms}$, thời gian đáp ứng sự cố có thể trễ tới $50\text{ms} = 50{,}000{,}000\text{ ns}$ (chậm hơn gần **$700{,}000$ lần** so với ngắt phần cứng, hoàn toàn không thể bảo vệ an toàn khi kẹt người).

### 7.5.2. Case Study (7.4.1): Hệ Thống Cửa Trượt Tự Động Trung Tâm Thương Mại (Commercial Sliding Doors)
- **Yêu cầu kỹ thuật:** Cửa trượt tại các sân bay và siêu thị phục vụ hàng nghìn lượt người mỗi giờ, bắt buộc phải tuân thủ chuẩn an toàn quốc tế EN 16005:
  1. *Cảm biến Radar vi ba (Microwave Motion Sensor):* Phát hiện người tiến lại gần từ khoảng cách 3-5 mét để mở cửa từ sớm.
  2. *Cảm biến màng quang điện hồng ngoại (Safety Light Curtain):* Chiếu các chùm tia hồng ngoại song song chắn ngang khe cửa. Nếu có bất kỳ đứa trẻ hay vật nuôi nào đứng chặn giữa khe cửa, ngắt NVIC ưu tiên cao lập tức ngăn cấm động cơ đóng cửa và phát chuông cảnh báo.
  3. *Cơ cấu nhả cơ khí khẩn cấp (Fail-Safe Breakout):* Khi mất điện toàn tòa nhà, một bộ tụ điện hoặc ắc-quy dự phòng kích hoạt Relay xả chốt, lò xo kéo bung cửa mở toang để thoát hiểm hỏa hoạn.

---

## 7.6. CÂU HỎI ÔN TẬP VÀ BÀI TẬP THIẾT KẾ CHƯƠNG 7

1. **Câu hỏi 1 (Lý thuyết mạch cảm kháng):** *Hãy giải thích tại sao khi ngắt dòng qua cuộn cảm, điện áp sinh ra lại có cực tính ngược với điện áp cấp ban đầu? Nếu lắp ngược chiều Diode dập xung (Anode lên cực dương nguồn, Cathode nối Collector) thì điều gì sẽ xảy ra khi Transistor bật dẫn?*
2. **Câu hỏi 2 (Kiến trúc NVIC):** *Phân biệt sự khác nhau giữa Preemption Priority và Sub-priority trong khối NVIC của ARM Cortex-M4. Cho ví dụ cụ thể về trường hợp hai ngắt cùng xảy ra đồng thời.*
3. **Câu hỏi 3 (Lập trình nhúng C):** *Giải thích tại sao một biến dùng trong hàm ngắt bắt buộc phải có từ khóa `volatile`. Nếu không có `volatile`, trình biên dịch sẽ tối ưu hóa mã nguồn như thế nào và gây ra hậu quả gì?*
4. **Bài tập thiết kế mạch:** *Thiết kế sơ đồ nguyên lý mạch cầu H đảo chiều động cơ DC $24\text{V} / 2\text{A}$ sử dụng 2 Relay SPDT $24\text{V}$, có cách ly quang bằng IC PC817, có đèn LED chỉ thị chiều quay và nút bấm dừng khẩn cấp. Tính toán đầy đủ giá trị các linh kiện.*
