# 🚀 Introduction to Zephyr RTOS — Bài 1: Cài đặt Môi trường & LED Blink

> **Khóa học:** Introduction to Zephyr RTOS (12 phần)  
> **Tác giả gốc:** [ShawnHymel](https://github.com/ShawnHymel/introduction-to-zephyr)
> **Board:** ESP32-S3-DevKitC  
> **Zephyr version:** 3.7  

---

## 📑 Mục lục

- [1. Zephyr RTOS là gì?](#1-zephyr-rtos-là-gì)
- [2. Tại sao cần RTOS?](#2-tại-sao-cần-rtos)
- [3. So sánh Zephyr với các RTOS khác](#3-so-sánh-zephyr-với-các-rtos-khác)
- [4. Hệ sinh thái Zephyr](#4-hệ-sinh-thái-zephyr)
- [5. Phần cứng cần chuẩn bị](#5-phần-cứng-cần-chuẩn-bị)
- [6. Python venv là gì và tại sao cần dùng?](#6-python-venv-là-gì-và-tại-sao-cần-dùng)
- [7. Cài đặt môi trường với Docker](#7-cài-đặt-môi-trường-với-docker)
- [8. Công cụ west — Trái tim của Zephyr](#8-công-cụ-west--trái-tim-của-zephyr)
- [9. Build ví dụ Blink](#9-build-ví-dụ-blink)
- [10. esptool — Nạp firmware lên ESP32](#10-esptool--nạp-firmware-lên-esp32)
- [11. Xem output qua Serial Monitor](#11-xem-output-qua-serial-monitor)
- [12. Tóm tắt luồng làm việc](#12-tóm-tắt-luồng-làm-việc)
- [13. Bảng tra cứu nhanh](#13-bảng-tra-cứu-nhanh)

---

## 1. Zephyr RTOS là gì?

**Zephyr** là một **hệ điều hành thời gian thực (Real-Time Operating System — RTOS)** mã nguồn mở, được thiết kế cho các thiết bị nhúng nhỏ và IoT.

> 💡 **Hãy hình dung như sau:**  
> Viết firmware không có RTOS giống như bạn tự làm mọi thứ trong bếp: tự đun nước, tự thái rau, tự chiên xào đồng thời — một mình, tuần tự từng bước.  
> Còn **Zephyr** giống như một nhà bếp chuyên nghiệp có nhiều đầu bếp (thread), mỗi người phụ trách một việc, có quản lý bếp trưởng (scheduler) điều phối tất cả cùng lúc.

Zephyr **không chỉ là một RTOS đơn thuần** — nó là cả một **hệ sinh thái** bao gồm:

- ✅ Scheduler & multithreading
- ✅ Bluetooth, WiFi, HTTP stacks
- ✅ Hệ thống build (CMake + west)
- ✅ Cấu hình phần cứng (Devicetree + Kconfig)
- ✅ Debugging tools (OpenOCD + GDB)
- ✅ Thư viện đồ họa (LVGL)
- ✅ Hàng trăm board được hỗ trợ sẵn

---

## 2. Tại sao cần RTOS?

### Vấn đề với lập trình tuần tự (bare-metal)

```
┌──────────────────────────────────────────────────┐
│  Loop thông thường (không có RTOS)               │
│                                                  │
│  while(1) {                                      │
│      doc_cam_bien();      // 100ms               │
│      gui_wifi();          // 500ms  ← CHẶN!      │
│      cap_nhat_man_hinh(); // 200ms               │
│      xu_ly_nut_bam();     // ?ms   ← Có thể bị  │
│  }                              bỏ lỡ!           │
└──────────────────────────────────────────────────┘
```

Nếu `gui_wifi()` mất 500ms, nút bấm trong thời gian đó sẽ **không được xử lý**. Đây là vấn đề nghiêm trọng trong hệ thống thực tế.

### Giải pháp với RTOS

```
┌──────────────────────────────────────────────────┐
│  Với Zephyr RTOS                                 │
│                                                  │
│  Thread 1 (priority cao):  xu_ly_nut_bam()       │
│  Thread 2 (priority TB):   doc_cam_bien()        │
│  Thread 3 (priority thấp): gui_wifi()            │
│                                                  │
│  → Scheduler tự động điều phối                   │
│  → Thread ưu tiên cao luôn chạy trước            │
│  → Không bao giờ bỏ lỡ sự kiện quan trọng       │
└──────────────────────────────────────────────────┘
```

---

## 3. So sánh Zephyr với các RTOS khác

| Tiêu chí | FreeRTOS | Zephyr | NuttX | VxWorks |
|---|---|---|---|---|
| **Mã nguồn mở** | ✅ | ✅ | ✅ | ❌ (trả phí) |
| **Độ phức tạp** | Thấp | Cao | Trung bình | Cao |
| **Tính năng** | Cơ bản | Đầy đủ nhất | Đầy đủ | Đầy đủ |
| **Hỗ trợ board** | Nhiều | Rất nhiều | Nhiều | Nhiều |
| **WiFi/BT tích hợp** | ❌ | ✅ | ✅ | ✅ |
| **Công ty hỗ trợ** | Amazon | Nordic, Intel, NXP, ST, TI... | Apache | Wind River |
| **Phù hợp cho** | Project nhỏ-vừa | Project chuyên nghiệp | Project phức tạp | Aerospace, Defense |

> 🎯 **Khi nào dùng Zephyr?**  
> Khi project cần: networking, multi-threading phức tạp, hỗ trợ nhiều board, hoặc bạn muốn chuẩn bị cho môi trường công nghiệp.

---

## 4. Hệ sinh thái Zephyr

```
┌─────────────────────────────────────────────────────────┐
│                   ỨNG DỤNG CỦA BẠN                      │
│              (main.c, business logic...)                 │
├─────────────────────────────────────────────────────────┤
│              THƯ VIỆN & MIDDLEWARE                       │
│    WiFi Stack │ Bluetooth │ HTTP │ LVGL Graphics         │
├─────────────────────────────────────────────────────────┤
│                   ZEPHYR RTOS CORE                       │
│        Scheduler │ Thread │ Mutex │ Semaphore            │
├──────────────────────┬──────────────────────────────────┤
│    CÔNG CỤ BUILD     │      CẤU HÌNH PHẦN CỨNG          │
│   CMake │ west       │   Devicetree │ Kconfig            │
├──────────────────────┴──────────────────────────────────┤
│                      PHẦN CỨNG                           │
│         ESP32 │ STM32 │ Nordic nRF │ NXP i.MX...        │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Phần cứng cần chuẩn bị

| STT | Linh kiện | Ghi chú |
|---|---|---|
| 1 | **ESP32-S3-DevKitC** | Board chính — có debugger tích hợp |
| 2 | Adafruit MCP9808 | Cảm biến nhiệt độ I2C |
| 3 | Adafruit 0.96" TFT LCD (ST7735R) | Màn hình hiển thị |
| 4 | Trimpot 10kΩ | Biến trở (dùng cho ADC) |
| 5 | 2× Pushbutton | Nút bấm |
| 6 | Điện trở 220Ω | Bảo vệ LED |
| 7 | LED | LED thường |
| 8 | Dây jumper | Đấu nối linh kiện |
| 9 | 2× Breadboard | Bo mạch thử nghiệm |
| 10 | Cáp USB-A to micro-B | Kết nối máy tính |

> 💡 **Tại sao chọn ESP32-S3-DevKitC?**  
> - Được Zephyr hỗ trợ chính thức  
> - Có chip JTAG debugger tích hợp sẵn (không cần mua thêm)  
> - Phổ biến, giá rẻ, dễ tìm mua

---

## 6. Python venv là gì và tại sao cần dùng?

### Vấn đề khi không dùng venv

Khi bạn cài Python packages trực tiếp lên máy (`pip install something`), tất cả packages đều vào **một "kho"** chung toàn hệ thống. Điều này gây ra xung đột:

```
Project A cần: esptool==3.0
Project B cần: esptool==4.8
→ Không thể cài cả 2 cùng lúc vào hệ thống! ❌
```

### venv giải quyết thế nào?

**Virtual Environment (venv)** tạo ra một **môi trường Python riêng biệt, độc lập** cho từng project:

```
Máy tính của bạn
├── Python (hệ thống)
├── venv/ (dự án Zephyr)   ← môi trường riêng
│   ├── python (bản sao)
│   ├── pip
│   └── packages/
│       ├── esptool==4.8.1
│       └── pyserial==3.5
└── venv_khac/ (dự án khác) ← môi trường khác
    └── packages/
        └── esptool==3.0
```

Mỗi venv hoàn toàn tách biệt — không ảnh hưởng nhau, không ảnh hưởng hệ thống.

### Phân tích từng lệnh venv

#### Tạo môi trường ảo

```bash
python -m venv venv
```

| Thành phần | Ý nghĩa |
|---|---|
| `python` | Gọi Python interpreter |
| `-m venv` | Chạy module `venv` có sẵn trong Python |
| `venv` (cuối) | Tên thư mục sẽ tạo ra (có thể đặt tên khác) |

Lệnh này tạo ra thư mục `venv/` chứa:
```
venv/
├── bin/          (Linux/macOS) hoặc Scripts/ (Windows)
│   ├── python    ← bản sao Python riêng
│   ├── pip       ← pip riêng
│   └── activate  ← script kích hoạt
├── lib/
│   └── python3.x/
│       └── site-packages/  ← packages sẽ cài vào đây
└── pyvenv.cfg    ← file cấu hình venv
```

#### Kích hoạt môi trường ảo

```bash
# Linux/macOS
source venv/bin/activate

# Windows (PowerShell)
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Unrestricted -Force
venv\Scripts\activate
```

| Thành phần | Ý nghĩa |
|---|---|
| `source` | Chạy script trong shell hiện tại (không mở shell mới) |
| `venv/bin/activate` | Script kích hoạt môi trường ảo |

Sau khi kích hoạt, terminal sẽ hiện `(venv)` ở đầu dòng:
```bash
(venv) user@machine:~$
```

**`Set-ExecutionPolicy` trên Windows:** Mặc định Windows chặn chạy script `.ps1`. Lệnh này cho phép chạy script trong phạm vi user hiện tại (`-Scope CurrentUser`).

#### Cài packages vào venv

```bash
python -m pip install pyserial==3.5 esptool==4.8.1
```

| Thành phần | Ý nghĩa |
|---|---|
| `python -m pip` | Gọi pip thông qua Python (đảm bảo đúng pip của venv) |
| `install` | Lệnh cài đặt |
| `pyserial==3.5` | Cài đúng version 3.5 (dùng `==` để pin version) |
| `esptool==4.8.1` | Công cụ flash firmware cho ESP32 |

> ⚠️ **Tại sao phải pin version (`==`)?**  
> Zephyr version 3.7 được test với các tool version cụ thể. Dùng version khác có thể gây lỗi không đoán trước được.

---

## 7. Cài đặt môi trường với Docker

### Docker là gì?

**Docker** là công cụ đóng gói ứng dụng và toàn bộ dependencies của nó vào một **container** — một môi trường ảo nhẹ, độc lập, chạy được trên mọi hệ điều hành.

```
Không có Docker:
  Máy A (Ubuntu 22.04) → Cài đặt thành công ✅
  Máy B (Ubuntu 20.04) → Lỗi dependency ❌
  Máy C (Windows)      → Không biết làm ❌

Với Docker:
  Máy A, B, C đều dùng cùng 1 container → Đều thành công ✅
```

### Các bước cài đặt chi tiết

#### Bước 1: Clone repository

```bash
git clone https://github.com/ShawnHymel/introduction-to-zephyr
cd introduction-to-zephyr
```

| Thành phần | Ý nghĩa |
|---|---|
| `git clone` | Tải toàn bộ repository về máy |
| `https://...` | URL của repo trên GitHub |
| `cd introduction-to-zephyr` | Di chuyển vào thư mục vừa clone |

#### Bước 2: Build Docker image

```bash
docker build -t env-zephyr-espressif -f Dockerfile.espressif .
```

| Tham số | Ý nghĩa |
|---|---|
| `build` | Lệnh tạo Docker image từ Dockerfile |
| `-t env-zephyr-espressif` | Đặt tên (tag) cho image: `env-zephyr-espressif` |
| `-f Dockerfile.espressif` | Chỉ định file Dockerfile cụ thể cần dùng |
| `.` | Build context là thư mục hiện tại (Docker sẽ đọc file từ đây) |

> ⏱️ **Lưu ý:** Lần đầu build có thể mất **15–30 phút** vì Docker phải tải Zephyr và toàn bộ toolchain.

#### Bước 3: Chạy Docker container

```bash
# Linux/macOS
docker run --rm -it \
  -p 3333:3333 \
  -p 2222:22 \
  -p 8800:8800 \
  -v "$(pwd)"/workspace:/workspace \
  -w /workspace \
  env-zephyr-espressif

# Windows (PowerShell)
docker run --rm -it `
  -p 3333:3333 `
  -p 2222:22 `
  -p 8800:8800 `
  -v "${PWD}\workspace:/workspace" `
  -w /workspace `
  env-zephyr-espressif
```

**Giải thích từng tham số:**

| Tham số | Ý nghĩa |
|---|---|
| `run` | Tạo và chạy một container từ image |
| `--rm` | **Tự động xóa container** khi dừng (giữ máy sạch). Mọi thứ trong container sẽ mất, **trừ** thư mục `/workspace` đã được mount |
| `-it` | `-i` (interactive): giữ stdin mở; `-t` (tty): tạo terminal ảo → kết hợp để dùng terminal tương tác |
| `-p 3333:3333` | Map port: `HOST:CONTAINER`. Port 3333 dùng cho **OpenOCD** (debugging) |
| `-p 2222:22` | Port 2222 trên máy thật → port 22 (SSH) trong container. Dùng để kết nối VS Code qua SSH |
| `-p 8800:8800` | Port 8800 cho **code-server** (VS Code chạy trên browser) |
| `-v "$(pwd)"/workspace:/workspace` | **Mount volume**: thư mục `workspace/` trên máy thật ↔ `/workspace` trong container. File lưu ở đây sẽ **tồn tại sau khi container bị xóa** |
| `-w /workspace` | Đặt working directory mặc định khi vào container |
| `env-zephyr-espressif` | Tên image để chạy |

> 📁 **Quan trọng về `--rm` và volume mount:**  
> - `--rm` xóa container → mọi file bên trong container (ngoài `/workspace`) đều mất  
> - Thư mục `/workspace` được mount từ máy thật → **luôn được lưu**  
> - Hãy **luôn lưu code vào `/workspace`** khi làm việc trong container!

#### Bước 4: Kết nối vào môi trường

**Cách 1: Browser (đơn giản nhất)**

Mở trình duyệt → `http://localhost:8800`

Bạn sẽ thấy VS Code chạy ngay trên browser, không cần cài thêm gì.

**Cách 2: VS Code local qua SSH (chuyên nghiệp hơn)**

```
1. Cài extension "Remote - SSH" trong VS Code
2. Kết nối: root@localhost:2222
3. Mật khẩu mặc định: zephyr (xem trong Dockerfile)
4. Mở workspace: File > Open Workspace from File... > /zephyr.code-workspace
```

Cài các extension được gợi ý khi được hỏi:
- **C/C++** — hỗ trợ ngôn ngữ C
- **CMake Tools** — hỗ trợ CMake
- **nRF DeviceTree** — highlight và autocomplete cho Devicetree
- **Microsoft Hex Editor** — xem file binary

---

## 8. Công cụ west — Trái tim của Zephyr

### west là gì?

**west** (Zephyr's meta-tool) là công cụ dòng lệnh "all-in-one" của Zephyr. Nó thực hiện tất cả các tác vụ phổ biến:

```
west
├── build      → Biên dịch firmware
├── flash      → Nạp firmware lên board
├── debug      → Debug qua GDB
├── update     → Cập nhật Zephyr và modules
├── boards     → Liệt kê các board được hỗ trợ
└── ...nhiều lệnh khác
```

### west là meta-tool — ý nghĩa là gì?

west không trực tiếp build code. Nó **gọi các công cụ khác** (CMake, Ninja, OpenOCD...) và truyền đúng tham số cho chúng:

```
west build
    └─→ gọi CMake (tạo build files)
            └─→ gọi Ninja/Make (thực sự compile)
                        └─→ gọi GCC/Clang (dịch từng file .c)
```

### Phân tích chi tiết lệnh west build

```bash
west build -p always -b esp32s3_devkitc/esp32s3/procpu \
  -- -DDTC_OVERLAY_FILE=boards/esp32s3_devkitc.overlay
```

**Phần trước `--`:** Tham số của `west build`

| Tham số | Ý nghĩa chi tiết |
|---|---|
| `build` | Subcommand: biên dịch project |
| `-p always` | `--pristine always`: **Luôn build sạch** — xóa toàn bộ thư mục `build/` trước khi build lại từ đầu. Dùng `auto` để chỉ clean khi cần thiết, hoặc bỏ qua để build incremental (nhanh hơn) |
| `-b esp32s3_devkitc/esp32s3/procpu` | `--board`: Chỉ định board target. Cú pháp: `<board>/<soc>/<cpu>`. `procpu` = primary CPU (ESP32-S3 có 2 CPU) |

**Phần sau `--`:** Tham số truyền thẳng cho CMake

| Tham số | Ý nghĩa chi tiết |
|---|---|
| `--` | Dấu ngăn cách: mọi thứ sau đây được truyền cho CMake, không phải west |
| `-DDTC_OVERLAY_FILE=...` | `-D` là cú pháp CMake để set biến. `DTC_OVERLAY_FILE` là biến CMake đặc biệt trong Zephyr: chỉ định file `.overlay` để ghi đè cấu hình Devicetree mặc định của board |

### Các lệnh west thường dùng khác

```bash
# Liệt kê tất cả board được Zephyr hỗ trợ
west boards

# Build với menuconfig (cấu hình Kconfig bằng giao diện TUI)
west build -t menuconfig

# Build incremental (không clean, nhanh hơn)
west build -b esp32s3_devkitc/esp32s3/procpu

# Xem log build chi tiết
west build -v

# Flash bằng west (nếu board hỗ trợ)
west flash
```

---

## 9. Build ví dụ Blink

### Cấu trúc project blink

```
workspace/apps/01_blink/
├── boards/
│   └── esp32s3_devkitc.overlay   ← Cấu hình phần cứng cho board cụ thể
├── src/
│   └── main.c                    ← Code ứng dụng chính
├── CMakeLists.txt                 ← Cấu hình build
└── prj.conf                      ← Kconfig options
```

### Chạy lệnh build

```bash
cd /workspace/apps/01_blink
west build -p always -b esp32s3_devkitc/esp32s3/procpu \
  -- -DDTC_OVERLAY_FILE=boards/esp32s3_devkitc.overlay
```

### Output sau khi build thành công

Các file quan trọng trong `build/zephyr/`:

| File | Ý nghĩa |
|---|---|
| `zephyr.bin` | Binary thuần — dùng để flash qua esptool |
| `zephyr.elf` | File ELF có debug info — dùng cho GDB debugging |
| `zephyr.hex` | Intel HEX format |
| `zephyr.map` | Memory map — xem hàm nào chiếm bao nhiêu RAM/Flash |
| `zephyr.dts` | Devicetree đã được merge hoàn chỉnh |

---

## 10. esptool — Nạp firmware lên ESP32

### esptool là gì?

**esptool** là công cụ Python mã nguồn mở do Espressif (hãng làm ESP32) phát triển. Nó giao tiếp với **ROM bootloader** trong chip ESP32 để:
- Flash firmware
- Đọc/ghi Flash memory
- Đọc thông tin chip
- Erase Flash

### Tại sao flash phải chạy trên máy thật, không phải trong Docker?

Docker container **không thể truy cập trực tiếp** cổng USB/Serial của máy thật (trừ khi cấu hình phức tạp). Vì vậy:

```
Docker container: BUILD firmware  ──→  tạo ra zephyr.bin
Máy thật:         FLASH firmware  ──→  nạp zephyr.bin lên board
```

Thư mục `workspace/` được mount nên `zephyr.bin` build trong container sẽ xuất hiện trên máy thật.

### Phân tích chi tiết lệnh flash

```bash
python -m esptool \
  --port "COM7" \
  --chip auto \
  --baud 921600 \
  --before default_reset \
  --after hard_reset \
  write_flash \
  -u \
  --flash_mode keep \
  --flash_freq 40m \
  --flash_size detect \
  0x0 \
  workspace/apps/01_blink/build/zephyr/zephyr.bin
```

**Giải thích từng tham số:**

| Tham số | Ý nghĩa |
|---|---|
| `python -m esptool` | Chạy esptool như một Python module (đảm bảo dùng đúng esptool trong venv) |
| `--port "COM7"` | Cổng Serial kết nối với board. Linux: `/dev/ttyUSB0` hoặc `/dev/ttyS0`. macOS: `/dev/tty.usbserial-XXXX`. Windows: `COM7` |
| `--chip auto` | Tự động nhận diện loại chip ESP (ESP32, ESP32-S3, ESP32-C3...). Có thể chỉ định thẳng `esp32s3` để nhanh hơn |
| `--baud 921600` | Tốc độ truyền dữ liệu (baud rate) khi flash. 921600 là tốc độ cao — flash nhanh hơn. Nếu gặp lỗi, thử giảm xuống `460800` hoặc `115200` |
| `--before default_reset` | Hành động **trước khi** flash: `default_reset` = dùng tín hiệu DTR/RTS trên Serial để tự động reset chip vào bootloader mode (không cần nhấn nút) |
| `--after hard_reset` | Hành động **sau khi** flash xong: `hard_reset` = reset chip để chạy firmware mới ngay lập tức |
| `write_flash` | Subcommand: ghi dữ liệu vào Flash memory |
| `-u` | `--no-compress`: không nén dữ liệu trước khi ghi. Đôi khi nén gây lỗi với một số board |
| `--flash_mode keep` | Giữ nguyên flash mode đã có trong firmware (không override). Các mode khác: `qio`, `dio`, `dout` |
| `--flash_freq 40m` | Tần số hoạt động của Flash: 40MHz. Các giá trị: `20m`, `40m`, `80m` |
| `--flash_size detect` | Tự động nhận diện dung lượng Flash. Thay bằng giá trị cụ thể: `2MB`, `4MB`, `8MB`... |
| `0x0` | Địa chỉ bắt đầu ghi trong Flash memory (`0x0` = địa chỉ đầu tiên) |
| `workspace/.../zephyr.bin` | Đường dẫn đến file binary cần flash |

### Tìm đúng cổng Serial

```bash
# Linux
ls /dev/tty*         # Tìm ttyUSB0, ttyACM0...

# macOS
ls /dev/tty.*        # Tìm tty.usbserial-...

# Windows (PowerShell)
Get-WMIObject Win32_SerialPort | Select Name, DeviceID
# Hoặc xem trong Device Manager
```

### Lỗi thường gặp khi flash

**Linux: `Permission denied`**
```bash
sudo usermod -a -G dialout $USER
# Đăng xuất và đăng nhập lại
```

**Không vào bootloader tự động:**  
Giữ nút **BOOT** (hoặc **IO0**) → nhấn **RESET** → thả BOOT → chạy lại lệnh flash

**Lỗi tốc độ:**  
Giảm baud rate: `--baud 460800` hoặc `--baud 115200`

---

## 11. Xem output qua Serial Monitor

### Mở Serial Monitor

```bash
python -m serial.tools.miniterm "COM7" 115200
```

| Tham số | Ý nghĩa |
|---|---|
| `serial.tools.miniterm` | Công cụ terminal Serial đơn giản có trong pyserial |
| `"COM7"` | Cổng Serial (thay bằng port của bạn) |
| `115200` | Baud rate — phải khớp với cấu hình trong firmware |

### Điều hướng trong miniterm

| Phím | Hành động |
|---|---|
| `Ctrl+]` | Thoát (Linux/macOS) |
| `Cmd+]` | Thoát (macOS) |
| `Ctrl+T` + `H` | Xem help |

### Thay thế miniterm

Bạn cũng có thể dùng các công cụ khác:
- **PuTTY** (Windows) — giao diện đồ họa
- **screen** (Linux/macOS): `screen /dev/ttyUSB0 115200`
- **VS Code Serial Monitor** extension

---

## 12. Tóm tắt luồng làm việc

```
┌─────────────────────────────────────────────────┐
│                LUỒNG LÀM VIỆC                   │
│                                                 │
│  1. VIẾT CODE                                   │
│     └── Trong Docker container (VS Code)        │
│         └── Lưu vào /workspace/apps/            │
│                                                 │
│  2. BUILD                                       │
│     └── Trong Docker container                  │
│         └── west build -p always -b <board>     │
│             └── Tạo ra build/zephyr/zephyr.bin  │
│                                                 │
│  3. FLASH                                       │
│     └── Trên máy thật (ngoài Docker)            │
│         └── python -m esptool ... zephyr.bin    │
│                                                 │
│  4. DEBUG/MONITOR                               │
│     └── Trên máy thật                           │
│         └── python -m serial.tools.miniterm     │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 13. Bảng tra cứu nhanh

### Lệnh cài đặt môi trường

```bash
# Clone repo
git clone https://github.com/ShawnHymel/introduction-to-zephyr
cd introduction-to-zephyr

# Tạo và kích hoạt venv
python -m venv venv
source venv/bin/activate          # Linux/macOS
venv\Scripts\activate             # Windows

# Cài packages
python -m pip install pyserial==3.5 esptool==4.8.1

# Build Docker image
docker build -t env-zephyr-espressif -f Dockerfile.espressif .

# Chạy Docker (Linux/macOS)
docker run --rm -it -p 3333:3333 -p 2222:22 -p 8800:8800 \
  -v "$(pwd)"/workspace:/workspace -w /workspace env-zephyr-espressif
```

### Lệnh build & flash hàng ngày

```bash
# Build project (trong Docker)
cd /workspace/apps/01_blink
west build -p always -b esp32s3_devkitc/esp32s3/procpu \
  -- -DDTC_OVERLAY_FILE=boards/esp32s3_devkitc.overlay

# Flash firmware (trên máy thật)
python -m esptool --port "<PORT>" --chip auto --baud 921600 \
  --before default_reset --after hard_reset \
  write_flash -u --flash_mode keep --flash_freq 40m \
  --flash_size detect 0x0 \
  workspace/apps/01_blink/build/zephyr/zephyr.bin

# Mở Serial Monitor (trên máy thật)
python -m serial.tools.miniterm "<PORT>" 115200
```

### Cổng Serial theo hệ điều hành

| OS | Ví dụ cổng |
|---|---|
| Windows | `COM3`, `COM7`... |
| Linux | `/dev/ttyUSB0`, `/dev/ttyACM0` |
| macOS | `/dev/tty.usbserial-1420` |

---

## 📚 Tài liệu tham khảo

- [Repository khóa học](https://github.com/ShawnHymel/introduction-to-zephyr)
- [Tài liệu chính thức Zephyr](https://docs.zephyrproject.org/latest/)
- [Hướng dẫn Getting Started Zephyr](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)
- [Tài liệu esptool](https://docs.espressif.com/projects/esptool/en/latest/)
- [Python venv documentation](https://docs.python.org/3/library/venv.html)
- [Docker documentation](https://docs.docker.com/)
- [west documentation](https://docs.zephyrproject.org/latest/develop/west/index.html)

---

> ✍️ **Ghi chú:** Tài liệu này được viết dựa trên bài giảng của ShawnHymel và được bổ sung thêm giải thích chi tiết về từng công cụ và lệnh. Zephyr version 3.7 được sử dụng xuyên suốt khóa học — version 4.0+ có thể có thay đổi về API.
