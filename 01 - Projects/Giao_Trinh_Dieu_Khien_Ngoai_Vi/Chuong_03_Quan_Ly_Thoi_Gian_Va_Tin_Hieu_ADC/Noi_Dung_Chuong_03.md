# CHƯƠNG 3: QUẢN LÝ THỜI GIAN VÀ ĐO LƯỜNG TÍN HIỆU TƯƠNG TỰ (ADC) TRÊN STM32F407
*(Biên soạn dựa trên Chapter 3: Time Management and Analog Signals - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 3.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Sinh viên đã thành thạo thao tác với các tín hiệu số logic 0/1 ở Chương 1; chương này mở rộng sang thế giới tín hiệu liên tục trong thực tế thông qua **Bộ chuyển đổi Tương tự - Số (SAR ADC 12-bit)** của vi điều khiển STM32F407, kết hợp với phương pháp luận **Quản lý thời gian không chặn (Non-blocking Timing)** bằng bộ đếm thời gian hệ thống SysTick.
- **Mục tiêu chi tiết:**
  1. **Lý thuyết Bộ Chuyển Đổi SAR ADC:** Nắm vững nguyên lý xấp xỉ liên tiếp (Successive Approximation Register), điện áp tham chiếu $V_{REF+} = 3.3\text{V}$, bước lượng tử hóa $q = \text{LSB} \approx 0.80566\text{ mV}$, sai số lượng tử hóa $\pm \frac{1}{2}\text{LSB}$, và thời gian lấy mẫu nạp tụ (Sampling Time).
  2. **Thiết Kế Mạch Lọc & Đệm Tín Hiệu Ngoại Vi:**
     - Thiết kế mạch lọc thông thấp RC (Anti-aliasing Filter) triệt tiêu nhiễu cao tần trước khi đưa vào chân ADC của vi điều khiển.
     - Thiết kế mạch đệm Transistor NPN và diode dập xung để điều khiển còi báo động Piezo Buzzer (`PD11`) công suất lớn.
  3. **Thuật Toán Lọc Số Xử Lý Tín Hiệu (DSP):** Xây dựng bộ lọc trung bình trượt (Moving Average Filter) 8 mẫu trên bộ đệm vòng (Ring Buffer) để loại bỏ hiện tượng giật số liệu cảm biến.
  4. **Tư Duy Nhúng Thời Gian Thực (Real-Time Scheduling):** Xóa bỏ hoàn toàn hàm trễ `HAL_Delay()` gây đóng băng CPU; hiện thực hóa siêu vòng lặp đa nhiệm tuần hoàn dựa trên hàm `HAL_GetTick()`.
  5. **Lập Trình Hệ Thống (Examples 3.1 $\div$ 3.5):**
     - Đọc biến trở đặt ngưỡng an toàn linh hoạt (`PA1`).
     - Đo nhiệt độ chính xác bằng cảm biến tương tự LM35DZ (`PA2`) và phát hiện khí gas MQ-2 (`PA3`).
     - Điều khiển còi Piezo Buzzer phát tín hiệu bíp ngắt quãng với tần số thay đổi theo mức độ nguy cấp.

---

## 3.2. CƠ SỞ LÝ THUYẾT CHUYÊN SÂU

### 3.2.1. Kiến Trúc Bộ Chuyển Đổi SAR ADC 12-bit Trên STM32F407
Khối ADC trên STM32F407 hoạt động theo nguyên lý **Xấp xỉ liên tiếp (Successive Approximation Register - SAR)**. Khối này bao gồm một bộ so sánh điện áp cực nhạy (Comparator), một bộ chuyển đổi số - tương tự nội (Internal DAC) và thanh ghi xấp xỉ liên tiếp:

```
                      V_REF+ (3.3V)
                         |
                         v
Tín hiệu Analog        +----+
V_in ----------------->| +  |
(PA1, PA2, PA3)        |    |---> [ Bộ logic điều khiển SAR ] ---> Thanh ghi kết quả
                  +--->| -  |                 |                    (ADC_DR: 12-bit)
                  |    +----+                 |
                  |                           | Cập nhật thử từng bit (MSB -> LSB)
                  +--- [ Internal 12-bit DAC ]<--+
```

#### Quá trình chuyển đổi từng bit (Binary Search):
1. **Lấy mẫu (Sampling Phase):** Công tắc ngõ vào đóng lại trong một khoảng thời gian $T_{sampling}$ để nạp đầy tụ điện lấy mẫu nội ($C_{sample} \approx 4\text{ pF}$) đạt tới điện áp $V_{in}$.
2. **Chuyển đổi (Conversion Phase):** Cần đúng **12 chu kỳ xung nhịp ADC** để xác định 12 bit từ MSB (Bit 11) đến LSB (Bit 0):
   - Bước 1: Đặt thử bit MSB = 1 (tương ứng $V_{DAC} = \frac{V_{REF}}{2} = 1.65\text{V}$). So sánh với $V_{in}$. Nếu $V_{in} \ge 1.65\text{V}$, giữ MSB = 1; nếu $V_{in} < 1.65\text{V}$, xóa MSB = 0.
   - Bước 2 $\div$ 12: Lặp lại quá trình so sánh cho các bit kế tiếp với trọng số $\frac{V_{REF}}{4}, \frac{V_{REF}}{8}, \dots, \frac{V_{REF}}{4096}$.
3. **Tổng thời gian chuyển đổi:**
   $$T_{conv} = T_{sampling} + 12 \text{ chu kỳ ADC}$$

#### Công thức tính toán điện áp và bước lượng tử hóa (LSB):
- Độ phân giải 12-bit tương ứng $2^{12} = 4096$ mức giá trị số ($0 \div 4095$).
- Bước lượng tử hóa nhỏ nhất (1 LSB):
  $$q = \text{LSB} = \frac{V_{REF+}}{2^N - 1} = \frac{3.3\text{V}}{4095} \approx 0.80566\text{ mV}$$
- Điện áp ngõ vào thực tế tính từ giá trị số đọc được:
  $$V_{in} = \frac{\text{ADC\_Value}}{4095} \times V_{REF+} = \frac{\text{ADC\_Value}}{4095} \times 3.3\text{V}$$

---

### 3.2.2. Đặc Tính Cảm Biến Nhiệt Độ Tương Tự LM35DZ
Cảm biến LM35DZ là vi mạch cảm biến nhiệt độ bán dẫn chính xác cao:
- Điện áp ngõ ra tỷ lệ thuận tuyến tính tuyệt đối với nhiệt độ Celsius:
  $$\text{Độ dốc (Scale Factor)} = 10\text{ mV}/^\circ\text{C} = 0.01\text{ V}/^\circ\text{C}$$
- Điện áp ngõ ra tại $0^\circ\text{C}$ là $0\text{V}$; tại $25^\circ\text{C}$ là $250\text{mV}$; tại $100^\circ\text{C}$ là $1.0\text{V}$.
- Công thức tính trực tiếp nhiệt độ Celsius từ giá trị thô ADC:
  $$T(^\circ\text{C}) = \frac{V_{in}}{0.01\text{V}/^\circ\text{C}} = \frac{\text{ADC\_Value} \times 3.3\text{V}}{4095 \times 0.01\text{V}} = \text{ADC\_Value} \times \frac{330}{4095} \approx \text{ADC\_Value} \times 0.080586$$

---

### 3.2.3. Mạch Lọc Thông Thấp RC Chống Răng Cưa (Anti-Aliasing Filter)
Trong môi trường công nghiệp, dây dẫn nối từ cảm biến LM35 về chân vi điều khiển đóng vai trò như một ăng-ten bắt các sóng nhiễu cao tần (tần số $50\text{Hz}$ điện lưới, xung clock vi điều khiển, sóng cao tần Wi-Fi/GSM).  
Nếu tần số nhiễu này lớn hơn một nửa tần số lấy mẫu ($f_{noise} > \frac{f_s}{2}$), hiện tượng **méo chồng phổ (Aliasing)** sẽ biến nhiễu cao tần thành sóng gợn tần số thấp không thể lọc bỏ được bằng phần mềm.

```
       Chân Vout LM35            R = 1k Ohm            PA2 (ADC1_IN2)
       ------------------------/\/\/\/\/----+---------> (STM32F407)
                                            |
                                          ===== C = 100nF
                                            |
       GND ---------------------------------+---------> GND
```
- Tần số cắt của mạch lọc thông thấp RC:
  $$f_c = \frac{1}{2\pi R C} = \frac{1}{2\pi \times 1000 \times 100 \times 10^{-9}} \approx 1{,}591\text{ Hz} \approx 1.6\text{ kHz}$$
- Tín hiệu nhiệt độ biến thiên rất chậm ($< 1\text{ Hz}$) đi qua mạch lọc nguyên vẹn không suy hao, trong khi toàn bộ các gai nhiễu cao tần ($> 10\text{ kHz}$) bị ngắn mạch xuống mass qua tụ điện $100\text{nF}$.

---

### 3.2.4. Thuật Toán Lọc Số Trung Bình Trượt (Moving Average Filter)
Sau khi lọc phần cứng, giá trị đọc ADC vẫn có thể dao động nhẹ $\pm 2 \div 3\text{ LSB}$ do nhiễu nhiệt lượng tử hóa. Ta áp dụng thuật toán lọc số trung bình trượt bậc $N = 8$:
$$y[n] = \frac{1}{N} \sum_{i=0}^{N-1} x[n-i]$$
- Sử dụng một bộ đệm vòng (Ring Buffer) kích thước 8 phần tử. Mỗi khi có mẫu đo mới $x[n]$, mẫu này thay thế vị trí của mẫu cũ nhất $x[n-8]$ trong bộ đệm.
- Thuật toán cộng dồn tối ưu O(1):
  $$\text{RunningSum} = \text{RunningSum} - x[n-8] + x[n]$$
  $$y[n] = \frac{\text{RunningSum}}{8} = \text{RunningSum} \gg 3$$
- Phép dịch phải 3 bit (`>> 3`) thay thế phép chia giúp CPU Cortex-M4 thực thi bộ lọc chỉ trong đúng **1 chu kỳ xung nhịp**!

---

### 3.2.5. Tư Duy Nhúng: Quản Lý Thời Gian Không Chặn (Non-Blocking Timing)
So sánh giữa hai phương pháp lập trình điều phối thời gian:

| Tiêu chí so sánh | Dùng Hàm Trễ Chặn (`HAL_Delay`) | Dùng Thời Gian Không Chặn (`HAL_GetTick`) |
| :--- | :--- | :--- |
| **Cơ chế hoạt động** | CPU chạy vòng lặp rỗng `while` đốt chu kỳ clock vô ích. | CPU chỉ kiểm tra hiệu số mốc thời gian: `if (now - last >= INTERVAL)`. |
| **Tính đa nhiệm** | **Đơn nhiệm hoàn toàn:** Khi đang delay, không thể quét phím, không thể nhận UART, không thể bắt khí gas. | **Đa nhiệm cộng tác (Cooperative):** Hàng chục tác vụ cùng chia sẻ CPU chạy song song mượt mà. |
| **Thời gian đáp ứng** | Chậm hàng giây (bỏ sót các sự kiện nguy hiểm tức thời). | **Tức thời (Micro-giây):** Vòng lặp chính quay hàng trăm nghìn lần mỗi giây. |
| **Ứng dụng công nghiệp** | Nghiêm cấm trong hệ thống thời gian thực. | **Chuẩn mực bắt buộc trong lập trình nhúng chuyên nghiệp.** |

---

## 3.3. SƠ ĐỒ NGUYÊN LÝ & SƠ ĐỒ ĐẤU NỐI CHÂN CHI TIẾT

### 3.3.1. Sơ đồ mạch nguyên lý điện tử (Schematics)

```
=============================================================================================
1. MẠCH ĐỆM TRANSISTOR NPN ĐIỀU KHIỂN CÒI BÁO ĐỘNG PIEZO BUZZER (PD11)
=============================================================================================
                          +5V / +3.3V
                               |
                               +---------------+
                               |               |
                              --- Diode       [+] Còi Báo Động
                         1N4148^  Dập Xung   [BUZZER] Piezo Buzzer 5V
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


=============================================================================================
2. MẠCH BIẾN TRỞ ĐẶT NGƯỠNG AN TOÀN VÀ CẢM BIẾN NHIỆT ĐỘ LM35 (PA1 & PA2)
=============================================================================================
       +3.3V                           +3.3V
         |                               |
       +---+                           +----+
       |   | Biến trở xoay             |LM35| Cảm biến nhiệt độ LM35
       |   | 10k Ohm                   +----+
       +---+                             | Vout
         | Chân giữa                     |
         +--------------------> PA1      +----/\/\/\/\/-----+---------> PA2
         |                      (ADC1_1)      R = 1k Ohm    |           (ADC1_2)
        GND                                               ===== C = 100nF
                                                            |
                                                           GND
```

---

### 3.3.2. Bảng Phân Bổ Chân Toàn Diện Trên STM32F407 (Master Pinout Table)

| Tên Chân STM32 | Tên Tín Hiệu | Chức Năng Phần Cứng / Kênh ADC | Cấu Hình Trên STM32CubeMX | Thiết Bị Ngoại Vi Kết Nối | Mức Điện Áp An Toàn |
| :---: | :---: | :---: | :---: | :--- | :--- |
| `PA1` | `POT_THRESH`| `ADC1_IN1` (Analog) | Analog Mode (Rank 1, Sampling Time 56 Cycles) | Chân giữa biến trở chiết áp $10\text{k}\Omega$ | $0 \div 3.3\text{V}$, chỉnh ngưỡng $20 \div 60^\circ\text{C}$ |
| `PA2` | `LM35_TEMP` | `ADC1_IN2` (Analog) | Analog Mode (Rank 2, Sampling Time 56 Cycles) | Chân Vout cảm biến LM35 sau mạch lọc RC | $0 \div 1.5\text{V}$ ($10\text{mV}/^\circ\text{C}$) |
| `PA3` | `MQ2_GAS` | `ADC1_IN3` (Analog) | Analog Mode (Rank 3, Sampling Time 56 Cycles) | Chân Aout cảm biến nồng độ khí gas MQ-2 | $0 \div 3.3\text{V}$ |
| `PD11` | `BUZZER_OUT`| GPIO_Output | Output Push-Pull, High Speed | Cực Base transistor 2N2222 lái còi Buzzer | Mức logic $3.3\text{V}$ qua trở hạn dòng $1\text{k}\Omega$ |
| `PD12` | `ALARM_LED` | GPIO_Output | Output Push-Pull | Đèn LED đỏ báo quá nhiệt ngưỡng | Mức logic $3.3\text{V}$ qua trở hạn dòng $330\Omega$ |
| `3V3` | `VDD` | Nguồn cấp $3.3\text{V}$ | Nguồn nuôi hệ thống | Cấp cho chân 1 biến trở và chân VCC LM35 | Nguồn nuôi sạch |
| `GND` | `GND` | Mass hệ thống | Mass chung | Nối chung mass STM32, Biến trở, LM35 và Buzzer | Bắt buộc chung mass |

---

## 3.4. HIỆN THỰC HÓA CHI TIẾT CÁC VÍ DỤ TỪ EXAMPLE 3.1 ĐẾN 3.5 TRÊN STM32F407

### 3.4.1. Cấu hình STM32CubeMX
1. **Khối ADC1:**
   - Chọn *Analog* $\rightarrow$ *ADC1*.
   - Kích hoạt các kênh: **IN1** (`PA1`), **IN2** (`PA2`), **IN3** (`PA3`).
   - Clock Prescaler: **PCLK2 divided by 4** ($84\text{ MHz} / 4 = 21\text{ MHz} \le 36\text{ MHz}$).
   - Resolution: **12 bits (15 ADC clock cycles)**.
   - Scan Conversion Mode: **Disabled** (hoặc đọc độc lập từng kênh bằng cấu hình lại Rank).
   - Sampling Time: **56 Cycles** cho tất cả các kênh để đảm bảo tụ nạp điện áp chính xác.
2. **Khối GPIO:**
   - Chân `PD11, PD12`: GPIO_Output Push-Pull.

---

### 3.4.2. Mã Nguồn Hoàn Chỉnh Của Chương

#### 1. Hàm Đọc Kênh ADC Độc Lập & Bộ Lọc Số Trung Bình Trượt
```c
#include "main.h"
#include <stdio.h>
#include <stdbool.h>

extern ADC_HandleTypeDef hadc1;
extern UART_HandleTypeDef huart2;

/* Đọc giá trị số 12-bit từ một kênh ADC chỉ định */
uint16_t ADC_ReadChannel(uint32_t channel)
{
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel = channel;
    sConfig.Rank = 1;
    sConfig.SamplingTime = ADC_SAMPLETIME_56CYCLES;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);

    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 10);
    uint16_t raw = (uint16_t)HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);

    return raw;
}

/* Bộ lọc trung bình trượt 8 mẫu cho nhiệt độ */
#define FILTER_DEPTH 8
float Filter_Temperature(float newSample)
{
    static float window[FILTER_DEPTH] = {0};
    static uint8_t head = 0;
    static float sum = 0.0f;

    sum -= window[head];
    window[head] = newSample;
    sum += newSample;
    head = (head + 1) % FILTER_DEPTH;

    return (sum / FILTER_DEPTH);
}
```

---

#### 2. Vòng Điều Khiển Non-Blocking & Báo Động Biến Thiên Chu Kỳ (Examples 3.3 $\div$ 3.5)
```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_ADC1_Init();
    MX_USART2_UART_Init();

    uint32_t lastAdcTime = 0;
    uint32_t lastBuzzerToggleTime = 0;
    uint32_t lastPrintTime = 0;

    float currentTempC = 0.0f;
    float thresholdTempC = 30.0f;
    uint32_t buzzerIntervalMs = 0; /* 0: Tắt còi, >0: Chu kỳ bíp */
    bool buzzerState = false;

    printf("SMART HOME TEMPERATURE MONITOR READY!\r\n");

    while (1)
    {
        uint32_t currentMillis = HAL_GetTick();

        /* TÁC VỤ 1: Đọc ADC và Lọc số mỗi 50ms (Tần số lấy mẫu 20Hz) */
        if (currentMillis - lastAdcTime >= 50)
        {
            lastAdcTime = currentMillis;

            /* 1. Đọc biến trở PA1 đặt ngưỡng (0 - 4095 ánh xạ sang dải 20.0 - 60.0 độ C) */
            uint16_t potRaw = ADC_ReadChannel(ADC_CHANNEL_1);
            thresholdTempC = 20.0f + ((float)potRaw / 4095.0f) * 40.0f;

            /* 2. Đọc cảm biến nhiệt độ LM35 PA2 */
            uint16_t lm35Raw = ADC_ReadChannel(ADC_CHANNEL_2);
            float rawVoltageMv = ((float)lm35Raw * 3300.0f) / 4095.0f;
            float rawTempC = rawVoltageMv / 10.0f; /* 10mV / độ C */

            /* 3. Lọc số trung bình trượt */
            currentTempC = Filter_Temperature(rawTempC);

            /* 4. Đánh giá điều kiện cảnh báo */
            if (currentTempC >= thresholdTempC)
            {
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET); /* Bật LED đỏ */
                float delta = currentTempC - thresholdTempC;

                /* Càng vượt ngưỡng nhiều, còi kêu càng dồn dập */
                if (delta > 5.0f) {
                    buzzerIntervalMs = 100; /* Quá nhiệt nặng: Bíp dồn dập 100ms */
                } else {
                    buzzerIntervalMs = 400; /* Quá nhiệt nhẹ: Bíp cảnh báo 400ms */
                }
            }
            else
            {
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET); /* Tắt LED đỏ */
                buzzerIntervalMs = 0; /* Tắt còi */
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_11, GPIO_PIN_RESET);
                buzzerState = false;
            }
        }

        /* TÁC VỤ 2: Điều khiển còi Piezo Buzzer nhấp nháy Non-blocking */
        if (buzzerIntervalMs > 0)
        {
            if (currentMillis - lastBuzzerToggleTime >= buzzerIntervalMs)
            {
                lastBuzzerToggleTime = currentMillis;
                buzzerState = !buzzerState;
                HAL_GPIO_WritePin(GPIOD, GPIO_PIN_11, buzzerState ? GPIO_PIN_SET : GPIO_PIN_RESET);
            }
        }

        /* TÁC VỤ 3: Gửi báo cáo định kỳ lên máy tính mỗi 1000ms */
        if (currentMillis - lastPrintTime >= 1000)
        {
            lastPrintTime = currentMillis;
            printf("[GIAM SAT] Nhiet do: %.2f oC | Nguong dat: %.2f oC | Trang thai: %s\r\n",
                   currentTempC, thresholdTempC,
                   (buzzerIntervalMs > 0) ? "QUA NHIET (ALARM ON)" : "AN TOAN (NORMAL)");
        }
    }
}
```

---

## 3.5. "UNDER THE HOOD" & CASE STUDY (THEO GIÁO TRÌNH ARM)

### 3.5.1. Under the Hood (3.3.1): Trôi Nhiệt Điện Áp Tham Chiếu $V_{REF}$ & Bù Trừ Sai Số
- **Vấn đề:** Nếu điện áp nguồn $V_{DD}$ (đồng thời là $V_{REF+}$) bị dao động (ví dụ sụt áp từ $3.3\text{V}$ xuống $3.1\text{V}$ khi bật loa/motor), kết quả tính toán nhiệt độ sẽ bị sai lệch nghiêm trọng:
  $$V_{in} = \frac{\text{ADC\_Val}}{4095} \times 3.1\text{V} \ne \frac{\text{ADC\_Val}}{4095} \times 3.3\text{V}$$
- **Giải pháp phần cứng STM32F4:**
  - STM32F407 tích hợp sẵn kênh điện áp tham chiếu nội **$V_{REFINT} = 1.21\text{V}$** ổn định tuyệt đối trước nhiệt độ và nguồn cấp.
  - Bằng cách đọc kênh $V_{REFINT}$, phần mềm có thể tính ngược lại điện áp nguồn thực tế hiện thời $V_{DDA\_actual}$:
    $$V_{DDA\_actual} = 3.3\text{V} \times \frac{\text{VREFINT\_CAL}}{\text{ADC\_VREFINT}}$$
    từ đó triệt tiêu hoàn toàn sai số trôi nhiệt của bộ chuyển đổi ADC!

### 3.5.2. Case Study (3.4.1): Hệ Thống Chống Đóng Băng Cây Trồng (Frost Prevention System)
- **Bối cảnh ứng dụng:** Trong các nông trường nho và cây ăn quả ôn đới, nhiệt độ đêm xuống dưới $0^\circ\text{C}$ sẽ làm nước bên trong tế bào cây đóng băng, phá hủy màng tế bào và làm hỏng toàn bộ vụ mùa trị giá hàng triệu đô la.
- **Giải pháp nhúng trên ARM:**
  - Hệ thống đặt các cảm biến nhiệt độ phân tán khắp trang trại, nối về bộ điều khiển trung tâm.
  - Biến trở xoay chuẩn hóa cho phép nông dân đặt ngưỡng kích hoạt nhạy bén ($+1.5^\circ\text{C}$).
  - Khi nhiệt độ tụt xuống dưới ngưỡng báo động, vi điều khiển kích hoạt hệ thống quạt gió khổng lồ xáo trộn tầng nghịch nhiệt khí quyển và bật các lò sưởi khí hóa lỏng tự động để bảo vệ cây trồng kịp thời.

---

## 3.6. CÂU HỎI ÔN TẬP VÀ BÀI TẬP THIẾT KẾ CHƯƠNG 3

1. **Câu hỏi 1 (Lượng tử hóa ADC):** *Hãy tính toán giá trị số nguyên đọc được từ thanh ghi `ADC1->DR` nếu điện áp ngõ vào tại chân `PA2` là $0.75\text{V}$ (tương ứng $75^\circ\text{C}$ của LM35) với $V_{REF+} = 3.3\text{V}$. Sai số lượng tử hóa tối đa là bao nhiêu mili-Volt?*
2. **Câu hỏi 2 (Mạch đệm còi):** *Tại sao không thể nối trực tiếp còi Piezo Buzzer vào chân GPIO của vi điều khiển? Nếu còi Buzzer ăn dòng $45\text{mA}$, hãy chứng minh rằng transistor NPN 2N2222 với điện trở cực Base $R_B = 1\text{k}\Omega$ hoạt động hoàn toàn ở chế độ bão hòa (Saturation).*
3. **Câu hỏi 3 (Tư duy nhúng):** *Phân tích hậu quả nếu lập trình viên dùng lệnh `HAL_Delay(500)` để tạo chu kỳ nhấp nháy cho còi báo động trong một hệ thống giám sát rò rỉ khí gas nguy cấp.*
4. **Bài tập thiết kế mạch:** *Thiết kế mạch nguyên lý đo mức chất lỏng bằng cảm biến áp suất tương tự ngõ ra $0 \div 5\text{V}$ kết nối vào chân `PA1` của STM32F407 (ngõ vào chịu áp tối đa 3.3V). Tính toán cầu phân áp và mạch lọc thông thấp RC.*
