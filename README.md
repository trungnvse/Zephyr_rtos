# 🦅 Introduction to Zephyr RTOS — Tổng quan Khóa học (Bài 1–6)

> **Series:** Introduction to Zephyr RTOS (12 phần)  
> **Tác giả:** [ShawnHymel](https://github.com/ShawnHymel/introduction-to-zephyr)  
> **Board:** ESP32-S3-DevKitC | **Zephyr version:** 3.7  

---

## 📑 Mục lục

- [Bài 1 — Giới thiệu & Cài đặt](#-bài-1--giới-thiệu--cài-đặt)
- [Bài 2 — CMake](#-bài-2--cmake)
- [Bài 3 — Kconfig](#-bài-3--kconfig)
- [Bài 4 — Devicetree](#-bài-4--devicetree)
- [Bài 5 — Devicetree Bindings](#-bài-5--devicetree-bindings)
- [Bài 6 — Device Driver](#-bài-6--device-driver)
- [Bức tranh toàn cảnh](#-bức-tranh-toàn-cảnh)
- [Yêu cầu kiến thức](#-yêu-cầu-kiến-thức)
- [Phần cứng cần chuẩn bị](#-phần-cứng-cần-chuẩn-bị)

---

## 🟢 Bài 1 — Giới thiệu & Cài đặt

> *Getting Started: Installation and Blink*

### Nội dung chính
Bài mở đầu giới thiệu Zephyr là gì, tại sao nên dùng nó thay vì các RTOS khác, và cách thiết lập toàn bộ môi trường phát triển từ đầu.

### Kiến thức thu được
- Hiểu Zephyr RTOS là gì và vị trí của nó trong thế giới embedded
- So sánh Zephyr với FreeRTOS, NuttX, VxWorks
- Cài đặt môi trường phát triển bằng **Docker** (cross-platform)
- Hiểu vai trò của **Python venv** trong việc quản lý dependencies
- Làm quen với công cụ **west** — meta-tool của Zephyr
- Build và flash chương trình LED Blink đầu tiên lên ESP32-S3
- Dùng **esptool** để nạp firmware và **miniterm** để xem Serial output

### Lệnh quan trọng
```bash
# Build project
west build -p always -b esp32s3_devkitc/esp32s3/procpu \
  -- -DDTC_OVERLAY_FILE=boards/esp32s3_devkitc.overlay

# Flash firmware
python -m esptool --port "<PORT>" --chip auto --baud 921600 \
  write_flash --flash_size detect 0x0 build/zephyr/zephyr.bin

# Serial monitor
python -m serial.tools.miniterm "<PORT>" 115200
```

### Kết quả đạt được
✅ LED nhấp nháy trên ESP32-S3 với firmware viết bằng Zephyr

---

## 🟡 Bài 2 — CMake

> *CMake Tutorial*

### Nội dung chính
CMake là hệ thống build được Zephyr sử dụng. Bài này đi sâu vào CMake độc lập trước khi kết hợp với Zephyr, giúp bạn hiểu "phía sau hậu trường" khi gọi `west build`.

### Kiến thức thu được
- Hiểu CMake là **meta build system** — không build trực tiếp mà tạo ra Makefile/Ninja files
- Viết file `CMakeLists.txt` từ đầu: `cmake_minimum_required`, `project`, `add_library`, `add_executable`, `target_link_libraries`
- Phân biệt `PUBLIC` vs `PRIVATE` trong `target_include_directories` và `target_link_libraries`
- Sử dụng nhiều **generator** khác nhau: Unix Makefiles, Ninja
- Tổ chức project lớn với **nested CMakeLists.txt** và `add_subdirectory()`
- Cách Zephyr tích hợp CMake: `find_package(Zephyr)`, target `app`, `target_sources()`

### Cấu trúc CMakeLists.txt cơ bản trong Zephyr
```cmake
cmake_minimum_required(VERSION 3.20.0)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_app)
target_sources(app PRIVATE src/main.c)
```

### Challenge
Tạo thư viện riêng (`say_hello`) và tích hợp vào ví dụ Blink từ Bài 1.

---

## 🟠 Bài 3 — Kconfig

> *Kconfig Tutorial*

### Nội dung chính
Kconfig là hệ thống cấu hình giúp bật/tắt tính năng trong firmware mà **không cần sửa code**. Bài này dạy cách dùng Kconfig để kiểm soát những gì được biên dịch vào firmware.

### Kiến thức thu được
- Hiểu **tại sao Kconfig tồn tại**: Zephyr hỗ trợ hàng trăm board và tính năng — không thể bật tất cả mọi lúc
- Dùng **menuconfig** (giao diện TUI) để khám phá và thay đổi cấu hình
- Hiểu file `.config` được sinh ra ở đâu và khi nào bị ghi đè
- Lưu cấu hình vào `prj.conf` để tái sử dụng
- Dùng **board-specific config** (`boards/esp32s3_devkitc.conf`) cho cấu hình riêng từng board
- **Tự định nghĩa Kconfig symbol** trong module tùy chỉnh

### Ví dụ: Định nghĩa Kconfig symbol riêng
```kconfig
config SAY_HELLO
    bool "Basic print test to console"
    default n
    depends on PRINTK
    help
        Adds say_hello() function to print a basic message.
```

### Sử dụng trong code C
```c
#ifdef CONFIG_SAY_HELLO
#include "say_hello.h"
#endif

// Trong main:
#ifdef CONFIG_SAY_HELLO
    say_hello();
#endif
```

### Challenge
Kích hoạt hỗ trợ **in số thực (float)** qua Kconfig (`CONFIG_NEWLIB_LIBC_FLOAT_PRINTF` hoặc `CONFIG_PICOLIBC_IO_FLOAT`) để in được random number dạng `0.731`.

---

## 🔵 Bài 4 — Devicetree

> *Devicetree Tutorial*

### Nội dung chính
Devicetree là cách Zephyr **mô tả phần cứng** bằng file văn bản, tách biệt hoàn toàn khỏi code. Đây là một trong những khái niệm khó nhất nhưng cũng quan trọng nhất của Zephyr.

### Kiến thức thu được
- Lịch sử Devicetree: từ Sun Microsystems (1988) → Linux kernel → Zephyr
- Sự khác biệt giữa Linux và Zephyr: Linux dùng DT **lúc runtime**, Zephyr dùng DT **lúc compile time**
- Cú pháp DTS: nodes, properties, labels, phandles, unit address
- Phân biệt **`.dts`** (board definition), **`.dtsi`** (shared/SoC definition), **`.overlay`** (user customization)
- Hiểu **aliases** và **chosen nodes** — cách viết code không phụ thuộc vào tên node cụ thể
- Viết **overlay file** để thêm/ghi đè cấu hình phần cứng mà không sửa board DTS gốc
- Đọc GPIO button qua Devicetree API: `GPIO_DT_SPEC_GET`, `DT_ALIAS`

### Ví dụ overlay file
```dts
/ {
    aliases {
        my-button = &button_1;
    };

    buttons {
        compatible = "gpio-keys";
        debounce-interval-ms = <50>;
        button_1: d5 {
            gpios = <&gpio0 5 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
        };
    };
};
```

### Đọc button trong code C
```c
static const struct gpio_dt_spec btn =
    GPIO_DT_SPEC_GET(DT_ALIAS(my_button), gpios);

gpio_pin_configure_dt(&btn, GPIO_INPUT);
int state = gpio_pin_get_dt(&btn);
```

### Challenge
Kết hợp button từ bài này với LED từ Bài 1: nhấn button để bật/tắt LED.

---

## 🟣 Bài 5 — Devicetree Bindings

> *Devicetree Bindings*

### Nội dung chính
Bindings là **cầu nối** giữa Devicetree (mô tả phần cứng) và driver code (xử lý phần cứng). Chúng là các file YAML định nghĩa "giao diện" mà một Devicetree node phải tuân theo.

### Kiến thức thu được
- Hiểu bindings là gì và tại sao chúng cần thiết
- Vị trí của bindings files: `zephyr/dts/bindings/`
- Cách đọc file YAML bindings để biết node cần những property gì
- Bindings **validate** DTS khi build — sai property sẽ báo lỗi ngay
- Bindings tự động **sinh ra macro C** từ DTS properties (không cần hardcode)
- Thực hành: cấu hình và đọc **ADC** (biến trở) trên ESP32-S3
- Dùng macro `DEVICE_DT_GET`, `ADC_CHANNEL_CFG_DT`, `DT_PROP` để truy cập DT data trong code

### Ví dụ overlay cho ADC
```dts
&adc0 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    adc0_ch0: channel@0 {
        reg = <0>;
        zephyr,gain = "ADC_GAIN_1_4";
        zephyr,reference = "ADC_REF_INTERNAL";
        zephyr,vref-mv = <3894>;
        zephyr,resolution = <12>;
    };
};
```

### Truy cập ADC trong code C
```c
static const struct device *adc = DEVICE_DT_GET(DT_ALIAS(my_adc));
static const struct adc_channel_cfg adc_ch =
    ADC_CHANNEL_CFG_DT(DT_ALIAS(my_adc_channel));

// Đọc ADC — không có địa chỉ hardcode nào!
adc_read(adc, &seq);
```

### Challenge
Điều khiển độ sáng LED bằng PWM dựa trên giá trị đọc từ biến trở (ADC).

---

## 🔴 Bài 6 — Device Driver

> *Device Driver Development*

### Nội dung chính
Bài đỉnh cao của nửa đầu khóa học — tự viết một **device driver hoàn chỉnh** từ đầu cho nút bấm (button), kết hợp tất cả những gì đã học: CMake, Kconfig, Devicetree, và Bindings.

### Kiến thức thu được
- Hiểu **device driver model** của Zephyr (tương tự Linux)
- Cấu trúc một **out-of-tree module** trong Zephyr
- Cách `DT_DRV_COMPAT` liên kết driver với `compatible` string trong DTS
- Macro `DT_INST_FOREACH_STATUS_OKAY` — tự động tạo driver instance cho **mỗi node** trong DTS
- Viết **bindings YAML** cho driver tự tạo (`custom,button.yaml`)
- Đăng ký driver với Zephyr kernel qua `DEVICE_DT_INST_DEFINE`
- Expose public API cho application dùng (`struct button_api`)

### Cấu trúc module driver

```
workspace/modules/button/
├── drivers/button/
│   ├── button.c        ← Logic driver chính
│   ├── button.h        ← Public API
│   ├── CMakeLists.txt  ← Build config
│   └── Kconfig         ← Enable/disable symbol
├── dts/bindings/input/
│   └── custom,button.yaml  ← Bindings file
└── zephyr/module.yaml  ← Đăng ký là Zephyr module
```

### "Phép thuật" macro trong driver

```c
// Khai báo DT_DRV_COMPAT để liên kết với "custom,button" trong DTS
#define DT_DRV_COMPAT custom_button

// Macro này tự động tạo 1 driver instance cho mỗi button node trong DTS
DT_INST_FOREACH_STATUS_OKAY(BUTTON_DEFINE)
```

→ Nếu DTS có 2 button node → tự động tạo 2 driver instances, không cần viết tay!

### Challenge
Hoàn thành workshop tự viết **I2C driver cho cảm biến nhiệt độ MCP9808**.
Xem: [workshop-zephyr-device-driver](https://github.com/ShawnHymel/workshop-zephyr-device-driver)

---

## 🗺️ Bức tranh toàn cảnh

Sau 6 bài, bạn đã hiểu cách **tất cả các lớp trong Zephyr kết hợp với nhau**:

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION (main.c)                     │
│         Dùng Devicetree macros, không hardcode gì           │
├─────────────────────────────────────────────────────────────┤
│                  DEVICE DRIVER (Bài 6)                      │
│     button.c — abstract hóa phần cứng cho app dùng         │
├──────────────────────┬──────────────────────────────────────┤
│   KCONFIG (Bài 3)    │       DEVICETREE (Bài 4 & 5)        │
│  Bật/tắt tính năng   │  Mô tả phần cứng + Bindings YAML    │
├──────────────────────┴──────────────────────────────────────┤
│                    CMAKE (Bài 2)                            │
│           Quản lý build, link libraries                     │
├─────────────────────────────────────────────────────────────┤
│             ZEPHYR RTOS CORE + TOOLCHAIN (Bài 1)            │
│         west, Docker, ESP32-S3 board support                │
├─────────────────────────────────────────────────────────────┤
│                      PHẦN CỨNG                              │
│              ESP32-S3 · GPIO · ADC · I2C...                 │
└─────────────────────────────────────────────────────────────┘
```

### Mối quan hệ giữa các thành phần

```
prj.conf ──────────────→ Kconfig
                              │
                              ↓ bật/tắt driver
CMakeLists.txt ────────→ CMake ──→ Build system
                              │
                              ↓ compile driver code
.overlay + .dts ───────→ Devicetree
                              │
                              ↓ generate macros
bindings YAML ─────────→ Bindings
                              │
                              ↓ validate + generate C headers
                         Application code sử dụng
                         DT_ALIAS(), DT_PROP()...
```

---

## 📋 Yêu cầu kiến thức

Trước khi bắt đầu khóa học, bạn nên biết:

| Kiến thức | Mức độ cần thiết |
|---|---|
| Lập trình C (pointer, struct, callback, interrupt) | ⭐⭐⭐ Bắt buộc |
| Vi điều khiển cơ bản (GPIO, UART, SPI, I2C, Timer) | ⭐⭐⭐ Bắt buộc |
| Multithreading (thread, mutex, semaphore, queue) | ⭐⭐ Quan trọng |
| Linux command line cơ bản | ⭐⭐ Quan trọng |
| CMake / build system | ⭐ Có thì tốt (Bài 2 sẽ dạy) |
| Linux kernel / device driver | ⭐ Có thì tốt |

> 💡 Nếu bạn đã hoàn thành **Introduction to RTOS** (series trước của ShawnHymel), bạn đã sẵn sàng!

---

## 🔧 Phần cứng cần chuẩn bị

| Linh kiện | Dùng từ bài |
|---|---|
| ESP32-S3-DevKitC | Bài 1 → hết khóa |
| LED + điện trở 220Ω | Bài 1 |
| 2× Nút bấm (pushbutton) | Bài 4, 6 |
| Biến trở 10kΩ (trimpot) | Bài 5 |
| Cảm biến nhiệt MCP9808 (I2C) | Bài 6 (challenge) |
| Màn hình TFT 0.96" ST7735R | Bài sau |
| Dây jumper + breadboard | Bài 1 → hết khóa |
| Cáp USB-A to micro-B | Bài 1 → hết khóa |

---

## 📚 Tài liệu tham khảo

| Tài liệu | Link |
|---|---|
| Repository khóa học | [ShawnHymel/introduction-to-zephyr](https://github.com/ShawnHymel/introduction-to-zephyr) |
| Zephyr official docs | [docs.zephyrproject.org](https://docs.zephyrproject.org/latest/) |
| Zephyr Getting Started | [Getting Started Guide](https://docs.zephyrproject.org/latest/develop/getting_started/index.html) |
| Kconfig documentation | [Zephyr Kconfig](https://docs.zephyrproject.org/latest/build/kconfig/index.html) |
| Devicetree documentation | [Zephyr Devicetree](https://docs.zephyrproject.org/latest/build/dts/index.html) |
| Device Driver Model | [Zephyr Driver Model](https://docs.zephyrproject.org/latest/kernel/drivers/index.html) |
| west documentation | [west docs](https://docs.zephyrproject.org/latest/develop/west/index.html) |
| esptool | [esptool docs](https://docs.espressif.com/projects/esptool/en/latest/) |
| Practical Zephyr (blog series) | [Kconfig Part 2](https://interrupt.memfault.com/blog/practical-zephyr-kconfig) |

---

> ✍️ *Ghi chú học tập cá nhân dựa trên khóa học của ShawnHymel, bổ sung thêm giải thích và phân tích chi tiết.*  
> ⚠️ *Sử dụng Zephyr **3.7** — version 4.0+ có thể có thay đổi về API.*
