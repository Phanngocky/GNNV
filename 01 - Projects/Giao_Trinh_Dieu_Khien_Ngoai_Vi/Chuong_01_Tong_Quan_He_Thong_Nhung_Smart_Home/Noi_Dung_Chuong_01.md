# CHƯƠNG 1: TỔNG QUAN HỆ THỐNG NHÚNG VÀ HỆ THỐNG BÁO ĐỘNG NHÀ THÔNG MINH ĐẦU TIÊN TRÊN STM32F407
*(Biên soạn dựa trên Chapter 1: Introduction to Embedded Systems - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 1.1. MỤC TIÊU VÀ ĐỐI TƯỢNG HỌC TẬP
- **Yêu cầu tiên quyết:** Sinh viên đã hoàn thành học phần Vi điều khiển cơ bản (đã nắm vững kiến trúc CPU, bộ nhớ, phép toán thao tác bit, lập trình C cơ bản).
- **Mục tiêu chương:**
  1. Chuyển dịch tư duy từ vi điều khiển 8-bit/16-bit truyền thống sang dòng vi điều khiển 32-bit hiệu năng cao **ARM Cortex-M4 (STM32F407)**: Kiến trúc Harvard đa bus (Multi-AHB Bus Matrix), không gian nhớ 4GB phẳng và cây phân phối xung nhịp phức hợp (RCC Clock Tree).
  2. Tiếp cận phương pháp luận "Learn-by-Doing" xuyên suốt của giáo trình: Làm quen với **Dự án Trục - Hệ thống Giám sát & Báo động Nhà Thông minh (Smart Home System)**.
  3. Hiện thực hóa phiên bản đầu tiên của hệ thống báo động trên kit STM32F407 (STM32F4-Discovery / STM32F407VET6): Ghép nối cảm biến khí gas, cảm biến nhiệt độ, còi đèn báo động, chốt trạng thái và giải mã mật mã (Code 1.1 $\div$ 1.5).
  4. Nắm vững cấu trúc tầng I/O của STM32F407: Push-pull, Open-drain, Pull-up, Pull-down, tính toán mạch ghép nối LED hạn dòng và mạch bảo vệ tải.

---

## 1.2. ĐẶC TÍNH PHẦN CỨNG VI ĐIỀU KHIỂN STM32F407
Vi điều khiển **STM32F407VGT6 / STM32F407VET6** thuộc dòng vi điều khiển hiệu năng cao của STMicroelectronics:
- **Lõi xử lý:** 32-bit ARM Cortex-M4F tích hợp phần cứng FPU (Single-precision Floating Point Unit) và tập lệnh DSP đơn chu kỳ.
- **Tốc độ xung nhịp cực đại:** $168\text{ MHz}$ (đạt 210 DMIPS / 1.25 DMIPS/MHz).
- **Bộ nhớ nội:** $1\text{ MB} \div 512\text{ KB}$ Flash nạp chương trình, $192\text{ KB}$ SRAM (gồm 112KB SRAM1, 16KB SRAM2 và 64KB CCM Core Coupled Memory tốc độ cao).
- **Hệ thống Bus Ma trận (32-bit Multi-AHB Bus Matrix):** Kết nối 8 Master (CPU I/D/S-bus, DMA1, DMA2, Ethernet, USB) với 7 Slave (Flash, SRAM1, SRAM2, AHB1, AHB2, APB1, APB2) cho phép truyền dữ liệu đồng thời mà không nghẽn bus.
- **Hệ thống cấp xung nhịp (RCC):**
  - Thạch anh ngoài $HSE = 8\text{ MHz}$ qua bộ nhân tần PLL ($PLLM=8, PLLN=336, PLLP=2$) tạo ra:
    $$f_{SYSCLK} = \frac{8\text{ MHz}}{8} \times \frac{336}{2} = 168\text{ MHz}$$
  - Bus ngoại vi APB1 (Low-speed): Tần số tối đa $42\text{ MHz}$ (Timer bus APB1 nhân đôi lên $84\text{ MHz}$).
  - Bus ngoại vi APB2 (High-speed): Tần số tối đa $84\text{ MHz}$ (Timer bus APB2 nhân đôi lên $168\text{ MHz}$).

---

## 1.3. PHÂN BỔ PHẦN CỨNG HỆ THỐNG SMART HOME TRÊN STM32F407 (PHIÊN BẢN 1)

Để đảm bảo đồng nhất với các bài học kế tiếp mà không bị xung đột chân phần cứng, bảng phân bổ chân trên STM32F407 được chuẩn hóa như sau:

| Tên biến trong sách ARM | Thiết bị phần cứng ngoại vi | Chân Pin STM32F407 | Chế độ cấu hình I/O | Mức logic tích cực |
| :--- | :--- | :---: | :--- | :---: |
| `gasDetector` | Cảm biến phát hiện khí gas | `PE2` | Input Pull-down | Mức 1 (Khi có gas) |
| `overTempDetector` | Cảm biến phát hiện quá nhiệt | `PE3` | Input Pull-down | Mức 1 (Khi quá nhiệt) |
| `aButton` | Nút bấm giải mã A | `PE4` | Input Pull-down | Mức 1 (Khi nhấn nút) |
| `bButton` | Nút bấm giải mã B | `PE5` | Input Pull-down | Mức 1 (Khi nhấn nút) |
| `alarmLed` | Đèn/Còi báo động chính | `PD12` (LED Xanh lá kit Discovery) | Output Push-Pull | Mức 1: Bật báo động |
| `incorrectCodeLed` | Đèn báo nhập sai mật mã | `PD14` (LED Đỏ kit Discovery) | Output Push-Pull | Mức 1: Báo sai mã |
| `systemBlockedLed` | Đèn báo hệ thống bị khóa | `PD15` (LED Xanh dương kit Discovery) | Output Push-Pull | Mức 1: Báo khóa hệ thống |

---

## 1.4. NGUYÊN LÝ GHÉP NỐI ĐIỆN TỬ CHO SINH VIÊN CHUYÊN NGÀNH

### 1. Mạch ngõ vào cảm biến số & Nút nhấn (Chế độ Pull-down)
- Chân vi điều khiển được cấu hình kích hoạt điện trở kéo xuống nội ($R_{PD} \approx 40\text{ k}\Omega$) hoặc trở kéo xuống ngoài $10\text{ k}\Omega$:
- Khi nút hở: Chân `PE2..PE5` được kéo về $0\text{V}$ (Mức logic 0 chắc chắn, chống nhiễu bắt sóng trôi nổi).
- Khi nút đóng: Tiếp điểm nối lên đường nguồn $3.3\text{V}$, dòng điện nạp vào chân MCU làm điện áp nhảy lên $3.3\text{V}$ (Mức logic 1).

```
        +3.3V
          |
         o/ o  Công tắc / Nút bấm (PE2..PE5)
          |
  PE2 ----+---------------------------------> Chân GPIO Input STM32F407
          |
         [R]   R_pulldown = 10k Ohm (Trở kéo xuống ngoài)
          |
         GND
```

### 2. Mạch ngõ ra điều khiển LED báo động và Tải còi Buzzer
- Chân `PD12` cấu hình **Output Push-Pull**, có khả năng cấp dòng (Source) lên tới $8\text{ mA} \div 20\text{ mA}$.
- Công thức tính toán điện trở hạn dòng LED màu xanh lá ($V_F \approx 2.2\text{ V}, I_F = 5\text{ mA}$):
  $$R_{limit} = \frac{V_{OH} - V_F}{I_F} = \frac{3.3\text{ V} - 2.2\text{ V}}{0.005\text{ A}} = 220\text{ }\Omega \implies \text{Chọn điện trở chuẩn } 220\Omega \div 330\Omega.$$

```
STM32F407                  R_limit = 220 - 330 Ohm
[ PD12 / GPIO_Out ] ------------/\/\/\/\/------------>|-----------+ (Anode LED)
                                                      LED 5mm     |
[ GND             ] ----------------------------------------------+ (Cathode LED)
```

- Đối với còi Buzzer hoặc Relay đóng cắt còi hú công suất lớn 12V: Bắt buộc dùng Transistor đệm dòng (C1815 / 2N2222) và diode dập xung ngược $1N4007$:

```
                            +5V / +12V
                                 |
                                 +---------------+
                                 |               |
                                --- Diode       [+] Còi Báo Động
                           1N4007^  Dập Xung   [BUZZER] Piezo Buzzer
                                 |               |
                                 +---------------+
                                 |
                                 | Collector
                             +-------+
         Base                |  NPN  |
  PD11 ------/\/\/\/\/-------| 2N2222|
             R_B = 1k Ohm    +-------+
                                 | Emitter
                                GND
```

---

## 1.5. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ CODE 1.1 ĐẾN 1.5 (SÁCH ARM) TRÊN STM32F407

### VÍ DỤ 1.1: KÍCH HOẠT BÁO ĐỘNG KHI PHÁT HIỆN KHÍ GAS (Listing 1.1)
- **Yêu cầu:** Giám sát liên tục cảm biến gas trên chân `PE2`. Nếu có khí gas (`PE2 == 1`), bật còi/đèn báo động trên chân `PD12`. Nếu không có gas, tắt đèn báo động.

#### Mã nguồn chuẩn STM32 HAL (C99 - STM32CubeIDE cho STM32F407):
```c
#include "main.h"

int main(void)
{
    HAL_Init();
    SystemClock_Config(); // Cấu hình xung 168MHz

    // 1. Cấp xung clock cho Port D và Port E trên bus AHB1
    __HAL_RCC_GPIOD_CLK_ENABLE();
    __HAL_RCC_GPIOE_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct = {0};

    // 2. Cấu hình chân PD12 là ngõ ra Output Push-Pull cho alarmLed
    GPIO_InitStruct.Pin = GPIO_PIN_12;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);

    // 3. Cấu hình chân PE2 là ngõ vào Input Pull-down cho gasDetector
    GPIO_InitStruct.Pin = GPIO_PIN_2;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLDOWN;
    HAL_GPIO_Init(GPIOE, &GPIO_InitStruct);

    // Ban đầu tắt báo động
    HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);

    // Vòng lặp điều khiển chính (Superloop)
    while (1)
    {
        if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) == GPIO_PIN_SET)
        {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);   // Bật báo động
        }
        else
        {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET); // Tắt báo động
        }
    }
}
```

---

### VÍ DỤ 1.2: BÁO ĐỘNG KHI CÓ KHÍ GAS HOẶC QUÁ NHIỆT (Listing 1.2)
- **Yêu cầu:** Bổ sung cảm biến quá nhiệt trên chân `PE3`. Báo động kích hoạt khi có Gas **HOẶC** Quá nhiệt (Toán tử logic OR `||`).
```c
// Vòng lặp điều khiển chính:
while (1)
{
    uint8_t gas  = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2);
    uint8_t temp = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3);

    if (gas || temp)
    {
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
    }
    else
    {
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
    }
}
```

---

### VÍ DỤ 1.3: CHỐT TRẠNG THÁI BÁO ĐỘNG (LATCHING ALARM) (Listing 1.3)
- **Tư duy nhúng:** Tránh trường hợp khí gas thoáng qua rồi hết làm tắt còi báo. Hệ thống cần chốt trạng thái bằng biến cờ hiệu `alarmState`.
```c
uint8_t alarmState = 0; // Biến trạng thái chốt

while (1)
{
    if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) || HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3))
    {
        alarmState = 1; // Kích hoạt chốt
    }

    if (alarmState)
    {
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
    }
}
```

---

### VÍ DỤ 1.4: TẮT BÁO ĐỘNG BẰNG TỔ HỢP PHÍM BẢO MẬT (Listing 1.4)
- **Yêu cầu:** Người dùng muốn tắt báo động phải bấm đồng thời cả 2 nút nhấn `aButton` (`PE4`) và `bButton` (`PE5`). Nếu bấm sai tổ hợp, đèn `incorrectCodeLed` (`PD14`) sẽ sáng báo hiệu.
```c
uint8_t alarmState = 0;

while (1)
{
    if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) || HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3))
    {
        alarmState = 1;
    }

    if (alarmState)
    {
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);

        uint8_t a = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_4);
        uint8_t b = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_5);

        if (a && b) // Nhập đúng mã: Cả A và B cùng nhấn
        {
            alarmState = 0; // Tắt chốt báo động
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_RESET); // Tắt đèn báo sai mã
        }
        else if (a || b) // Nhấn chỉ A hoặc chỉ B: Sai mã
        {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET);   // Bật đèn báo sai mã
        }
    }
}
```

---

### VÍ DỤ 1.5: ĐẾM SỐ LẦN NHẬP SAI VÀ KHÓA CỨNG HỆ THỐNG (Listing 1.5)
- **Yêu cầu:** Nếu nhập sai mã quá 5 lần, chuyển hệ thống sang trạng thái khóa an toàn (`systemBlockedLed` sáng trên `PD15`), còi hú liên tục và vô hiệu hóa nút mở khóa.
```c
uint8_t alarmState = 0;
uint8_t incorrectCodeCount = 0;
uint8_t systemBlocked = 0;

while (1)
{
    if (systemBlocked)
    {
        // Khi bị khóa: Giữ còi báo và bật LED báo khóa hệ thống vĩnh viễn
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_15, GPIO_PIN_SET);
        continue;
    }

    if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2) || HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3))
    {
        alarmState = 1;
    }

    if (alarmState)
    {
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);

        uint8_t a = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_4);
        uint8_t b = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_5);

        if (a && b) // Đúng mã
        {
            alarmState = 0;
            incorrectCodeCount = 0;
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_RESET);
        }
        else if (a || b) // Sai mã
        {
            HAL_Delay(250); // Chống rung phím cơ bản
            incorrectCodeCount++;
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_14, GPIO_PIN_SET);

            if (incorrectCodeCount >= 5)
            {
                systemBlocked = 1; // Khóa hệ thống
            }
        }
    }
}
```

---

## 1.6. "UNDER THE HOOD" & CASE STUDY (THEO SÁCH ARM)
- **Under the Hood (1.3.1):** So sánh kiến trúc tập lệnh Thumb-2 của ARM Cortex-M4 so với kiến trúc RISC cổ điển; Phân tích ưu thế của bộ tăng tốc Flash ART Accelerator trên STM32F407 giúp thực thi lệnh với 0 chu kỳ chờ (0 wait-state) ngay cả ở tần số 168 MHz.
- **Case Study (1.4.1):** Phân tích hệ thống Khóa Cửa Thông Minh (Smart Door Locks) trong công nghiệp: Yêu cầu về cơ chế chống tấn công vét cạn mật mã, cơ chế chốt trạng thái và giải pháp cấp nguồn dự phòng khẩn cấp khi mất điện lưới.
