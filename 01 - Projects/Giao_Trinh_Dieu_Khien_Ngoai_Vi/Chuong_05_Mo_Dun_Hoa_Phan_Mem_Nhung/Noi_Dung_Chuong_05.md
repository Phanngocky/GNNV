# CHƯƠNG 5: MÔ-ĐUN HÓA MÃ NGUỒN PHẦN MỀM HỆ THỐNG NHÚNG TRÊN STM32F407
*(Biên soạn dựa trên Chapter 5: Modularization Applied to Embedded Systems Programming - Arm Education Textbook, hiệu chỉnh chuyên sâu cho STM32F407)*

---

## 5.1. MỤC TIÊU CỦA CHƯƠNG
- **Bối cảnh kiến thức:** Người mới học lập trình vi điều khiển thường có thói quen nguy hiểm là viết dồn toàn bộ mã nguồn vào một tệp duy nhất `main.c`. Khi hệ thống nhúng mở rộng nhiều khối ngoại vi (ADC, UART, RTC, Bàn phím ma trận, Màn hình hiển thị), tệp `main.c` dài hàng nghìn dòng sẽ trở nên "bất khả kiểm soát", cực kỳ dễ phát sinh lỗi xung đột tên biến và không thể làm việc nhóm.
- **Mục tiêu chi tiết:**
  1. **Lý thuyết Công Nghệ Phần Mềm Nhúng:** Nắm vững các nguyên lý thiết kế phần mềm cốt lõi: **Mô-đun hóa (Modularization)**, **Phân tách trách nhiệm (Separation of Concerns)**, **Độ kết dính cao (High Cohesion)** và **Độ liên kết lỏng (Low Coupling)**.
  2. **Vòng Đời Biên Dịch & Quản Lý Bộ Nhớ C:** Hiểu rõ 4 giai đoạn biên dịch C (Preprocessor $\rightarrow$ Compiler $\rightarrow$ Assembler $\rightarrow$ Linker), phân biệt các phân vùng nhớ (Text, Data, BSS, Stack, Heap), và kỹ thuật đóng gói bảo vệ dữ liệu bằng từ khóa `static` và chia sẻ giao diện công khai bằng `extern`.
  3. **Tiêu Chuẩn Lập Trình An Toàn MISRA-C:** Áp dụng các quy tắc lập trình nhúng an toàn trong ngành ô tô / hàng không (MISRA-C:2012): Sử dụng kiểu dữ liệu có độ rộng cố định `<stdint.h>`, loại bỏ đệ quy, cấm cấp phát bộ nhớ động (`malloc`/`free`), và kiểm tra ranh giới con trỏ.
  4. **Tái Cấu Trúc Toàn Diện Hệ Thống Smart Home (Refactoring):** Tách toàn bộ mã nguồn từ Chương 1 đến Chương 4 thành 6 mô-đun độc lập (`inputs`, `outputs`, `alarm_system`, `matrix_keypad`, `pc_serial_com`, `real_time_clock`), tối giản tệp `main.c` thành vòng lặp điều phối sạch sẽ (Clean Architecture).

---

## 5.2. CƠ SỞ LÝ THUYẾT CHUYÊN SÂU

### 5.2.1. Vòng Đời Biên Dịch Mã Nguồn C Trên Hệ Thống Nhúng
Để chuyển đổi các tệp mã nguồn C sang mã máy nhị phân chạy trên lõi ARM Cortex-M4, chuỗi công cụ GNU ARM GCC Toolchain thực hiện 4 bước kế tiếp:

```
+------------+       +------------+
|  inputs.c  |       |  inputs.h  |
+------------+       +------------+
       |                   |
       v                   v
+---------------------------------------+
|  1. TRÌNH TIỀN XỬ LÝ (PREPROCESSOR)   | ---> Mở rộng macro #define, chèn nội dung #include,
+---------------------------------------+      xử lý biên dịch có điều kiện #ifdef
                   |
                   v (inputs.i)
+---------------------------------------+
|  2. TRÌNH BIÊN DỊCH (COMPILER - GCC)  | ---> Kiểm tra cú pháp, tối ưu hóa (-O2), sinh mã
+---------------------------------------+      hợp âm ARM Thumb-2 Assembly
                   |
                   v (inputs.s)
+---------------------------------------+
|  3. TRÌNH HỢP DỊCH (ASSEMBLER - AS)   | ---> Dịch các lệnh Assembly thành mã máy nhị phân
+---------------------------------------+      chưa định vị (Relocatable Object File)
                   |
                   v (inputs.o)
+---------------------------------------+      +-------------------+
|  4. BỘ LIÊN KẾT (LINKER - LD)         | <--- | Linker Script     | (stm32f407vg_flash.ld)
+---------------------------------------+      | (Bản đồ bộ nhớ)   |
                   |
                   +---> main.elf  (Tệp thực thi đầy đủ ký hiệu debug)
                   +---> main.bin  (Tệp nhị phân nạp thẳng vào Flash)
                   +---> main.hex  (Định dạng Intel Hex)
```

#### Vai trò sống còn của Header Guards:
Nếu một tệp tiêu đề `inputs.h` được cả `alarm_system.c` và `main.c` cùng `#include`, trình tiền xử lý sẽ sao chép nội dung tệp đó hai lần. Nếu trong tệp `.h` có định nghĩa cấu trúc dữ liệu (`struct`), trình biên dịch sẽ báo lỗi nghiêm trọng: **`redefinition of struct`**.  
$\implies$ **Kỹ thuật Header Guard bắt buộc:**
```c
#ifndef INPUTS_H_
#define INPUTS_H_

/* Toàn bộ khai báo hàm nguyên mẫu, kiểu dữ liệu, hằng số */

#endif /* INPUTS_H_ */
```
*(Trong các trình biên dịch hiện đại, chỉ thị `#pragma once` đặt ở đầu tệp có tác dụng tương đương nhưng ngắn gọn hơn).*

---

### 5.2.2. Phân Bổ Vùng Nhớ Trong Vi Điều Khiển & Từ Khóa `static` / `extern`
Bộ nhớ của STM32F407 được chia thành các phân vùng chuyên biệt:
1. **Phân vùng Text (Code Segment - Nằm trong Flash):** Chứa toàn bộ mã lệnh nhị phân thực thi và các biến hằng số (`const`). Vùng này chỉ đọc (Read-only).
2. **Phân vùng Data (Nằm trong RAM):** Chứa các biến toàn cục và biến `static` **đã được khởi tạo giá trị ban đầu khác 0** (ví dụ `uint32_t baud = 115200;`).
3. **Phân vùng BSS (Block Started by Symbol - Nằm trong RAM):** Chứa các biến toàn cục và biến `static` **chưa khởi tạo hoặc khởi tạo bằng 0**. Trình khởi động (Startup code) tự động xóa sạch vùng này về 0 khi cấp nguồn.
4. **Phân vùng Stack (Ngăn xếp - Nằm trong RAM):** Chứa các biến cục bộ khai báo bên trong hàm, địa chỉ quay về của chương trình con và ngữ cảnh khi xảy ra ngắt NVIC. Phát triển từ địa chỉ cao xuống địa chỉ thấp.
5. **Phân vùng Heap (Cấp phát động - Nằm trong RAM):** Dành cho các hàm `malloc()` và `free()`.

```
                    BẢN ĐỒ VÙNG NHỚ RAM STM32F407 (128 KB)
      Địa chỉ cao: 0x2002 0000 +----------------------------------+
                               |              STACK               |  (Phát triển đi xuống v)
                               |  (Biến cục bộ, Ngữ cảnh ngắt)    |
                               +----------------------------------+
                               |                |                 |
                               |                v                 |
                               |          VÙNG NHỚ TRỐNG          |
                               |                ^                 |
                               |                |                 |
                               +----------------------------------+
                               |              HEAP                |  (Phát triển đi lên ^)
                               |   (Bộ nhớ cấp phát malloc)       |
                               +----------------------------------+
                               |              BSS                 |
                               | (Biến toàn cục khởi tạo = 0)     |
                               +----------------------------------+
      Địa chỉ thấp: 0x2000 0000|              DATA                |
                               | (Biến toàn cục khởi tạo khác 0)  |
                               +----------------------------------+
```

#### Kiểm soát phạm vi biến bằng `static` và `extern`:
- **Từ khóa `static` đặt trước biến toàn cục hoặc hàm:** Giới hạn tầm nhìn của biến/hàm đó **chỉ duy nhất trong nội bộ tệp `.c` hiện thời**. Bộ liên kết (Linker) sẽ ẩn ký hiệu này đi, ngăn chặn hoàn toàn việc các tệp khác vô tình sửa đổi giá trị. Đây là cách hiện thực hóa tính đóng gói (Encapsulation / Private member) trong ngôn ngữ C!
- **Từ khóa `extern` đặt trước biến:** Thông báo cho trình biên dịch biết rằng: *"Biến này đã được cấp phát ô nhớ thực sự ở một tệp `.c` khác; ở tệp này tôi chỉ tham chiếu tới nó chứ không cấp phát thêm ô nhớ mới"*.

---

### 5.2.3. Tiêu Chuẩn An Toàn Phần Mềm Nhúng MISRA-C:2012
Hiệp hội Công nghiệp Ô tô Vương quốc Anh (Motor Industry Software Reliability Association) ban hành chuẩn MISRA-C nhằm loại bỏ các hành vi bất định (Undefined Behavior) của ngôn ngữ C:
1. **Quy tắc 4.6 (Kiểu dữ liệu cố định):** Tuyệt đối không dùng các kiểu biến số cơ bản không rõ độ rộng như `int`, `long`, `short`, `char`. Bắt buộc dùng chuẩn `<stdint.h>`: `int8_t`, `uint8_t`, `int16_t`, `uint16_t`, `int32_t`, `uint32_t`.
2. **Quy tắc 21.3 (Cấm cấp phát động):** Nghiêm cấm sử dụng các hàm cấp phát bộ nhớ động (`malloc`, `calloc`, `free`) trong hệ thống thời gian thực vì nguy cơ rò rỉ vùng nhớ (Memory Leak) và phân mảnh bộ nhớ (Heap Fragmentation) gây treo chip sau thời gian dài vận hành.
3. **Quy tắc 15.1 (Loại bỏ `goto`):** Không sử dụng câu lệnh nhảy `goto` vì phá vỡ cấu trúc luồng điều khiển của chương trình.
4. **Quy tắc 17.2 (Cấm đệ quy):** Các hàm không được phép gọi lại chính nó (Recursion) để ngăn chặn nguy cơ tràn ngăn xếp (Stack Overflow).

---

## 5.3. BẢNG PHÂN BỔ TÀI NGUYÊN CHÂN PHẦN CỨNG TỔNG HỢP CÁC MÔ-ĐUN

Hệ thống mã nguồn sau khi mô-đun hóa quản lý toàn diện các chân phần cứng trên STM32F407:

| Mô-đun Phần Mềm | Tệp Mã Nguồn (`.h` / `.c`) | Chân STM32 Quản Lý | Chức Năng Ngoại Vi |
| :--- | :--- | :---: | :--- |
| **Inputs Module** | `inputs.h` / `inputs.c` | `PE2`, `PE3` | Cảm biến rò rỉ Gas (MQ-2) & Cảm biến quá nhiệt |
| **Outputs Module** | `outputs.h` / `outputs.c` | `PD11`, `PD12`, `PD14`, `PD15` | Còi hú Buzzer, Đèn LED báo động, LED báo lỗi mã |
| **Keypad Module** | `matrix_keypad.h` / `.c` | `PE7..PE10` (Hàng)<br>`PE11..PE14` (Cột) | Quét bàn phím ma trận $4\times 4$ và chống rung |
| **Serial Module** | `pc_serial_com.h` / `.c` | `PA2` (TX), `PA3` (RX) | Giao tiếp nối tiếp máy tính PC qua `USART2` |
| **RTC Service** | `real_time_clock.h` / `.c` | `PC14, PC15`, `VBAT` | Đồng hồ thời gian thực LSE 32.768kHz |
| **Core Logic** | `alarm_system.h` / `.c` | Toàn hệ thống | Điều phối máy trạng thái FSM lõi |

---

## 5.4. HIỆN THỰC HÓA CHI TIẾT MÃ NGUỒN CÁC MÔ-ĐUN

### 5.4.1. Mô-đun Ngõ Vào Cảm Biến (`inputs.h` & `inputs.c`)
Tệp `Core/Inc/inputs.h`:
```c
#ifndef INC_INPUTS_H_
#define INC_INPUTS_H_

#include <stdbool.h>
#include "stm32f4xx_hal.h"

void Inputs_Init(void);
bool Inputs_IsGasDetected(void);
bool Inputs_IsOverTemp(void);

#endif /* INC_INPUTS_H_ */
```

Tệp `Core/Src/inputs.c`:
```c
#include "inputs.h"

#define GAS_PIN     GPIO_PIN_2
#define TEMP_PIN    GPIO_PIN_3
#define SENSOR_PORT GPIOE

void Inputs_Init(void)
{
    __HAL_RCC_GPIOE_CLK_ENABLE();
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    GPIO_InitStruct.Pin = GAS_PIN | TEMP_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLDOWN;
    HAL_GPIO_Init(SENSOR_PORT, &GPIO_InitStruct);
}

bool Inputs_IsGasDetected(void)
{
    return (HAL_GPIO_ReadPin(SENSOR_PORT, GAS_PIN) == GPIO_PIN_SET);
}

bool Inputs_IsOverTemp(void)
{
    return (HAL_GPIO_ReadPin(SENSOR_PORT, TEMP_PIN) == GPIO_PIN_SET);
}
```

---

### 5.4.2. Mô-đun Ngõ Ra Chấp Hành (`outputs.h` & `outputs.c`)
Tệp `Core/Inc/outputs.h`:
```c
#ifndef INC_OUTPUTS_H_
#define INC_OUTPUTS_H_

#include <stdbool.h>
#include <stdint.h>
#include "stm32f4xx_hal.h"

typedef enum {
    LED_ALARM_GREEN  = GPIO_PIN_12,
    LED_WRONG_CODE   = GPIO_PIN_14,
    LED_SYS_LOCKED   = GPIO_PIN_15
} Led_Id_t;

void Outputs_Init(void);
void Outputs_SetLed(Led_Id_t led, bool state);
void Outputs_ToggleLed(Led_Id_t led);
void Outputs_SetBuzzer(bool state);
void Outputs_BuzzerBeepNonBlocking(uint32_t intervalMs);

#endif /* INC_OUTPUTS_H_ */
```

Tệp `Core/Src/outputs.c`:
```c
#include "outputs.h"

#define BUZZER_PIN   GPIO_PIN_11
#define BUZZER_PORT  GPIOD
#define LED_PORT     GPIOD

static uint32_t lastBeepTime = 0;
static bool buzzerState = false;

void Outputs_Init(void)
{
    __HAL_RCC_GPIOD_CLK_ENABLE();
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    GPIO_InitStruct.Pin = GPIO_PIN_11 | GPIO_PIN_12 | GPIO_PIN_14 | GPIO_PIN_15;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);

    /* Ban đầu tắt tất cả ngõ ra */
    Outputs_SetLed(LED_ALARM_GREEN, false);
    Outputs_SetLed(LED_WRONG_CODE, false);
    Outputs_SetBuzzer(false);
}

void Outputs_SetLed(Led_Id_t led, bool state)
{
    HAL_GPIO_WritePin(LED_PORT, (uint16_t)led, state ? GPIO_PIN_SET : GPIO_PIN_RESET);
}

void Outputs_ToggleLed(Led_Id_t led)
{
    HAL_GPIO_TogglePin(LED_PORT, (uint16_t)led);
}

void Outputs_SetBuzzer(bool state)
{
    HAL_GPIO_WritePin(BUZZER_PORT, BUZZER_PIN, state ? GPIO_PIN_SET : GPIO_PIN_RESET);
}

void Outputs_BuzzerBeepNonBlocking(uint32_t intervalMs)
{
    if (intervalMs == 0) {
        Outputs_SetBuzzer(false);
        return;
    }
    uint32_t now = HAL_GetTick();
    if (now - lastBeepTime >= intervalMs) {
        lastBeepTime = now;
        buzzerState = !buzzerState;
        Outputs_SetBuzzer(buzzerState);
    }
}
```

---

### 5.4.3. Mô-đun Hệ Thống Báo Động Lõi (`alarm_system.h` & `alarm_system.c`)
Tệp `Core/Inc/alarm_system.h`:
```c
#ifndef INC_ALARM_SYSTEM_H_
#define INC_ALARM_SYSTEM_H_

#include <stdbool.h>

typedef enum {
    SYS_SAFE,
    SYS_ARMED,
    SYS_ALARM_TRIGGERED
} Alarm_State_t;

void AlarmSystem_Init(void);
void AlarmSystem_Process(void);
Alarm_State_t AlarmSystem_GetState(void);
void AlarmSystem_SetState(Alarm_State_t newState);
bool AlarmSystem_ValidatePIN(const char *pin);
void AlarmSystem_UpdatePIN(const char *newPin);

#endif /* INC_ALARM_SYSTEM_H_ */
```

Tệp `Core/Src/alarm_system.c`:
```c
#include "alarm_system.h"
#include "inputs.h"
#include "outputs.h"
#include <string.h>

/* Biến đóng gói static - Chỉ nội bộ tệp này truy cập được */
static Alarm_State_t currentState = SYS_SAFE;
static char activePIN[5] = "1234";

void AlarmSystem_Init(void)
{
    currentState = SYS_SAFE;
    Outputs_SetLed(LED_ALARM_GREEN, true);
    Outputs_SetLed(LED_WRONG_CODE, false);
}

Alarm_State_t AlarmSystem_GetState(void)
{
    return currentState;
}

void AlarmSystem_SetState(Alarm_State_t newState)
{
    currentState = newState;
    if (currentState == SYS_SAFE) {
        Outputs_SetLed(LED_ALARM_GREEN, true);
        Outputs_SetLed(LED_WRONG_CODE, false);
        Outputs_BuzzerSet(false);
    } else if (currentState == SYS_ARMED) {
        Outputs_SetLed(LED_ALARM_GREEN, false);
        Outputs_SetLed(LED_WRONG_CODE, true);
        Outputs_BuzzerSet(false);
    }
}

bool AlarmSystem_ValidatePIN(const char *pin)
{
    if (pin == NULL) return false;
    return (strcmp(activePIN, pin) == 0);
}

void AlarmSystem_UpdatePIN(const char *newPin)
{
    if (newPin != NULL && strlen(newPin) == 4) {
        strncpy(activePIN, newPin, 4);
        activePIN[4] = '\0';
    }
}

void AlarmSystem_Process(void)
{
    /* Đánh giá logic tự động */
    if (currentState == SYS_ARMED) {
        if (Inputs_IsGasDetected() && Inputs_IsOverTemp()) {
            currentState = SYS_ALARM_TRIGGERED;
        }
    }

    if (currentState == SYS_ALARM_TRIGGERED) {
        Outputs_BuzzerBeepNonBlocking(150);
        Outputs_ToggleLed(LED_WRONG_CODE);
    }
}
```

---

### 5.4.4. Tối Ưu Hóa Tệp `main.c` Chuẩn Kiến Trúc Sạch (Clean Main)
Toàn bộ tệp `Core/Src/main.c` chỉ còn lại vài chục dòng trong sáng tuyệt đối:

```c
#include "main.h"
#include "inputs.h"
#include "outputs.h"
#include "matrix_keypad.h"
#include "alarm_system.h"
#include "pc_serial_com.h"

int main(void)
{
    /* 1. Khởi tạo tầng phần cứng cơ sở */
    HAL_Init();
    SystemClock_Config();

    /* 2. Khởi tạo các phân hệ ứng dụng mô-đun */
    Inputs_Init();
    Outputs_Init();
    MatrixKeypad_Init();
    AlarmSystem_Init();
    PC_SerialCom_Init();

    /* 3. Siêu vòng lặp điều phối chính (Super Loop) */
    while (1)
    {
        AlarmSystem_Process();
        PC_SerialCom_Process();
    }
}
```

---

## 5.5. "UNDER THE HOOD" & CASE STUDY (THEO GIÁO TRÌNH ARM)

### 5.5.1. Under the Hood (5.3.1): Rủi Ro Của Biến Toàn Cục Không Kiểm Soát
- **Hiện tượng chạy đua dữ liệu (Data Race Condition):** Khi nhiều mô-đun hoặc một chương trình phục vụ ngắt (ISR) cùng ghi vào một biến toàn cục không được đóng gói:
  - Giá trị biến có thể bị ghi đè dở dang giữa chu trình tính toán, làm sập logic hệ thống mà không thể truy tìm dấu vết (Heisenbug).
- **Nguyên tắc đóng gói:** Luôn khai báo biến là `static` trong tệp `.c`, chỉ cung cấp các hàm Getter và Setter (`AlarmSystem_GetState()`, `AlarmSystem_SetState()`) để kiểm soát chặt chẽ điều kiện biên mỗi khi dữ liệu được thay đổi.

### 5.5.2. Case Study (5.4.1): Kiến Trúc Phần Mềm Nhúng Chuẩn AUTOSAR Trong Công Nghiệp Ô Tô
- Trong các xe ô tô hiện đại (Tesla, Mercedes), hàng trăm bộ điều khiển vi mạch ECU (Electronic Control Unit) giao tiếp với nhau.
- Hiệp hội các nhà sản xuất ô tô toàn cầu chuẩn hóa kiến trúc **AUTOSAR (Automotive Open System Architecture)**:
  - Phân tách tuyệt đối giữa phần cứng vi điều khiển (Tầng MCAL - Microcontroller Abstraction Layer) và phần mềm logic nghiệp vụ (Tầng Application Software Component).
  - Nhờ tính mô-đun hóa tuyệt đối, khi chuyển đổi dòng vi điều khiển từ STM32 sang NXP hay Infineon, toàn bộ phần mềm ứng dụng bên trên được tái sử dụng $100\%$ mà không cần viết lại một dòng code nào!

---

## 5.6. CÂU HỎI ÔN TẬP VÀ BÀI TẬP THIẾT KẾ CHƯƠNG 5

1. **Câu hỏi 1 (Kỹ thuật C):** *Từ khóa `static` đặt trước một biến toàn cục trong tệp `.c` có tác dụng gì đối với Bộ liên kết (Linker)? Tại sao kỹ thuật này giúp tránh được lỗi xung đột tên biến khi ghép nối code nhóm?*
2. **Câu hỏi 2 (Cơ chế biên dịch):** *Trình bày sự khác biệt giữa hai chỉ thị tiền xử lý: `#include <header.h>` và `#include "header.h"`. Trình biên dịch tìm kiếm các tệp này ở đâu?*
3. **Câu hỏi 3 (Tiêu chuẩn MISRA-C):** *Tại sao chuẩn MISRA-C nghiêm cấm việc sử dụng hàm cấp phát bộ nhớ động `malloc()` trong các hệ thống nhúng thời gian thực? Hãy nêu 2 rủi ro kỹ thuật cụ thể.*
4. **Bài tập tái cấu trúc phần mềm:** *Cho một đoạn mã nguồn điều khiển động cơ bước dài 300 dòng viết lẫn lộn trong `main.c`. Hãy thiết kế cấu trúc tệp tiêu đề `stepper_motor.h` và tệp thực thi `stepper_motor.c` với các hàm API chuẩn hóa để đóng gói phân hệ này.*
