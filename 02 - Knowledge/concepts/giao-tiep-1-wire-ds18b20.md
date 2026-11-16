---
type: concept
title: "Chuẩn giao tiếp 1-Wire và Cảm biến nhiệt độ số DS18B20 trên STM32F407"
sources: 1
tags: [1-wire, ds18b20, stm32f407, microsecond-timing, open-drain]
updated: 2026-11-16
---

# Chuẩn Giao Tiếp 1-Wire và Cảm Biến Nhiệt Độ Số DS18B20

## 1. Bản chất Vật lý Chuẩn 1-Wire
- Do Dallas Semiconductor (nay là Maxim Integrated / Analog Devices) phát minh.
- Sử dụng **duy nhất 1 đường dây dữ liệu ($DQ$)** kết hợp điện trở kéo lên ngoài $R_{pullup} = 4.7\text{ k}\Omega$.
- Tầng vật lý: Cực máng hở (Open-Drain). Master và Slave kéo đường dây về mass ($0\text{V}$) để truyền tín hiệu hoặc thả nổi (Hi-Z) để điện trở kéo lên mức $3.3\text{V}$.

## 2. Giản đồ Định thời Micro-giây ($\mu\text{s}$) Cốt lõi
- **Reset Pulse:** Master kéo thấp $\ge 480\mu\text{s}$, sau đó thả nổi.
- **Presence Pulse:** Sau $15 \div 60\mu\text{s}$, cảm biến DS18B20 kéo thấp $60 \div 240\mu\text{s}$ để báo hiện diện.
- **Write 0 Slot:** Master kéo thấp $60 \div 120\mu\text{s}$.
- **Write 1 Slot:** Master kéo thấp $1 \div 15\mu\text{s}$ rồi thả nổi.
- **Read Slot:** Master kéo thấp $1 \div 3\mu\text{s}$, thả nổi, lấy mẫu trạng thái bus tại $12 \div 15\mu\text{s}$.

## 3. Đặc tính Kỹ thuật DS18B20
- Dải đo: $-55^\circ\text{C} \div +125^\circ\text{C}$, sai số $\pm 0.5^\circ\text{C}$.
- Độ phân giải: 9 đến 12 bit (12-bit có bước nhảy $0.0625^\circ\text{C}$, thời gian chuyển đổi tối đa $750\text{ ms}$).
- Mã định danh ROM 64-bit duy nhất: Cho phép ghép song song nhiều cảm biến trên cùng một đường dây.
- Tập lệnh cơ bản: Skip ROM (`0xCC`), Convert T (`0x44`), Read Scratchpad (`0xBE`).

## 4. Ứng dụng trên STM32F407
- Chân cấu hình: `PB12` (GPIO Output Open-Drain).
- Tạo trễ $\mu\text{s}$: Sử dụng bộ đếm chu kỳ Cortex-M4 DWT (`DWT->CYCCNT`) tại tần số $168\text{ MHz}$ (168 xung clock / $1\mu\text{s}$).

Liên kết liên quan:
- [[chuong-06-man-hinh-lcd-bus-i2c-va-spi]]
- [[truyen-thong-i2c]]
- [[truyen-thong-spi]]
