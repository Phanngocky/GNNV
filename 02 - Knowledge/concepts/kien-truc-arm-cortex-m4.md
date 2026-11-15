---
type: concept
title: "Kiến trúc lõi ARM Cortex-M4 và Bus Matrix trong STM32F4"
sources: 1
tags: [cortex-m4, stm32f4, bus-matrix, ahb-apb, memory-map]
updated: 2026-11-15
---

# Kiến trúc lõi ARM Cortex-M4 và Bus Matrix trong STM32F4

## 1. Tổng quan lõi Cortex-M4
- **Kiến trúc:** ARMv7E-M, tập lệnh Thumb-2 (hiệu năng cao, tiết kiệm bộ nhớ).
- **Khối xử lý tín hiệu số (DSP) & FPU:** Tích hợp phần cứng tính toán dấu phẩy động đơn chính xác (Single Precision Floating Point Unit) và tập lệnh SIMD.
- **Tần số hoạt động:** Lên đến 84 MHz (STM32F401), 168 MHz (STM32F407) hoặc 180 MHz (STM32F429).
- **Bộ điều khiển ngắt NVIC:** Hỗ trợ tới 240 ngắt ngoài với mức độ ưu tiên có thể cấu hình (Preemption & Sub-priority).

## 2. Hệ thống Bus Matrix & Phân bổ Ngoại vi
Hệ thống kết nối đa lớp (Multi-AHB Bus Matrix) cho phép CPU và bộ điều khiển DMA truy cập đồng thời vào các khối ngoại vi và bộ nhớ khác nhau mà không bị xung đột bus:
- **I-Bus (Instruction Bus):** Tải lệnh từ bộ nhớ Flash.
- **D-Bus (Data Bus):** Đọc/ghi dữ liệu từ Flash (hằng số, bảng tra).
- **S-Bus (System Bus):** Truy cập SRAM và các thanh ghi ngoại vi.
- **AHB1 / AHB2 (Advanced High-performance Bus):** Kết nối GPIO (Port A -> K), DMA, Ethernet, USB OTG, CRC. Tốc độ xung nhịp cao (bằng tần số CPU `HCLK`).
- **APB1 (Advanced Peripheral Bus 1 - Tốc độ thấp):** Giới hạn tần số tối đa (ví dụ 42 MHz hoặc 45 MHz). Chứa USART2/3, UART4/5, SPI2/3, I2C1/2/3, TIM2/3/4/5/6/7/12/13/14, DAC, RTC, CAN1/CAN2.
- **APB2 (Advanced Peripheral Bus 2 - Tốc độ cao):** Tần số xung nhịp lên đến 84 MHz hoặc 90 MHz. Chứa USART1, USART6, SPI1, SPI4, TIM1, TIM8, TIM9, TIM10, TIM11, ADC1/2/3, EXTI, SYSCFG.

## 3. Không gian địa chỉ (Memory Mapping)
- Toàn bộ không gian 4GB (32-bit) được phân chia cố định:
  - `0x0800 0000`: Flash memory.
  - `0x2000 0000`: SRAM nội.
  - `0x4000 0000`: Vùng nhớ ngoại vi (Peripheral Memory Map) chia thành APB1, APB2, AHB1, AHB2.
  - `0xE000 0000`: Không gian lõi Cortex-M (NVIC, SysTick, MPU, SCB).

## 4. Ý nghĩa sư phạm trong giảng dạy ngoại vi
- Sinh viên cần nắm được: Trước khi giao tiếp với bất kỳ thiết bị ngoại vi nào, **bắt buộc phải cấp xung clock** cho ngoại vi đó thông qua thanh ghi `RCC` (Reset and Clock Control).
- Hiểu được sự khác biệt giữa bus APB1 và APB2 để tính toán chính xác tần số làm việc của Timer, Baudrate của UART và xung nhịp SPI/I2C.

Liên kết liên quan:
- [[dieu-khien-gpio]]
- [[he-thong-ngat-nvic-exti]]
- [[timer-counter-pwm]]
