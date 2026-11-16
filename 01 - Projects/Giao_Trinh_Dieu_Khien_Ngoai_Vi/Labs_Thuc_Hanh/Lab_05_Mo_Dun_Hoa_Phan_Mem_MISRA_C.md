# BÀI THỰC HÀNH SỐ 05: MÔ-ĐUN HÓA PHẦN MỀM NHÚNG VÀ TỐI ƯU KIẾN TRÚC MÃ NGUỒN CHUẨN CÔNG NGHIỆP
**Học phần:** Điều khiển Thiết bị Ngoại vi & Ghép nối Hệ thống Nhúng (STM32F407)  
**Thời lượng:** 3 tiết (135 phút) | **Mô hình:** Studio (Lý thuyết kiến trúc $\rightarrow$ Tái cấu trúc mã nguồn $\rightarrow$ Biên dịch $\rightarrow$ Đánh giá MISRA-C)  
**Chuẩn đầu ra học phần (CLO):** CLO 1 (Kiến trúc phân tầng), CLO 3 (Mô-đun hóa `.h`/`.c`), CLO 4 (Phần mềm nhúng chuẩn công nghiệp)  
**Căn cứ giáo trình:** *ARM Education - Chương 5: Software Modularity and Architecture*

---

## 1. MỤC TIÊU BÀI THÍ NGHIỆM
1. **Lý thuyết Kiến trúc Phần mềm Nhúng:**
   - Nắm vững nguyên tắc phân tầng (Layered Architecture): Tầng Ứng dụng (Application), Tầng Dịch vụ (Services/Middleware), Tầng Trình điều khiển Ngoại vi (Device Drivers), và Tầng Phần cứng (Hardware/BSP).
   - Hiểu rõ cơ chế của Trình tiền xử lý (Preprocessor), Trình biên dịch (Compiler) và Bộ liên kết (Linker) trong C.
   - Nắm vững quy tắc bảo vệ tệp tiêu đề (Header Guards `#ifndef ... #define ... #endif` hoặc `#pragma once`).
   - Kiểm soát phạm vi biến và hàm: Sử dụng từ khóa `static` để đóng gói biến cục bộ mô-đun (Private/Encapsulation) và `extern` để chia sẻ giao diện công khai (Public Interface).
2. **Kỹ thuật Lập trình:**
   - Tái cấu trúc (Refactoring) toàn bộ mã nguồn lộn xộn của các bài Lab 1, 2, 3, 4 thành các thư mục và cặp tệp riêng biệt:
     - `inputs.h` / `inputs.c`: Xử lý nút bấm, cảm biến Gas, cảm biến Nhiệt.
     - `outputs.h` / `outputs.c`: Điều khiển LED, Buzzer.
     - `matrix_keypad.h` / `matrix_keypad.c`: Quét và chống rung phím ma trận.
     - `alarm_system.h` / `alarm_system.c`: Quản lý logic máy trạng thái FSM lõi.
     - `pc_serial_com.h` / `pc_serial_com.c`: Giao tiếp UART với máy tính.
   - Tối giản hàm `main.c` thành vòng lặp sạch sẽ (Clean main) đạt chuẩn bảo trì công nghiệp.
3. **Phân tích Chất lượng Mã nguồn (MISRA-C):**
   - Áp dụng các quy tắc an toàn lập trình C trong ngành ô tô / y tế (MISRA-C:2012): Kiểu dữ liệu có kích thước cố định (`stdint.h`), loại bỏ `goto`, khởi tạo con trỏ an toàn, kiểm tra giới hạn mảng.

---

## 2. CẤU TRÚC THƯ MỤC DỰ ÁN CHUẨN CÔNG NGHIỆP
Sinh viên tổ chức cây thư mục trong STM32CubeIDE theo chuẩn sau:

```
Core/
├── Inc/
│   ├── main.h
│   ├── stm32f4xx_hal_conf.h
│   └── stm32f4xx_it.h
├── Src/
│   ├── main.c
│   ├── stm32f4xx_hal_msp.c
│   └── stm32f4xx_it.c
└── App/                    <-- Thư mục chứa các mô-đun ứng dụng tự viết
    ├── Inc/
    │   ├── inputs.h
    │   ├── outputs.h
    │   ├── matrix_keypad.h
    │   ├── alarm_system.h
    │   └── pc_serial_com.h
    └── Src/
        ├── inputs.c
        ├── outputs.c
        ├── matrix_keypad.c
        ├── alarm_system.c
        └── pc_serial_com.c
```

> [!TIP]
> Để STM32CubeIDE nhận diện thư mục tệp tiêu đề mới: Chuột phải vào Project $\rightarrow$ *Properties* $\rightarrow$ *C/C++ Build* $\rightarrow$ *Settings* $\rightarrow$ *Tool Settings* $\rightarrow$ *MCU GCC Compiler* $\rightarrow$ *Include paths* $\rightarrow$ Nhấn nút Add và thêm: `"${workspace_loc:/${ProjName}/Core/App/Inc}"`.

---

## 3. MÃ NGUỒN CÁC MÔ-ĐUN CHI TIẾT

### 3.1. Mô-đun Ngõ vào (`inputs.h` & `inputs.c`)
Tệp `Core/App/Inc/inputs.h`:
```c
#ifndef APP_INC_INPUTS_H_
#define APP_INC_INPUTS_H_

#include "stm32f4xx_hal.h"
#include <stdbool.h>

void Inputs_Init(void);
bool Inputs_IsGasDetected(void);
bool Inputs_IsOverTemp(void);
bool Inputs_IsButtonAPressed(void);
bool Inputs_IsButtonBPressed(void);

#endif /* APP_INC_INPUTS_H_ */
```

Tệp `Core/App/Src/inputs.c`:
```c
#include "inputs.h"

#define GAS_PIN         GPIO_PIN_2
#define TEMP_PIN        GPIO_PIN_3
#define BTN_A_PIN       GPIO_PIN_4
#define BTN_B_PIN       GPIO_PIN_5
#define INPUT_PORT      GPIOE

void Inputs_Init(void) {
    /* Khởi tạo phần cứng đã được cấu hình trong MX_GPIO_Init */
}

bool Inputs_IsGasDetected(void) {
    return (HAL_GPIO_ReadPin(INPUT_PORT, GAS_PIN) == GPIO_PIN_SET);
}

bool Inputs_IsOverTemp(void) {
    return (HAL_GPIO_ReadPin(INPUT_PORT, TEMP_PIN) == GPIO_PIN_SET);
}

bool Inputs_IsButtonAPressed(void) {
    return (HAL_GPIO_ReadPin(INPUT_PORT, BTN_A_PIN) == GPIO_PIN_SET);
}

bool Inputs_IsButtonBPressed(void) {
    return (HAL_GPIO_ReadPin(INPUT_PORT, BTN_B_PIN) == GPIO_PIN_SET);
}
```

---

### 3.2. Mô-đun Ngõ ra (`outputs.h` & `outputs.c`)
Tệp `Core/App/Inc/outputs.h`:
```c
#ifndef APP_INC_OUTPUTS_H_
#define APP_INC_OUTPUTS_H_

#include "stm32f4xx_hal.h"
#include <stdbool.h>

typedef enum {
    LED_GREEN = GPIO_PIN_12,
    LED_ORANGE = GPIO_PIN_13,
    LED_RED = GPIO_PIN_14,
    LED_BLUE = GPIO_PIN_15
} Led_Id_t;

void Outputs_Init(void);
void Outputs_LedSet(Led_Id_t led, bool state);
void Outputs_LedToggle(Led_Id_t led);
void Outputs_BuzzerSet(bool state);
void Outputs_BuzzerBeepNonBlocking(uint32_t intervalMs);

#endif /* APP_INC_OUTPUTS_H_ */
```

Tệp `Core/App/Src/outputs.c`:
```c
#include "outputs.h"

#define BUZZER_PIN      GPIO_PIN_11
#define BUZZER_PORT     GPIOD
#define LED_PORT        GPIOD

static uint32_t lastBeepTime = 0;
static bool buzzerState = false;

void Outputs_Init(void) {
    Outputs_LedSet(LED_GREEN, false);
    Outputs_LedSet(LED_RED, false);
    Outputs_BuzzerSet(false);
}

void Outputs_LedSet(Led_Id_t led, bool state) {
    HAL_GPIO_WritePin(LED_PORT, (uint16_t)led, state ? GPIO_PIN_SET : GPIO_PIN_RESET);
}

void Outputs_LedToggle(Led_Id_t led) {
    HAL_GPIO_TogglePin(LED_PORT, (uint16_t)led);
}

void Outputs_BuzzerSet(bool state) {
    HAL_GPIO_WritePin(BUZZER_PORT, BUZZER_PIN, state ? GPIO_PIN_SET : GPIO_PIN_RESET);
}

void Outputs_BuzzerBeepNonBlocking(uint32_t intervalMs) {
    if (intervalMs == 0) {
        Outputs_BuzzerSet(false);
        return;
    }
    uint32_t now = HAL_GetTick();
    if (now - lastBeepTime >= intervalMs) {
        lastBeepTime = now;
        buzzerState = !buzzerState;
        Outputs_BuzzerSet(buzzerState);
    }
}
```

---

### 3.3. Mô-đun Hệ thống Báo động Lõi (`alarm_system.h` & `alarm_system.c`)
Tệp `Core/App/Inc/alarm_system.h`:
```c
#ifndef APP_INC_ALARM_SYSTEM_H_
#define APP_INC_ALARM_SYSTEM_H_

#include <stdint.h>
#include <stdbool.h>

typedef enum {
    ALARM_STATE_SAFE,
    ALARM_STATE_ARMED,
    ALARM_STATE_TRIGGERED
} Alarm_State_t;

void AlarmSystem_Init(void);
void AlarmSystem_Process(void);
Alarm_State_t AlarmSystem_GetState(void);
void AlarmSystem_SetState(Alarm_State_t newState);
bool AlarmSystem_ValidatePIN(const char* enteredPIN);
void AlarmSystem_UpdatePIN(const char* newPIN);

#endif /* APP_INC_ALARM_SYSTEM_H_ */
```

Tệp `Core/App/Src/alarm_system.c`:
```c
#include "alarm_system.h"
#include "inputs.h"
#include "outputs.h"
#include <string.h>

static Alarm_State_t currentState = ALARM_STATE_SAFE;
static char currentPIN[5] = "1234";

void AlarmSystem_Init(void) {
    currentState = ALARM_STATE_SAFE;
    Outputs_LedSet(LED_GREEN, true);
    Outputs_LedSet(LED_RED, false);
}

Alarm_State_t AlarmSystem_GetState(void) {
    return currentState;
}

void AlarmSystem_SetState(Alarm_State_t newState) {
    currentState = newState;
    if (currentState == ALARM_STATE_SAFE) {
        Outputs_LedSet(LED_GREEN, true);
        Outputs_LedSet(LED_RED, false);
        Outputs_BuzzerSet(false);
    } else if (currentState == ALARM_STATE_ARMED) {
        Outputs_LedSet(LED_GREEN, false);
        Outputs_LedSet(LED_RED, true);
        Outputs_BuzzerSet(false);
    } else if (currentState == ALARM_STATE_TRIGGERED) {
        Outputs_LedSet(LED_GREEN, false);
        Outputs_LedSet(LED_RED, true);
    }
}

bool AlarmSystem_ValidatePIN(const char* enteredPIN) {
    if (enteredPIN == NULL) return false;
    return (strcmp(currentPIN, enteredPIN) == 0);
}

void AlarmSystem_UpdatePIN(const char* newPIN) {
    if (newPIN != NULL && strlen(newPIN) == 4) {
        strncpy(currentPIN, newPIN, 4);
        currentPIN[4] = '\0';
    }
}

void AlarmSystem_Process(void) {
    /* Đánh giá logic tự động dựa trên trạng thái FSM */
    if (currentState == ALARM_STATE_ARMED) {
        if (Inputs_IsGasDetected() && Inputs_IsOverTemp()) {
            AlarmSystem_SetState(ALARM_STATE_TRIGGERED);
        }
    }
    
    if (currentState == ALARM_STATE_TRIGGERED) {
        /* Báo động dồn dập 150ms */
        Outputs_BuzzerBeepNonBlocking(150);
        Outputs_LedToggle(LED_RED);
    }
}
```

---

### 3.4. Tối ưu hàm `main.c` Chuẩn Công nghiệp (Clean Architecture)
Toàn bộ file `Core/Src/main.c` chỉ còn lại:

```c
#include "main.h"
#include "inputs.h"
#include "outputs.h"
#include "matrix_keypad.h"
#include "alarm_system.h"
#include "pc_serial_com.h"

/* Các hàm nguyên mẫu của CubeMX */
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART2_UART_Init(void);

int main(void)
{
    /* 1. Khởi tạo tầng HAL và Xung nhịp */
    HAL_Init();
    SystemClock_Config();

    /* 2. Khởi tạo Driver phần cứng */
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    /* 3. Khởi tạo tầng dịch vụ ứng dụng */
    Inputs_Init();
    Outputs_Init();
    AlarmSystem_Init();
    PC_SerialCom_Init();

    /* 4. Vòng lặp điều phối chính - Siêu vòng lặp (Super Loop) */
    while (1)
    {
        AlarmSystem_Process();
        PC_SerialCom_Process();
    }
}
```

---

## 4. ĐÁNH GIÁ CHẤT LƯỢNG MÃ NGUỒN VÀ BẢO TRÌ

### 4.1. Lợi ích vượt trội so với mã nguồn không mô-đun
| Tiêu chí | Mã nguồn cũ (Tất cả trong main.c) | Mã nguồn Mô-đun hóa chuẩn |
| :--- | :--- | :--- |
| **Số dòng trong `main.c`** | > 400 dòng, đan xen lẫn lộn | < 50 dòng, trong sáng tuyệt đối |
| **Phụ thuộc phần cứng** | Sửa chân pin phải sửa toàn bộ code | Chỉ cần sửa định nghĩa chân trong `inputs.c`/`outputs.c` |
| **Làm việc nhóm (Teamwork)** | Rất dễ xung đột Git Merge Conflict | Mỗi lập trình viên code trên một module độc lập |
| **Tái sử dụng (Reusability)** | Không thể tái sử dụng sang dự án khác | Tái sử dụng trọn vẹn thư viện sang vi điều khiển khác |

### 4.2. Bài tập thực hành tuân thủ MISRA-C
1. Kiểm tra tất cả các kiểu biến số: Thay thế toàn bộ `int`, `unsigned char`, `long` bằng chuẩn cố định `int32_t`, `uint8_t`, `uint32_t` từ `<stdint.h>`.
2. Kiểm tra biến con trỏ: Mọi hàm nhận con trỏ chuỗi (`const char*`) đều phải kiểm tra `if (ptr == NULL)` trước khi truy cập nhằm ngăn chặn lỗi rò rỉ vùng nhớ / HardFault.

---

## 5. BÁO CÁO KẾT QUẢ VÀ CÂU HỎI ĐÁNH GIÁ (RUBRIC)

### 5.1. Báo cáo thí nghiệm
- Nộp thư mục mã nguồn hoàn chỉnh biên dịch không có lỗi và không có cảnh báo (0 Errors, 0 Warnings).
- Bảng phân tích kích thước bộ nhớ (Flash Memory & RAM consumption) do Compiler thông báo.

### 5.2. Câu hỏi đánh giá
1. *Từ khóa `static` đặt trước một biến toàn cục trong file `.c` có tác dụng gì đối với Linker?*
2. *Nếu không có `#ifndef ... #define ... #endif` (Header Guards), trình biên dịch sẽ báo lỗi gì khi hai file `.c` cùng `#include` một file `.h`?*
3. *Tại sao các chuẩn an toàn như MISRA-C nghiêm cấm việc sử dụng hàm cấp phát bộ nhớ động (`malloc()` / `free()`) trong hệ thống nhúng thời gian thực?*