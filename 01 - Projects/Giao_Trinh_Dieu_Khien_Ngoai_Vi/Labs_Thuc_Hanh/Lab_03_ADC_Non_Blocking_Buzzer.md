# BÀI THỰC HÀNH SỐ 03: QUẢN LÝ THỜI GIAN KHÔNG CHẶN VÀ THU THẬP TÍN HIỆU CẢM BIẾN ADC
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết chuyển đổi tương tự-số $\rightarrow$ Mạch lọc $\rightarrow$ Lập trình $\rightarrow$ Đo kiểm)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Hiểu SAR ADC), CLO 2 (Tính lượng tử hóa & mạch lọc RC), CLO 3 (Lập trình Non-blocking & Moving Average), CLO 5 (Đo kiểm)  
**Căn cứ giáo trình:** *ARM Education - Chương 3: Real-Time Clock and Time Management, Analog-to-Digital Conversion (Ex 3.1 $\div$ 3.5)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Chuyển đổi Tương tự - Số (ADC):**
   - Nắm vững nguyên lý hoạt động của khối SAR ADC 12-bit trên STM32F407: Điện áp tham chiếu $V_{REF+} = 3.3\text{V}$, bước lượng tử hóa $q = \frac{3.3\text{V}}{4095} \approx 0.8056\text{ mV}$.
   - Phân tích đáp ứng điện áp tuyến tính của cảm biến nhiệt độ tương tự LM35 ($10\text{ mV}/^\circ\text{C}$) và cảm biến khí nồng độ gas MQ-2.
   - Thiết kế mạch lọc thông thấp RC (Anti-aliasing filter) cắt nhiễu tần số cao trước khi đưa vào chân ADC.
2. **Kỹ thuật Lập trình Nhúng Hiện đại:**
   - Xóa bỏ tư duy sử dụng hàm `HAL_Delay()` (Blocking delay làm tê liệt CPU); chuyển đổi 100% sang kỹ thuật **Non-blocking Timing** dựa trên bộ đếm thời gian hệ thống SysTick (`HAL_GetTick()`).
   - Cấu hình ADC đa kênh tuần tự (Scan Conversion) lấy mẫu luân phiên: Biến trở đặt ngưỡng an toàn (`PA1`) và Cảm biến nhiệt độ LM35 (`PA2`).
   - Cài đặt thuật toán **Lọc số trung bình trượt (Moving Average Filter)** 8 mẫu để triệt tiêu nhiễu giật giá trị ADC.
   - Lập trình điều khiển còi Piezo Buzzer (`PD11`) phát tiếng bíp ngắt quãng với tần số chu kỳ thay đổi tỷ lệ thuận với mức độ vượt ngưỡng cảnh báo (Ex 3.1 $\div$ 3.5).
3. **Đo kiểm & Đánh giá:**
   - Dùng máy hiện sóng quan sát dạng sóng điện áp analog trên chân LM35 trước và sau khi lọc RC.
   - Đo chu kỳ xung kích còi Buzzer trên Logic Analyzer / Oscilloscope để kiểm chứng tính phi khối (non-blocking).

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Thiết bị / Linh kiện | Thông số kỹ thuật | Số lượng | Mục đích |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Bo mạch STM32F407 | STM32F407VGT6 | 1 | Xử lý trung tâm |
| 2 | Cảm biến nhiệt độ LM35DZ | Dải đo $0 \div 100^\circ\text{C}$, độ dốc $10\text{ mV}/^\circ\text{C}$ | 1 | Đo nhiệt độ tương tự |
| 3 | Biến trở xoay (Potentiometer) | $10\text{ k}\Omega$ đơn vòng hoặc đa vòng | 1 | Đặt ngưỡng nhiệt độ kích hoạt |
| 4 | Còi báo Piezo Buzzer 5V/3V | Loại tích cực (Active Buzzer) | 1 | Phát âm thanh cảnh báo |
| 5 | Transistor NPN 2N2222 / C1815 | $V_{CEO} = 40\text{V}, I_C = 600\text{mA}$ | 1 | Mạch đệm dòng lái còi Buzzer |
| 6 | Điện trở & Tụ điện | Trở $1\text{ k}\Omega, 10\text{ k}\Omega$; Tụ gốm $100\text{ nF}$ (104) | Mỗi loại 3 | Phân cực transistor & lọc nhiễu RC |
| 7 | Máy đo | Oscilloscope 2CH & Logic Analyzer | 1 | Bắt tín hiệu analog và kiểm tra timing |

---

## 3. THIẾT KẾ MẠCH NGUYÊN LÝ & ĐẤU NỐI LINH KIỆN

### 3.1. Tính toán mạch điện tử
1. **Mạch lọc thông thấp RC cho chân tín hiệu LM35:**
   - Tín hiệu nhiệt độ biến thiên rất chậm (tần số dải thông dưới vài Hz). Nhiễu môi trường công nghiệp và xung clock thường nằm ở tần số hàng chục kHz.
   - Chọn $R = 1\text{ k}\Omega$, $C = 100\text{ nF} = 0.1\mu\text{F}$.
   $$f_c = \frac{1}{2\pi R C} = \frac{1}{2\pi \times 1000 \times 100 \times 10^{-9}} \approx 1{,}591\text{ Hz} \approx 1.6\text{ kHz}.$$
   *(Tần số cắt $1.6\text{ kHz}$ triệt tiêu hoàn toàn sóng gợn cao tần nhưng giữ nguyên vẹn tín hiệu nhiệt độ).*

2. **Mạch đệm transistor điều khiển còi Piezo Buzzer:**
   - Chân GPIO của STM32F4 chỉ cấp dòng tối đa $25\text{mA}$. Còi Buzzer khi kêu cần dòng $30 \div 50\text{mA}$, do đó bắt buộc phải dùng Transistor NPN đệm dòng:
   - Cực Base của NPN nối vào chân `PD11` qua điện trở hạn dòng $R_B = 1\text{ k}\Omega$.
   - Dòng kích Base: $I_B = \frac{3.3\text{V} - 0.7\text{V}}{1\text{ k}\Omega} = 2.6\text{mA}$.
   - Với độ lợi dòng $\beta \approx 100$, dòng bão hòa cực góp $I_C = \beta I_B \ge 260\text{mA}$, đảm bảo transistor mở bão hòa đóng ngắt còi an toàn.

### 3.2. Bảng đấu nối chân hệ thống
| Chân STM32F407 | Ngoại vi / Linh kiện | Chi tiết kết nối mạch | Chức năng |
| :---: | :---: | :--- | :--- |
| `PA1` (ADC1_IN1) | Biến trở $10\text{ k}\Omega$ | Chân giữa biến trở nối vào `PA1`; Hai chân bìa nối `3.3V` và `GND` | Điện áp ngưỡng $0 \div 3.3\text{V}$ |
| `PA2` (ADC1_IN2) | Cảm biến LM35 | Chân Vout của LM35 qua trở $1\text{k}\Omega \rightarrow$ `PA2`, tụ $100\text{nF}$ từ `PA2` xuống `GND` | Điện áp nhiệt độ thực tế |
| `PD11` | Cực Base NPN lái Buzzer | `PD11` $\rightarrow$ Trở $1\text{k}\Omega \rightarrow$ Base NPN; Collector $\rightarrow$ Buzzer (-); Buzzer (+) $\rightarrow$ `3.3V` | Kích còi báo động Active-HIGH |
| `PD12` | LED đỏ cảnh báo | Anode qua trở $330\Omega \rightarrow$ `PD12`; Cathode $\rightarrow$ `GND` | Báo quá ngưỡng nhiệt |

---

## 4. CẤU HÌNH TRÊN STM32CubeMX
1. **Clock Tree:** Thiết lập HSE 8MHz $\rightarrow$ SYSCLK 168MHz $\rightarrow$ APB2 84MHz.
2. **Cấu hình khối ADC1:**
   - Chọn tab *Analog* $\rightarrow$ *ADC1*.
   - Kích hoạt kênh `IN1` và `IN2`: Tích chọn **IN1** và **IN2**.
   - Cửa sổ *ADC Settings*:
     - Clock Prescaler: **PCLK2 divided by 4** ($84\text{ MHz} / 4 = 21\text{ MHz} \le 36\text{ MHz}$ tối đa).
     - Resolution: **12 bits (15 ADC clock cycles)**.
     - Data Alignment: **Right alignment**.
     - Scan Conversion Mode: **Disabled** (Ta thực hiện đọc từng kênh độc lập theo yêu cầu hoặc Enabled nếu dùng DMA).
   - Chọn thời gian lấy mẫu (*Sampling Time*): **56 Cycles** cho cả 2 kênh để đảm bảo tụ lấy mẫu bên trong nạp đầy đủ điện áp từ nguồn trở kháng của cảm biến.
3. **Cấu hình GPIO & UART2:**
   - `PD11, PD12`: **GPIO_Output**.
   - `USART2`: Mode Asynchronous 115200 bps để gửi số liệu nhiệt độ lên màn hình PC.

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### 5.1. Hàm đọc kênh ADC độc lập
Viết vào `Core/Src/main.c`:

```c
/* USER CODE BEGIN 0 */
#include <stdio.h>

extern ADC_HandleTypeDef hadc1;
extern UART_HandleTypeDef huart2;

int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}

/* Hàm cấu hình và đọc giá trị số từ 1 kênh ADC chỉ định */
uint32_t Read_ADC_Channel(uint32_t channel)
{
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel = channel;
    sConfig.Rank = 1;
    sConfig.SamplingTime = ADC_SAMPLETIME_56CYCLES;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);

    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 10);
    uint32_t adcVal = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);

    return adcVal;
}

/* Bộ lọc trung bình trượt (Moving Average Filter) 8 mẫu */
#define FILTER_DEPTH 8
float Filter_Temperature(float newSample)
{
    static float samples[FILTER_DEPTH] = {0};
    static uint8_t sampleIdx = 0;
    static float runningSum = 0;

    runningSum -= samples[sampleIdx];
    samples[sampleIdx] = newSample;
    runningSum += newSample;
    
    sampleIdx = (sampleIdx + 1) % FILTER_DEPTH;
    return (runningSum / FILTER_DEPTH);
}
/* USER CODE END 0 */
```

### 5.2. Thuật toán Non-blocking Timing & Cảnh báo âm thanh (Ex 3.1 $\div$ 3.5)
Tại vòng lặp `while(1)`:

```c
/* USER CODE BEGIN 2 */
uint32_t lastAdcReadTime = 0;
uint32_t lastPrintTime = 0;
uint32_t lastBuzzerToggleTime = 0;

float currentTempC = 0.0f;
float thresholdTempC = 0.0f;
uint8_t isBuzzerOn = 0;
uint32_t buzzerIntervalMs = 0; /* 0: Im lặng, >0: Chu kỳ bíp */
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    uint32_t currentMillis = HAL_GetTick();

    /* TÁC VỤ 1: Đọc ADC và Lọc số mỗi 50ms (Tần số lấy mẫu 20Hz) - KHÔNG DÙNG DELAY */
    if (currentMillis - lastAdcReadTime >= 50)
    {
        lastAdcReadTime = currentMillis;

        /* Đọc biến trở PA1 đặt ngưỡng (0 - 4095 ánh xạ sang 20.0 - 60.0 độ C) */
        uint32_t potRaw = Read_ADC_Channel(ADC_CHANNEL_1);
        thresholdTempC = 20.0f + ((float)potRaw / 4095.0f) * 40.0f;

        /* Đọc LM35 PA2: Điện áp V_out = Raw * 3300mV / 4095; Temp = V_out / 10mV/C */
        uint32_t lm35Raw = Read_ADC_Channel(ADC_CHANNEL_2);
        float rawVoltageMv = ((float)lm35Raw * 3300.0f) / 4095.0f;
        float rawTempC = rawVoltageMv / 10.0f;

        /* Áp dụng bộ lọc trung bình trượt */
        currentTempC = Filter_Temperature(rawTempC);

        /* Xác định mức độ cảnh báo */
        if (currentTempC >= thresholdTempC)
        {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET); /* Bật LED đỏ */
            float delta = currentTempC - thresholdTempC;
            
            /* Càng vượt ngưỡng nhiều, tiếng bíp càng dồn dập */
            if (delta > 5.0f) {
                buzzerIntervalMs = 100; /* Báo động khẩn cấp: 100ms */
            } else {
                buzzerIntervalMs = 400; /* Báo động cảnh báo: 400ms */
            }
        }
        else
        {
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
            buzzerIntervalMs = 0; /* An toàn: Tắt còi */
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_11, GPIO_PIN_RESET);
            isBuzzerOn = 0;
        }
    }

    /* TÁC VỤ 2: Điều khiển còi Buzzer nhấp nháy Non-blocking */
    if (buzzerIntervalMs > 0)
    {
        if (currentMillis - lastBuzzerToggleTime >= buzzerIntervalMs)
        {
            lastBuzzerToggleTime = currentMillis;
            isBuzzerOn = !isBuzzerOn;
            HAL_GPIO_WritePin(GPIOD, GPIO_PIN_11, isBuzzerOn ? GPIO_PIN_SET : GPIO_PIN_RESET);
        }
    }

    /* TÁC VỤ 3: Gửi báo cáo định kỳ lên PC Terminal mỗi 1000ms */
    if (currentMillis - lastPrintTime >= 1000)
    {
        lastPrintTime = currentMillis;
        printf("[GIAM SAT] Nhiet do: %.2f oC | Nguong cai dat: %.2f oC | Canh bao: %s\r\n",
               currentTempC, thresholdTempC, 
               (buzzerIntervalMs > 0) ? "NGUY HIEM (ALARM ON)" : "AN TOAN (NORMAL)");
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & PHÂN TÍCH TÍN HIỆU THỰC TẾ

### 6.1. Đo kiểm tín hiệu Analog trên Oscilloscope
1. Kết nối kênh `CH1` vào chân Vout trực tiếp của LM35; kết nối kênh `CH2` vào chân sau mạch lọc RC (tại chân `PA2` vi điều khiển).
2. Bật nguồn, hơ tay hoặc bật quẹt lửa gần LM35 (không đốt trực tiếp vào vỏ nhựa).
3. Quan sát trên Oscilloscope:
   - Kênh CH1 có biên độ nhiễu gai nhọn (Noise spikes) biên độ $20 \div 50\text{mV}$ do ghép tạp âm nguồn.
   - Kênh CH2 sau mạch lọc RC có dạng sóng mượt mà, đường dốc nhiệt độ tăng dần đều đặn, không còn gai nhiễu.

### 6.2. Kiểm chứng tính chất Non-blocking của hệ thống
1. Dùng Logic Analyzer cắm vào chân `PD11` (Buzzer) và `PA2` (UART TX).
2. Đo tần số lật trạng thái chân còi khi đang ở chế độ báo động dồn dập ($100\text{ms}$).
3. Quan sát thấy chuỗi ký tự UART vẫn được truyền ra đều đặn chính xác từng giây một mà không hề bị ngắt quãng hay bị trễ bởi hoạt động của còi báo động.

---

## 7. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 7.1. Báo cáo thí nghiệm
- Đồ thị đo điện áp chân LM35 trước và sau mạch lọc RC trên máy hiện sóng.
- Bảng số liệu so sánh nhiệt độ thực tế hiển thị trên Terminal máy tính so với nhiệt kế chuẩn trong phòng thí nghiệm.

### 7.2. Câu hỏi đánh giá
1. *Tại sao nếu dùng hàm `HAL_Delay(100)` để tạo tiếng bíp cho còi thì hệ thống không thể đọc được cảm biến nhiệt độ đúng chu kỳ 50ms?*
2. *Nếu điện áp nguồn $V_{DD}$ bị sụt giảm từ $3.3\text{V}$ xuống $3.1\text{V}$ do pin yếu, kết quả tính toán nhiệt độ của LM35 theo công thức trên sẽ tăng lên hay giảm đi so với thực tế? Giải thích.*
3. *Độ sâu của bộ lọc trung bình trượt (số lượng mẫu $N$) ảnh hưởng như thế nào đến độ mượt của tín hiệu và độ trễ đáp ứng (Response time)?*