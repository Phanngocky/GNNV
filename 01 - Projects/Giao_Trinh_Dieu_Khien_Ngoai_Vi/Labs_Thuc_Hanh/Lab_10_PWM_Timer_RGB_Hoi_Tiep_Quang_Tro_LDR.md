# BÀI THỰC HÀNH SỐ 10: KHỐI TIMER NÂNG CAO, BĂM XUNG PWM ĐA KÊNH VÀ VÒNG ĐIỀU KHIỂN HỒI TIẾP ÂM
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết Timer PWM $\rightarrow$ Tính toán thanh ghi $\rightarrow$ Lập trình Closed-Loop $\rightarrow$ Đo kiểm Oscilloscope)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Khối Timer & PWM), CLO 2 (Tính PSC, ARR, CCR), CLO 3 (Lập trình điều khiển hồi tiếp kín), CLO 5 (Bắt dạng sóng PWM)  
**Căn cứ giáo trình:** *ARM Education - Chương 8: Pulse Width Modulation and Negative Feedback (Ex 8.1 $\div$ 8.5)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Khối Timer & Điều chế Độ rộng Xung (PWM):**
   - Phân tích cấu trúc khối Timer đa năng trên STM32F407: Bộ đếm `CNT`, Bộ chia tần `PSC` (Prescaler), Thanh ghi tự động nạp lại `ARR` (Auto-Reload Register), và Thanh ghi so sánh bắt giữ `CCR` (Capture/Compare Register).
   - Tự tay tính toán cấu hình để tạo ra tần số xung PWM chính xác và hệ số công tác (Duty Cycle $D\%$).
   - Nắm vững cơ sở lý thuyết của **Hệ thống điều khiển kín có hồi tiếp âm (Closed-Loop Negative Feedback)**: Tín hiệu đặt (Set-point), Tín hiệu đo được (Process Variable), Tín hiệu sai lệch ($e = SP - PV$), và Tín hiệu điều khiển bù trừ.
2. **Kỹ thuật Lập trình:**
   - Cấu hình khối `TIM4` xuất 3 kênh PWM độc lập (`CH1` trên `PD12`, `CH2` trên `PD13`, `CH4` trên `PD15`) để pha trộn màu sắc LED RGB theo không gian màu HSV/RGB.
   - Ghép nối quang trở LDR (`PA4`) qua bộ chuyển đổi `ADC1_IN4` đo cường độ ánh sáng môi trường.
   - Xây dựng thuật toán điều khiển đèn thông minh tự động thích ứng: Khi phòng tối dần, hệ thống tự động tăng Duty Cycle của LED để bù sáng; khi trời sáng, tự động giảm độ sáng LED nhằm duy trì độ rọi quang học ổn định (Ex 8.1 $\div$ 8.5).
3. **Đo kiểm & Phân tích:**
   - Dùng máy hiện sóng quan sát biến thiên độ rộng xung PWM của kênh `TIM4_CH1` theo thời gian thực khi che tối hoặc rọi đèn vào quang trở LDR.

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Kèm cáp nạp ST-Link |
| 2 | LED RGB 4 chân loại Anode chung / Cathode chung | Đỏ (R), Lục (G), Lam (B) 5mm | 1 | Mắc trở hạn dòng cho từng kênh |
| 3 | Quang trở LDR (Light Dependent Resistor) | Đường kính 5mm, $R_{dark} > 1\text{ M}\Omega, R_{light} \approx 5 \div 10\text{ k}\Omega$ | 1 | Cảm biến ánh sáng |
| 4 | Điện trở kim loại 0.25W | $330\Omega$ (hạn dòng LED RGB), $10\text{ k}\Omega$ (phân áp LDR) | Mỗi loại 3 | Lắp mạch Breadboard |
| 5 | Thiết bị đo | Oscilloscope số 2CH | 1 | Đo tần số và Duty Cycle |

---

## 3. CƠ SỞ TÍNH TOÁN VÀ THIẾT KẾ MẠCH NGUYÊN LÝ

### 3.1. Tính toán cấu hình thanh ghi Timer PWM
Khối `TIM4` nằm trên bus ngoại vi **APB1**. Khi bộ chia APB1 Prescaler khác 1 (ở đây APB1 = 42MHz), bộ nhân xung nhịp của Timer tự động nhân đôi:
$$f_{TIM4\_CLK} = 2 \times f_{APB1} = 2 \times 42\text{ MHz} = 84\text{ MHz}.$$

Ta cần tạo ra tần số xung PWM là $f_{PWM} = 1\text{ kHz}$ ($1000\text{ Hz}$) với độ phân giải Duty Cycle là $1000$ bước ($0.1\%$ mỗi bước):
- Chọn giá trị thanh ghi tự động nạp lại: $\text{ARR} = 999$ (tương ứng $ARR + 1 = 1000$ đếm).
- Công thức tính tần số PWM:
  $$f_{PWM} = \frac{f_{TIM4\_CLK}}{(PSC + 1) \times (ARR + 1)}$$
  $$\implies PSC + 1 = \frac{f_{TIM4\_CLK}}{f_{PWM} \times (ARR + 1)} = \frac{84{,}000{,}000}{1000 \times 1000} = 84 \implies \text{PSC} = 83.$$
- **Độ rộng xung (Duty Cycle $D\%$):**
  $$D = \frac{\text{CCR}}{\text{ARR} + 1} \times 100\% = \frac{\text{CCR}}{1000} \times 100\% \implies \text{CCR} = 0 \div 1000.$$

---

### 3.2. Sơ đồ mạch phân áp quang trở LDR
Mắc mạch cầu phân áp giữa quang trở LDR và điện trở chuẩn $R_1 = 10\text{ k}\Omega$:
- Chân 1 của LDR nối lên nguồn $3.3\text{V}$.
- Chân 2 của LDR nối vào chân ngõ vào `PA4` của STM32 và nối tiếp với trở $R_1 = 10\text{ k}\Omega$ xuống mass ($GND$).
- **Điện áp đưa vào chân `PA4`:**
  $$V_{ADC} = 3.3\text{V} \times \frac{R_1}{R_{LDR} + R_1}$$
  - Khi trời sáng: $R_{LDR}$ giảm nhỏ ($\approx 1\text{ k}\Omega$) $\implies V_{ADC}$ tăng cao ($\approx 3.0\text{V}$).
  - Khi trời tối: $R_{LDR}$ tăng rất lớn ($> 100\text{ k}\Omega$) $\implies V_{ADC}$ giảm thấp ($\approx 0.3\text{V}$).

---

### 3.3. Bảng phân bổ chân kết nối
| Chân STM32F407 | Ngoại vi / Linh kiện | Chức năng phần cứng | Cấu hình CubeMX |
| :---: | :---: | :--- | :--- |
| `PD12` | Kênh R (LED RGB) | TIM4_CH1 (PWM) | Alternate Function TIM4_CH1 |
| `PD13` | Kênh G (LED RGB) | TIM4_CH2 (PWM) | Alternate Function TIM4_CH2 |
| `PD15` | Kênh B (LED RGB) | TIM4_CH4 (PWM) | Alternate Function TIM4_CH4 |
| `PA4` | Cầu phân áp LDR | ADC1_IN4 | Analog Mode ADC1_IN4 |
| `GND` | Cathode chung LED RGB & Mass LDR | Mass chung | Mass hệ thống |

---

## 4. QUY TRÌNH CẤU HÌNH TRÊN STM32CubeMX
1. **Khối TIM4 (Timer 4 PWM):**
   - Vào mục *Timers* $\rightarrow$ *TIM4*.
   - Clock Source: **Internal Clock**.
   - Channel 1: **PWM Generation CH1**.
   - Channel 2: **PWM Generation CH2**.
   - Channel 4: **PWM Generation CH4**.
   - Thẻ *Parameter Settings*:
     - Prescaler (PSC): **83**.
     - Counter Period (ARR): **999**.
     - Counter Mode: **Up**.
     - PWM Mode: **PWM mode 1**.
     - Pulse (CCR ban đầu): **0**.
     - CH Polarity: **High**.
2. **Khối ADC1 (Đọc LDR):**
   - Chọn *Analog* $\rightarrow$ *ADC1* $\rightarrow$ Tích chọn **IN4** (`PA4`).
   - Resolution: **12-bit**.
   - Sampling Time: **56 Cycles**.

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### 5.1. Khởi động các kênh Timer PWM và Macro điều khiển
Thêm vào `Core/Src/main.c`:

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>

extern TIM_HandleTypeDef htim4;
extern ADC_HandleTypeDef hadc1;

/* Đặt giá trị độ rộng xung CCR cho từng kênh màu */
void RGB_SetColor(uint16_t red, uint16_t green, uint16_t blue)
{
    __HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_1, red);   /* PD12 Red */
    __HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_2, green); /* PD13 Green */
    __HAL_TIM_SET_COMPARE(&htim4, TIM_CHANNEL_4, blue);  /* PD15 Blue */
}

/* Đọc giá trị ADC1 từ chân PA4 (LDR) */
uint32_t LDR_ReadRaw(void)
{
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 10);
    uint32_t val = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);
    return val;
}
/* USER CODE END 0 */
```

### 5.2. Vòng điều khiển hồi tiếp âm tự động ổn định ánh sáng (Ex 8.4, 8.5)
```c
/* USER CODE BEGIN 2 */
/* Khởi động tạo xung PWM trên cả 3 kênh phần cứng của TIM4 */
HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_1);
HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_2);
HAL_TIM_PWM_Start(&htim4, TIM_CHANNEL_4);

/* Mục tiêu ánh sáng mong muốn (Set-point: mức ADC tương ứng ánh sáng phòng chuẩn) */
const int32_t LIGHT_SETPOINT = 2000; 
float Kp = 0.5f; /* Hệ số khuếch đại tỷ lệ P */
int32_t currentDuty = 500;
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    static uint32_t lastControlTime = 0;
    if (HAL_GetTick() - lastControlTime >= 50) /* Chu kỳ điều khiển 20Hz (50ms) */
    {
        lastControlTime = HAL_GetTick();

        /* 1. Đọc giá trị phản hồi hiện tại (Process Variable) */
        int32_t currentLight = (int32_t)LDR_ReadRaw();

        /* 2. Tính toán sai lệch tín hiệu (Error) */
        /* Nếu ánh sáng môi trường < SETPOINT (Trời tối): error > 0 -> Cần tăng đèn */
        int32_t error = LIGHT_SETPOINT - currentLight;

        /* 3. Thuật toán bù trừ hồi tiếp tỷ lệ (Proportional Closed-Loop) */
        currentDuty += (int32_t)(Kp * (float)error);

        /* 4. Giới hạn bão hòa xung PWM (Anti-windup Saturation: 0 - 999) */
        if (currentDuty > 999) currentDuty = 999;
        if (currentDuty < 0)   currentDuty = 0;

        /* 5. Cập nhật Duty Cycle ra LED để bù sáng tự động */
        RGB_SetColor((uint16_t)currentDuty, (uint16_t)currentDuty, (uint16_t)currentDuty);
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM DẠNG SÓNG PWM TRÊN MÁY HIỆN SÓNG

### 6.1. Thiết lập máy hiện sóng (Oscilloscope)
1. Kẹp que đo `CH1` vào chân `PD12` (PWM Red), kẹp mass vào `GND`.
2. Timebase: $200\mu s\text{/div}$, V/div: $1\text{ V/div}$.
3. Bật tính năng đo tự động (Measure): **Frequency** và **+Duty Cycle**.

### 6.2. Phân tích kết quả
1. **Kiểm tra tần số:**
   - Màn hình Oscilloscope đo được chính xác $f = 1.000\text{ kHz}$ ($\pm 0.1\%$).
   - Chu kỳ toàn phần của 1 xung: $T = \frac{1}{1000\text{ Hz}} = 1.0\text{ ms}$ (đúng 5 ô ngang timebase).
2. **Kiểm tra vòng hồi tiếp kín:**
   - Dùng tay che dần quang trở LDR (môi trường tối lại): Độ rộng mức cao $t_{on}$ trên Oscilloscope tự động giãn rộng ra từ $100\mu s$ lên tới $900\mu s$ (Duty Cycle tăng từ $10\%$ lên $90\%$).
   - Rọi đèn flash điện thoại vào LDR: Độ rộng xung mức cao $t_{on}$ tự động co hẹp về gần $0\mu s$ (Duty Cycle giảm về $0\%$).

```
                      +Duty = 25% (Trời sáng)
         +---+               +---+
   3.3V: |   |               |   |
     0V: +   +---------------+   +---------------+
         |<---- 1.0 ms ----->|

                      +Duty = 75% (Trời tối)
         +-----------+       +-----------+
   3.3V: |           |       |           |
     0V: +           +-------+           +-------+
```

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Ảnh chụp dạng sóng Oscilloscope ở 3 mức: Khi rọi sáng (Duty thấp), bình thường, và khi che tối (Duty cao).
- Bảng tính giá trị `PSC` và `ARR` nếu cần tạo ra tần số PWM công nghiệp $25\text{ kHz}$ để điều khiển quạt làm mát không phát ra tiếng rít tai người nghe được.

### 7.2. Câu hỏi đánh giá
1. *Tại sao tần số PWM điều khiển LED mắt người nhìn thấy chỉ cần $> 100\text{ Hz}$, nhưng khi điều khiển động cơ hoặc cuộn cảm người ta thường dùng tần số trên $20\text{ kHz}$?*
2. *Nếu ta đổi cực tính chân thành Active-LOW (`TIM_OCPOLARITY_LOW`), hình dạng xung và hệ số công tác sẽ đảo ngược như thế nào?*
3. *Hiện tượng "sai số xác lập" (Steady-State Error) trong bộ điều khiển chỉ có khâu tỷ lệ P xuất hiện như thế nào khi ánh sáng môi trường thay đổi đột ngột? Cần bổ sung thêm khâu tích phân (I) hay vi phân (D) để triệt tiêu sai số này?*