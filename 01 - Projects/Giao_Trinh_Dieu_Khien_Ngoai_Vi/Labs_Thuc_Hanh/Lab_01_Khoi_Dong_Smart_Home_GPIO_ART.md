# BÀI THỰC HÀNH SỐ 01: KHỞI ĐỘNG HỆ THỐNG SMART HOME - GPIO NÂNG CAO VÀ FLASH ART ACCELERATOR
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết mạch $\rightarrow$ Cấu hình $\rightarrow$ Lập trình $\rightarrow$ Đo kiểm)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Hiểu Bus Matrix & ART), CLO 2 (Tính mạch LED/Nút), CLO 3 (Lập trình I/O), CLO 5 (Đo kiểm logic)  
**Căn cứ giáo trình:** *ARM Education - Chương 1: Introduction to Embedded Systems (Code 1.1 $\div$ 1.5)*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết & Mạch:**
   - Phân tích cấu trúc tầng ngõ ra GPIO vi điều khiển STM32F407 (Push-Pull, Open-Drain, Pull-up, Pull-down, Speed).
   - Thiết kế sơ đồ nguyên lý và tính toán trị số điện trở hạn dòng cho LED báo động và điện trở phân áp/kéo lên cho nút bấm, cảm biến phát hiện khí Gas / Quá nhiệt.
2. **Kỹ thuật Lập trình:**
   - Khởi tạo project chuẩn trên STM32CubeIDE cho STM32F407 (Clock 168MHz qua HSE thạch anh ngoài 8MHz).
   - Lập trình đọc đồng thời nhiều ngõ vào số (Gas `PE2`, Nhiệt `PE3`, Nút `PE4`, `PE5`) và điều khiển LED cảnh báo `PD12`.
   - Ứng dụng kỹ thuật gom bit thanh ghi ngõ vào `IDR` và ngõ ra `ODR`/`BSRR` để kiểm tra toàn bộ điều kiện an toàn chỉ trong 1 chu kỳ xung nhịp.
3. **Đo kiểm & Under the Hood:**
   - Dùng máy hiện sóng (Oscilloscope) đo thời gian đáp ứng từ lúc chân ngõ vào chuyển mức logic đến khi chân ngõ ra LED kích hoạt.
   - Bật/tắt Flash ART Accelerator (Adaptive Real-Time Accelerator) và kiểm chứng thời gian thực thi lệnh khi chạy từ bộ nhớ Flash không chu kỳ chờ (0-wait state equivalent).

---

## 2. CHUẨN BỊ THIẾT BỊ VÀ LINH KIỆN
| STT | Tên thiết bị / Module / Linh kiện | Thông số kỹ thuật / Mã hiệu | Số lượng | Mục đích sử dụng |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Kit phát triển STM32F407 | STM32F407VGT6 (Discovery / Black Board) | 1 | Bo mạch điều khiển trung tâm |
| 2 | Mạch nạp & Gỡ lỗi | ST-Link V2 tích hợp trên bo hoặc cắm rời | 1 | Nạp chương trình và Debug SWD |
| 3 | LED đơn 5mm | Màu đỏ / xanh lục, $V_F \approx 2.0\text{V}, I_F \approx 5\text{mA}$ | 4 | Chỉ thị trạng thái báo động |
| 4 | Nút bấm 4 chân | Nút nhấn nhả (Tactile Switch) 6x6mm | 2 | Nút A kích hoạt, Nút B vô hiệu |
| 5 | Công tắc gạt logic (hoặc DIP Switch) | Công tắc 2 vị trí đóng/ngắt | 2 | Giả lập cảm biến Gas & Quá nhiệt |
| 6 | Điện trở kim loại 0.25W | $330\Omega$ (hạn dòng LED), $10\text{ k}\Omega$ (Pull-down/Pull-up) | Mỗi loại 4 | Lắp mạch ngoại vi trên breadboard |
| 7 | Breadboard & Dây nối | Breadboard 830 lỗ, bộ dây cắm Breadboard đực-đực/đực-cái | 1 bộ | Đấu nối mạch thí nghiệm |
| 8 | Thiết bị đo | Logic Analyzer (24MHz 8CH) hoặc Oscilloscope | 1 | Đo đáp ứng thời gian và bắt mức logic |

---

## 3. SƠ ĐỒ NGUYÊN LÝ VÀ ĐẤU NỐI MẠCH (SCHEMATIC & PIN CONNECTIONS)

### 3.1. Tính toán trị số linh kiện
1. **Điện trở hạn dòng LED báo động (`PD12`):**
   - Vi điều khiển STM32F407 cấp mức logic cao $V_{OH} = 3.3\text{V}$.
   - LED xanh lục có điện áp phân cực thuận $V_F \approx 2.1\text{V}$, dòng định mức an toàn $I_F = 4\text{mA}$.
   $$R_{LED} = \frac{V_{OH} - V_F}{I_F} = \frac{3.3\text{V} - 2.1\text{V}}{0.004\text{A}} = 300\Omega \implies \text{Chọn điện trở chuẩn } 330\Omega.$$
   *(Dòng điện $I_{LED} = 3.63\text{mA} < 25\text{mA}$, hoàn toàn nằm trong giới hạn chịu đựng của chân GPIO STM32F407).*

2. **Mạch nút bấm và cảm biến logic ngõ vào (`PE2`, `PE3`, `PE4`, `PE5`):**
   - Sử dụng mạch nút bấm tích cực mức cao (Active-HIGH) kèm trở kéo xuống (Pull-down) ngoài $R_{pull} = 10\text{ k}\Omega$:
   - Khi nút hở: Chân GPIO nhận mức $0\text{V}$ (Logic 0) qua điện trở kéo xuống.
   - Khi nút đóng: Chân GPIO nối thẳng lên $V_{DD} = 3.3\text{V}$ (Logic 1), dòng rò qua trở kéo là $I = 3.3\text{V}/10\text{k}\Omega = 0.33\text{mA}$ (tiết kiệm điện năng).

### 3.2. Bảng đấu nối dây chi tiết
| Chân STM32F407 | Ngoại vi kết nối | Chi tiết đấu nối trên Breadboard | Mức logic tác động |
| :---: | :---: | :--- | :---: |
| `PD12` | LED Báo động trung tâm | Anode qua trở $330\Omega \rightarrow$ `PD12`; Cathode $\rightarrow$ `GND` | Active-HIGH (Mức 1 = Sáng) |
| `PD13` | LED Chỉ thị hệ thống ON | Anode qua trở $330\Omega \rightarrow$ `PD13`; Cathode $\rightarrow$ `GND` | Active-HIGH (Mức 1 = Sáng) |
| `PE2` | Cảm biến Gas (Công tắc 1) | Đầu 1 $\rightarrow$ `3.3V`; Đầu 2 $\rightarrow$ `PE2` và trở $10\text{k}\Omega$ xuống `GND` | Active-HIGH (1 = Có rò rỉ Gas) |
| `PE3` | Cảm biến Nhiệt (Công tắc 2) | Đầu 1 $\rightarrow$ `3.3V`; Đầu 2 $\rightarrow$ `PE3` và trở $10\text{k}\Omega$ xuống `GND` | Active-HIGH (1 = Quá nhiệt) |
| `PE4` | Nút nhấn A (Kích hoạt) | Đầu 1 $\rightarrow$ `3.3V`; Đầu 2 $\rightarrow$ `PE4` và trở $10\text{k}\Omega$ xuống `GND` | Active-HIGH (1 = Đang nhấn) |
| `PE5` | Nút nhấn B (Vô hiệu hóa) | Đầu 1 $\rightarrow$ `3.3V`; Đầu 2 $\rightarrow$ `PE5` và trở $10\text{k}\Omega$ xuống `GND` | Active-HIGH (1 = Đang nhấn) |
| `3V3` | Nguồn cấp Breadboard | Nối vào thanh nguồn dương $(+)$ của Breadboard | Tuyệt đối không cắm nhầm 5V! |
| `GND` | Mass hệ thống | Nối vào thanh nguồn âm $(-)$ của Breadboard | Chung mass giữa STM32 & Đo |

> [!CAUTION]
> Tuyệt đối không cấp nguồn $5\text{V}$ vào các chân GPIO của STM32F407 khi chân đó chưa được xác nhận là loại chịu điện áp 5V (5V-tolerant `FT`). Để đảm bảo an toàn tuyệt đối, toàn bộ tín hiệu ngoại vi trong bài Lab đều sử dụng chuẩn điện áp **$3.3\text{V}$**.

---

## 4. QUY TRÌNH CẤU HÌNH TRÊN STM32CubeMX
1. **Tạo Project mới:**
   - Khởi động STM32CubeIDE $\rightarrow$ *File* $\rightarrow$ *New* $\rightarrow$ *STM32 Project*.
   - Nhập mã chip: `STM32F407VGTx`.
2. **Cấu hình Xung nhịp hệ thống (RCC & Clock Tree):**
   - Mục *System Core* $\rightarrow$ *RCC*: Tại `High Speed Clock (HSE)`, chọn **Crystal/Ceramic Resonator**.
   - Chuyển sang tab *Clock Configuration*:
     - Chọn nguồn `PLL Source Mux` là **HSE** ($8\text{ MHz}$).
     - Cấu hình bộ nhân PLL: chia $M = 8$, nhân $N = 336$, chia $P = 2 \implies \text{SYSCLK} = 168\text{ MHz}$.
     - Bộ chia bus: `AHB Prescaler = /1` (168MHz), `APB1 Prescaler = /4` (42MHz), `APB2 Prescaler = /2` (84MHz).
3. **Cấu hình GPIO:**
   - Click chuột trái vào chân `PD12` $\rightarrow$ Chọn **GPIO_Output**.
   - Click chuột trái vào chân `PD13` $\rightarrow$ Chọn **GPIO_Output**.
   - Click chuột trái vào các chân `PE2`, `PE3`, `PE4`, `PE5` $\rightarrow$ Chọn **GPIO_Input**.
   - Mở rộng rộng mục *System Core* $\rightarrow$ *GPIO*:
     - `PD12, PD13`: *GPIO Output Level* = **Low**, *GPIO mode* = **Output Push-Pull**, *Pull-up/Pull-down* = **No pull-up and no pull-down**, *Maximum output speed* = **Very High**.
     - `PE2, PE3, PE4, PE5`: *GPIO mode* = **Input mode**, *Pull-up/Pull-down* = **No pull-up and no pull-down** (vì ta đã lắp trở $10\text{k}\Omega$ ngoài breadboard).
4. **Cấu hình Flash ART Accelerator:**
   - Mục *System Core* $\rightarrow$ *CORTEX_M4*: Đảm bảo đã tích chọn **ART Accelerator Instruction cache enabled** và **ART Accelerator Data cache enabled**.
5. **Sinh mã nguồn (Generate Code):**
   - Đặt tên project: `Lab01_SmartHome_GPIO_F407`.
   - Toolchain / IDE: chọn **STM32CubeIDE** $\rightarrow$ Nhấn **Finish**.

---

## 5. MÃ NGUỒN THỰC HÀNH TỪNG BƯỚC

### Nhiệm vụ 1: Lập trình kiểm tra điều kiện an toàn tuần tự (Mô phỏng Code 1.1 $\div$ 1.3)
Mở tệp `Core/Src/main.c`. Tại vòng lặp `while (1)` trong `main()`, viết mã nguồn:

```c
/* USER CODE BEGIN WHILE */
while (1)
{
    /* Đọc trạng thái các cảm biến và nút bấm từ Port E */
    GPIO_PinState gasSensorState   = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_2);
    GPIO_PinState tempSensorState  = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_3);
    GPIO_PinState buttonA_State    = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_4);
    GPIO_PinState buttonB_State    = HAL_GPIO_ReadPin(GPIOE, GPIO_PIN_5);

    /* Logic kiểm tra: Kích hoạt báo động khi Gas VÀ Quá nhiệt cùng xảy ra, 
       HOẶC khi cả nút A và nút B cùng được nhấn */
    if ((gasSensorState == GPIO_PIN_SET && tempSensorState == GPIO_PIN_SET) ||
        (buttonA_State == GPIO_PIN_SET && buttonB_State == GPIO_PIN_SET))
    {
        /* Bật đèn LED báo động nguy cấp */
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
    }
    else
    {
        /* Tắt đèn LED khi trạng thái an toàn */
        HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET);
    }

    /* Đèn LED PD13 nhấp nháy báo hiệu hệ thống (Heartbeat) chu kỳ 500ms */
    HAL_GPIO_TogglePin(GPIOD, GPIO_PIN_13);
    HAL_Delay(500);

    /* USER CODE END WHILE */
```

### Nhiệm vụ 2: Tối ưu hóa đọc gom bit thanh ghi (Bit-masking) theo chuẩn Code 1.4 ARM
Trong các hệ thống thời gian thực khắt khe, việc gọi 4 hàm `HAL_GPIO_ReadPin` tiêu tốn hàng chục chu kỳ clock. Ta thay thế bằng kỹ thuật đọc trực tiếp thanh ghi `GPIOE->IDR`:

```c
/* USER CODE BEGIN 2 */
/* Định nghĩa mặt nạ bit cho các cảm biến trên Port E */
#define MASK_GAS_SENSOR     (1U << 2)   /* PE2 */
#define MASK_TEMP_SENSOR    (1U << 3)   /* PE3 */
#define MASK_BUTTON_A       (1U << 4)   /* PE4 */
#define MASK_BUTTON_B       (1U << 5)   /* PE5 */

#define MASK_FIRE_HAZARD    (MASK_GAS_SENSOR | MASK_TEMP_SENSOR)
#define MASK_MANUAL_PANIC   (MASK_BUTTON_A | MASK_BUTTON_B)
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    /* Đọc toàn bộ 16 chân ngõ vào Port E trong DUY NHẤT 1 chu kỳ xung nhịp */
    uint32_t portE_inputs = GPIOE->IDR;

    /* Đánh giá đồng thời bằng phép toán logic bit */
    uint8_t isFireEmergency   = ((portE_inputs & MASK_FIRE_HAZARD) == MASK_FIRE_HAZARD);
    uint8_t isManualPanic     = ((portE_inputs & MASK_MANUAL_PANIC) == MASK_MANUAL_PANIC);

    if (isFireEmergency || isManualPanic)
    {
        /* Kích hoạt LED PD12 qua thanh ghi BSRR (Atomic operation) */
        GPIOD->BSRR = GPIO_PIN_12; 
    }
    else
    {
        /* Tắt LED PD12 qua nửa trên thanh ghi BSRR (Bit Reset) */
        GPIOD->BSRR = (uint32_t)GPIO_PIN_12 << 16;
    }
}
/* USER CODE END WHILE */
```

---

## 6. ĐO KIỂM & PHÂN TÍCH DẠNG SÓNG TÍN HIỆU

### 6.1. Thiết lập đo lường trên Oscilloscope / Logic Analyzer
1. Kết nối kênh **CH1 (Màu vàng)** vào chân ngõ vào `PE2` (Gas Sensor).
2. Kết nối kênh **CH2 (Màu xanh)** vào chân ngõ ra `PD12` (Alarm LED).
3. Kẹp dây mass của cả 2 que đo vào cọc `GND` của bo mạch STM32F407.
4. Thiết lập Trigger trên Oscilloscope:
   - Source: **CH1**.
   - Slope: **Rising Edge** (Sườn lên).
   - Mode: **Normal** (hoặc Single).
   - Mức điện áp kích hoạt Trigger Level: $1.65\text{V}$.
   - Timebase: $50\text{ ns/div}$ hoặc $100\text{ ns/div}$.

### 6.2. Kiểm tra độ trễ đáp ứng (Propagation Latency)
1. Giữ nút `PE3` (Nhiệt độ) ở mức cao. Bấm nút `PE2` (Gas) để tạo sườn lên kích hoạt báo động.
2. Đo khoảng thời gian $\Delta t$ giữa lúc CH1 đạt $1.65\text{V}$ đến lúc CH2 đạt $1.65\text{V}$:
   - Khi dùng hàm HAL tiêu chuẩn: $\Delta t_{HAL} \approx 450 \div 850\text{ ns}$.
   - Khi dùng đọc thanh ghi trực tiếp `GPIOE->IDR`: $\Delta t_{Direct} \approx 60 \div 95\text{ ns}$ (chỉ khoảng 10-15 chu kỳ xung nhịp ở 168MHz).

```
   CH1 (PE2 Input) : ______/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
                           |<- Δt ->|
   CH2 (PD12 Output): _______________/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
```

### 6.3. Thử nghiệm ART Accelerator "Under the Hood"
1. Trong file `main.c`, vô hiệu hóa Flash Prefetch và ART Cache:
   ```c
   __HAL_FLASH_INSTRUCTION_CACHE_DISABLE();
   __HAL_FLASH_DATA_CACHE_DISABLE();
   __HAL_FLASH_PREFETCH_BUFFER_DISABLE();
   ```
2. Thực hiện vòng lặp tính toán kiểm tra 100,000 lần và đo tần số chân `PD12` lật trạng thái:
   - Khi **Tắt ART**: Tần số lật trạng thái bị tụt giảm nghiêm trọng do Flash STM32F4 chỉ chạy tối đa 30MHz (cần tới 5 Wait States ở 168MHz).
   - Khi **Bật ART**: Tần số tăng vọt gần gấp đôi nhờ bộ đệm nạp trước (Instruction Cache 64-line x 128-bit) cho phép thực thi 0-wait state.

---

## 7. XỬ LÝ SỰ CỐ THƯỜNG GẶP (TROUBLESHOOTING)
1. **Hiện tượng LED không sáng khi bấm nút:**
   - *Kiểm tra 1:* Dùng đồng hồ VOM đo điện áp tại chân `PE2`: Khi chưa nhấn phải là $0\text{V}$, khi nhấn phải là $3.3\text{V}$. Nếu luôn là $0\text{V}$, kiểm tra lại tiếp xúc chân nút bấm trên breadboard.
   - *Kiểm tra 2:* LED bị cắm ngược cực Anode - Cathode. Chân dài là Anode (nối vào trở hạn dòng), cạnh vát phẳng trên thân LED là Cathode (nối mass).
2. **Hiện tượng vi điều khiển bị treo sau khi nạp code:**
   - *Nguyên nhân:* Cấu hình sai thạch anh HSE trong STM32CubeMX (chọn nhầm tần số thạch anh khác 8MHz, hoặc chọn HSE nhưng kit không hàn thạch anh ngoài).
   - *Khắc phục:* Kiểm tra xem kit dùng thạch anh 8MHz hay 25MHz để cấu hình đúng bộ chia $M$ trong tab Clock Tree.

---

## 8. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 8.1. Yêu cầu sản phẩm nộp của sinh viên
1. Ảnh chụp sơ đồ mạch thực tế trên Breadboard nối với STM32F407.
2. Ảnh chụp màn hình máy hiện sóng (Oscilloscope) hiển thị thời gian trễ đáp ứng $\Delta t$ giữa ngõ vào và ngõ ra.
3. Mã nguồn `main.c` hoàn chỉnh được chú thích rõ ràng.

### 8.2. Câu hỏi đánh giá & Mở rộng (Vấn đáp cuối buổi)
1. **Câu hỏi 1 (Lý thuyết mạch):** *Tại sao không nên nối trực tiếp ngõ ra `PD12` vào cực dương LED mà bắt buộc phải có điện trở hạn dòng $330\Omega$? Nếu nối thẳng thì hiện tượng vật lý gì sẽ xảy ra?*
2. **Câu hỏi 2 (Thanh ghi):** *Sự khác biệt cốt lõi giữa việc ghi ra chân qua thanh ghi `ODR` và thanh ghi `BSRR` là gì? Tại sao trong hệ thống có nhiều luồng hoặc ngắt, ghi qua `BSRR` được coi là an toàn tuyệt đối (Atomic)?*
3. **Câu hỏi 3 (Mở rộng):** *Viết một đoạn macro C kiểm tra xem trong 4 cảm biến (`PE2..PE5`), có ĐÚNG MỘT cảm biến tích cực (Parity check / One-hot check) để kích hoạt chế độ cảnh báo mức nhẹ.*