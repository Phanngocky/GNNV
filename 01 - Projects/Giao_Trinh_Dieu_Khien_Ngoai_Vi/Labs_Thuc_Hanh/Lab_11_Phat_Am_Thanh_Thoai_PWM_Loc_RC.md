# BÀI THỰC HÀNH SỐ 11: PHÁT ÂM THANH THOẠI CẢNH BÁO BẰNG KỸ THUẬT PWM TỐC ĐỘ CAO VÀ MẠCH LỌC TÍCH PHÂN RC
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết điều chế sóng mang $\rightarrow$ Mạch lọc tích phân RC $\rightarrow$ Lập trình nạp mẫu PCM qua Ngắt $\rightarrow$ Đo kiểm âm thanh)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Nguyên lý PWM Audio), CLO 2 (Tính tần số cắt bộ lọc RC), CLO 3 (Lập trình ngắt Timer nạp mẫu PCM), CLO 5 (Phân tích phổ Oscilloscope)  
**Căn cứ giáo trình:** *ARM Education - Chương 8: Audio Output using PWM (Ex 8.6)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Kỹ thuật Phát Âm thanh Kỹ thuật số:**
   - Phân tích nguyên lý tạo tín hiệu tương tự (DAC giả lập) bằng kỹ thuật **Điều chế độ rộng xung tốc độ cao (High-Frequency PWM)**: Biến đổi từng mẫu âm thanh số (PCM - Pulse Code Modulation 8-bit) thành độ rộng xung tương ứng.
   - Hiểu rõ vai trò của tần số sóng mang (Carrier Frequency $f_{carrier} = 100\text{ kHz}$) và tần số lấy mẫu thoại chuẩn ($f_s = 8\text{ kHz}$, chu kỳ $T_s = 125\mu s$).
   - Thiết kế mạch tích phân lọc thông thấp RC (Low-Pass Filter) bậc 1: Triệt tiêu hoàn toàn sóng mang cao tần $100\text{ kHz}$ và khôi phục mượt mà tín hiệu âm thanh thoại analog trong dải nghe được ($300\text{ Hz} \div 3.4\text{ kHz}$).
2. **Kỹ thuật Lập trình:**
   - Cấu hình khối Timer nâng cao `TIM1` (`PE9` - TIM1_CH1) chạy ở xung nhịp tối đa $168\text{ MHz}$ phát sóng mang PWM $100\text{ kHz}$ với độ phân giải 8-bit ($0 \div 255$).
   - Cấu hình Timer cơ bản `TIM6` tạo ngắt định thời chính xác mỗi $125\mu s$ ($8\text{ kHz}$) để nạp từng mẫu âm thanh từ mảng dữ liệu trong bộ nhớ Flash (`const uint8_t audioData[]`) vào thanh ghi so sánh `TIM1->CCR1`.
   - Phát câu thông báo thoại cảnh báo: *"Warning! Fire alarm activated, please evacuate!"* (Ex 8.6).
3. **Đo kiểm & Đánh giá Dạng sóng:**
   - Dùng Oscilloscope 2 kênh đối chiếu trực quan dạng sóng: Kênh 1 đo tại ngõ ra chân STM32 (xung vuông số $100\text{ kHz}$) và Kênh 2 đo sau mạch lọc RC (sóng analog âm thanh liên tục).

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Module / Linh kiện | Thông số kỹ thuật | Số lượng | Ghi chú |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 (Flash 1MB) | 1 | Kèm cáp nạp ST-Link |
| 2 | Loa mini hoặc Jack tai nghe 3.5mm | Trở kháng $8\Omega$ hoặc $32\Omega$ | 1 | Phát âm thanh nghe được |
| 3 | Tụ điện & Điện trở | Trở $1\text{ k}\Omega$; Tụ hóa $10\mu\text{F}$; Tụ gốm $100\text{ nF}$ (104) | Mỗi loại 2 | Mạch lọc RC và ngăn dòng DC |
| 4 | Mạch khuếch đại âm thanh mini | Module PAM8403 hoặc LM386 (tùy chọn) | 1 | Khuếch đại nếu dùng loa lớn |
| 5 | Thiết bị đo | Oscilloscope số 2 kênh | 1 | Đo dạng sóng trước và sau lọc |

---

## 3. CƠ SỞ TÍNH TOÁN VÀ THIẾT KẾ MẠCH NGUYÊN LÝ

### 3.1. Tính toán tần số sóng mang `TIM1` ($100\text{ kHz}$)
Khối `TIM1` nằm trên bus ngoại vi **APB2** với tần số xung nhịp $f_{PCLK2} = 84\text{ MHz}$. Vì bộ chia APB2 Prescaler = 2, bộ nhân tần của Timer tự động nhân 2:
$$f_{TIM1\_CLK} = 168\text{ MHz}.$$

Ta cần tần số sóng mang $f_{PWM} = 100\text{ kHz}$ với độ phân giải biên độ 8-bit ($ARR = 255 \implies ARR + 1 = 256$ mức):
$$f_{PWM} = \frac{f_{TIM1\_CLK}}{(PSC + 1) \times (ARR + 1)}$$
$$\implies PSC + 1 = \frac{168{,}000{,}000\text{ Hz}}{100{,}000\text{ Hz} \times 256} = \frac{1680}{256} \approx 6.5625$$
Để có giá trị nguyên chính xác, ta chọn:
- $\text{PSC} = 0 \implies PSC + 1 = 1$.
- Chu kỳ đếm:
  $$\text{ARR} + 1 = \frac{168{,}000{,}000}{100{,}000 \times 1} = 1680 \implies \text{ARR} = 1679.$$
- Để nạp mẫu âm thanh 8-bit ($0 \div 255$) vào thang đo $1680$, ta nhân tỷ lệ:
  $$\text{CCR1} = \text{sample} \times \frac{1680}{255} \approx \text{sample} \times 6.588.$$

---

### 3.2. Tính toán định thời ngắt lấy mẫu âm thanh `TIM6` ($8\text{ kHz}$)
Khối `TIM6` nằm trên bus **APB1** với xung nhịp Timer là $84\text{ MHz}$.  
Tần số lấy mẫu thoại tiêu chuẩn là $f_s = 8{,}000\text{ Hz}$ (Chu kỳ $T_s = 125\mu s$):
$$f_s = \frac{84{,}000{,}000}{(PSC + 1) \times (ARR + 1)} = 8{,}000\text{ Hz}$$
- Chọn $\text{PSC} = 83 \implies PSC + 1 = 84$ (Tần số đếm timer là $1\text{ MHz}$, mỗi tick $= 1\mu s$).
- Chu kỳ tự động nạp lại: $\text{ARR} + 1 = \frac{1{,}000{,}000}{8{,}000} = 125 \implies \text{ARR} = 124$.
- $\implies$ Cứ đúng $125\mu s$, Timer 6 sẽ kích hoạt ngắt phần cứng `TIM6_DAC_IRQHandler` một lần!

---

### 3.3. Mạch lọc thông thấp RC tích phân và tụ chặn DC
- Tín hiệu âm thanh thoại của con người tập trung chủ yếu dưới $3.4\text{ kHz}$.
- Sóng mang PWM nằm ở $100\text{ kHz}$.
- Chọn $R = 1\text{ k}\Omega$, $C = 100\text{ nF}$:
  $$f_c = \frac{1}{2\pi R C} = \frac{1}{2\pi \times 1000 \times 100 \times 10^{-9}} \approx 1.59\text{ kHz}.$$
  - Tại tần số âm thanh $1\text{ kHz}$: Mạch cho qua gần như toàn bộ (Độ suy hao nhỏ hơn $3\text{dB}$).
  - Tại tần số sóng mang $100\text{ kHz}$: Mạch suy hao hơn $36\text{dB}$ (sóng mang bị triệt tiêu phẳng lì thành điện áp một chiều trung bình).
- Tụ hóa $C_{block} = 10\mu\text{F}$ mắc nối tiếp với loa để chặn thành phần một chiều $V_{DC} = 1.65\text{V}$, ngăn cuộn dây loa bị nóng và méo âm.

```
       STM32F407
     +-----------+            R = 1k
     |  PE9(PWM) |------------/\/\/\/-----+-----------+
     |           |                        |           |
     |           |                      ===== C      [+] C_block = 10uF
     |           |                   100nF|           |
     |       GND |------------+-----------+-----------(---- Loa / Tai nghe (8-32 Ohm)
     +-----------+            |                       |
                             GND                     GND
```

---

## 4. QUY TRÌNH CẤU HÌNH TRÊN STM32CubeMX
1. **Cấu hình TIM1 (PWM sóng mang 100kHz):**
   - Vào *Timers* $\rightarrow$ *TIM1*.
   - Channel 1: **PWM Generation CH1** (Tương ứng chân `PE9`).
   - Prescaler: **0**.
   - Counter Period (ARR): **1679**.
   - PWM Mode: **PWM mode 1**.
2. **Cấu hình TIM6 (Ngắt định thời 8kHz):**
   - Vào *Timers* $\rightarrow$ *TIM6*.
   - Tích chọn **Activated**.
   - Prescaler: **83**.
   - Counter Period: **124**.
   - Vào tab *NVIC Settings* của TIM6: **Tích chọn TIM6 global interrupt**, Preemption Priority = **1**.

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### 5.1. Mảng dữ liệu âm thanh số PCM (Lưu trong Flash)
Tạo file `Core/Inc/audio_sample.h` chứa chuỗi mẫu âm thanh dạng sóng còi hú cảnh báo:

```c
#ifndef AUDIO_SAMPLE_H_
#define AUDIO_SAMPLE_H_

#include <stdint.h>

/* Bảng mẫu dạng sóng âm thanh PCM 8-bit (128 mức trung tính không tải) */
#define AUDIO_NUM_SAMPLES  8000 /* 8000 mẫu = đúng 1 giây âm thanh ở 8kHz */

extern const uint8_t soundWaveAlarm[AUDIO_NUM_SAMPLES];

#endif
```

*(Trong file `audio_sample.c`, tạo mảng `const uint8_t soundWaveAlarm[8000]` chứa mẫu sóng sin điều tần từ 500Hz đến 2kHz để tạo tiếng còi hú báo động).*

### 5.2. Trình phục vụ ngắt nạp mẫu âm thanh tốc độ cao
Trong file `Core/Src/main.c`:

```c
/* USER CODE BEGIN 0 */
#include "audio_sample.h"

extern TIM_HandleTypeDef htim1;
extern TIM_HandleTypeDef htim6;

static volatile uint32_t audioIndex = 0;
static volatile uint8_t isAudioPlaying = 0;

/* Bắt đầu phát câu thoại/còi báo động */
void Audio_PlayAlarm(void)
{
    audioIndex = 0;
    isAudioPlaying = 1;
    HAL_TIM_Base_Start_IT(&htim6); /* Bật ngắt 8kHz */
}

/* Dừng phát âm thanh */
void Audio_Stop(void)
{
    isAudioPlaying = 0;
    HAL_TIM_Base_Stop_IT(&htim6);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 0); /* Tắt xung */
}

/* Trình phục vụ ngắt định thời của TIM6 (Chu kỳ chính xác 125us) */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM6)
    {
        if (isAudioPlaying)
        {
            /* Đọc mẫu âm thanh PCM 8-bit (0 - 255) từ Flash */
            uint8_t sample = soundWaveAlarm[audioIndex++];

            /* Ánh xạ sang thang đo CCR1 của TIM1 (0 - 1679) */
            uint32_t ccrVal = (uint32_t)sample * 1679 / 255;
            __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, ccrVal);

            /* Nếu phát hết mảng âm thanh */
            if (audioIndex >= AUDIO_NUM_SAMPLES)
            {
                audioIndex = 0; /* Lặp lại tiếng còi hú hoặc dừng */
            }
        }
    }
}
/* USER CODE END 0 */
```

### 5.3. Vòng lặp chính kích hoạt phát âm thanh khi có cháy
```c
/* USER CODE BEGIN 2 */
/* Khởi động Timer 1 phát sóng mang PWM 100kHz */
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);

printf("HE THONG AM THANH PWM READY!\r\n");
printf("Nhan nut PE4 de phat tieng coi bao dong...\r\n");
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    /* Nếu nút nhấn kích hoạt PE4 được bấm */
    if (HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_4) == GPIO_PIN_SET)
    {
        printf("KICH HOAT COI HU AM THANH SO PWM!\r\n");
        Audio_PlayAlarm();
        HAL_Delay(5000); /* Hú còi trong 5 giây */
        Audio_Stop();
        printf("DA TAT COI HU.\r\n");
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & PHÂN TÍCH DẠNG SÓNG TRÊN OSCILLOSCOPE

### 6.1. Thiết lập máy hiện sóng
1. Kênh **CH1 (Vàng)**: Kẹp vào chân `PE9` (Tín hiệu PWM trước mạch lọc).
2. Kênh **CH2 (Xanh)**: Kẹp vào điểm nối giữa trở $R$ và tụ $C$ (Tín hiệu sau lọc RC).
3. Kẹp mass của cả 2 que đo vào `GND`.
4. Timebase: Đặt ở mức $500\mu s\text{/div}$ để quan sát dạng sóng âm thanh tần số thấp, hoặc chuyển sang $5\mu s\text{/div}$ để phóng to sóng mang PWM $100\text{ kHz}$.

### 6.2. Phân tích hiện tượng vật lý
1. **Ở thang đo $5\mu s\text{/div}$ (Phóng to sóng mang):**
   - Kênh CH1 là chùm xung vuông sắc nét biên độ $3.3\text{V}$, chu kỳ đúng $10\mu s$ ($100\text{ kHz}$). Độ rộng mức cao liên tục thay đổi theo giá trị mẫu âm thanh.
   - Kênh CH2 là một đường điện áp tương đối phẳng, bằng giá trị trung bình tích phân của xung vuông: $V_{analog} = V_{DD} \times \text{Duty}$.
2. **Ở thang đo $500\mu s\text{/div}$ (Dạng sóng âm thanh):**
   - Kênh CH1 dày đặc xung vuông do timebase nén lại.
   - Kênh CH2 vẽ nên một đường cong hình sin / âm thanh thoại trơn tru, hoàn toàn không còn vết răng cưa của sóng mang, chứng minh mạch lọc RC đã tích phân hoàn hảo tín hiệu số thành tín hiệu tương tự!

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Ảnh chụp màn hình Oscilloscope đồng thời 2 kênh CH1 và CH2 chứng minh hiệu quả tích phân của bộ lọc RC.
- Thu âm đoạn âm thanh thực tế phát ra từ loa khi bấm nút kích hoạt báo động.

### 7.2. Câu hỏi đánh giá
1. *Tại sao tần số sóng mang PWM bắt buộc phải cao hơn rất nhiều (ít nhất gấp 10 lần) so với tần số âm thanh cần phát? Nếu ta chọn tần số PWM chỉ là $4\text{ kHz}$ thì người nghe sẽ nghe thấy âm thanh gì?*
2. *Tại sao phải sử dụng ngắt `TIM6` định thời $125\mu s$ để nạp mẫu âm thanh mà không thể nạp trong vòng lặp `while(1)` bằng hàm `HAL_Delay()`?*
3. *Một mảng âm thanh PCM 8-bit dài 10 giây ở tần số lấy mẫu 8kHz chiếm bao nhiêu dung lượng bộ nhớ? Bộ nhớ Flash 1MB của STM32F407 có thể lưu trữ tối đa bao nhiêu giây âm thanh không nén loại này?*