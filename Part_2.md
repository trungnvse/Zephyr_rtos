# 🔨 Bài 2: CMake Tutorial — Hiểu tường tận hệ thống build của Zephyr

> **Khóa học:** Introduction to Zephyr RTOS  
> **Bài trước:** [Bài 1 — Cài đặt môi trường & LED Blink](./README_Zephyr_Bai1.md)  
> **Tác giả gốc:** [ShawnHymel](https://github.com/ShawnHymel/introduction-to-zephyr)  
> **CMake version:** 3.20+  

---

## 📑 Mục lục

- [1. Vấn đề trước khi có CMake](#1-vấn-đề-trước-khi-có-cmake)
- [2. CMake là gì?](#2-cmake-là-gì)
- [3. CMake hoạt động như thế nào?](#3-cmake-hoạt-động-như-thế-nào)
- [4. Ví dụ Hello World với CMake](#4-ví-dụ-hello-world-với-cmake)
- [5. Phân tích từng file trong project](#5-phân-tích-từng-file-trong-project)
- [6. Phân tích CMakeLists.txt từng dòng](#6-phân-tích-cmakeliststxt-từng-dòng)
- [7. Biến nội trang trong CMake](#7-biến-nội-trang-trong-cmake)
- [8. PUBLIC vs PRIVATE vs INTERFACE](#8-public-vs-private-vs-interface)
- [9. Static Library vs Shared Library](#9-static-library-vs-shared-library)
- [10. Lệnh build và các tham số](#10-lệnh-build-và-các-tham-số)
- [11. Generators — Make vs Ninja](#11-generators--make-vs-ninja)
- [12. Tổ chức project lớn](#12-tổ-chức-project-lớn)
- [13. CMake với Zephyr](#13-cmake-với-zephyr)
- [14. So sánh CMakeLists.txt thuần và Zephyr](#14-so-sánh-cmakeliststxt-thuần-và-zephyr)
- [15. Challenge — Hello Blink](#15-challenge--hello-blink)
- [16. Lỗi thường gặp](#16-lỗi-thường-gặp)
- [17. Bảng tra cứu nhanh](#17-bảng-tra-cứu-nhanh)

---

## 1. Vấn đề trước khi có CMake

Trước khi hiểu CMake là gì, hãy hiểu **tại sao nó ra đời**.

### Khi project chỉ có 1 file

Khi bạn có một file C duy nhất, biên dịch rất đơn giản:

```bash
gcc main.c -o my_program
./my_program
```

Xong. Không vấn đề gì.

### Khi project lớn dần

Nhưng khi project có 50 file `.c`, 30 thư viện ngoài, cần biên dịch khác nhau trên Windows/Linux/macOS... bạn cần một cái gì đó để **quản lý toàn bộ quá trình biên dịch**.

Trước đây người ta dùng **Makefile**:

```makefile
CC = gcc
CFLAGS = -Wall -I./include

all: my_program

my_program: main.o my_lib.o
    $(CC) -o my_program main.o my_lib.o

main.o: src/main.c
    $(CC) $(CFLAGS) -c src/main.c

my_lib.o: src/my_lib.c
    $(CC) $(CFLAGS) -c src/my_lib.c

clean:
    rm -f *.o my_program
```

**Vấn đề của Makefile:**

| Vấn đề | Chi tiết |
|---|---|
| **Không portable** | Makefile trên Linux không chạy được trên Windows |
| **Viết tay tẻ nhạt** | Thêm 1 file `.c` → phải sửa Makefile thủ công |
| **Khó quản lý dependencies** | Phải tự theo dõi thư viện nào phụ thuộc thư viện nào |
| **Không có IDE support** | Visual Studio, Xcode không đọc được Makefile của nhau |

→ **CMake ra đời để giải quyết tất cả những vấn đề này.**

---

## 2. CMake là gì?

**CMake** là một **meta build system** — hay nói đơn giản hơn: **CMake không trực tiếp biên dịch code của bạn**, mà nó **tạo ra files cấu hình** cho các build system khác (Make, Ninja, Visual Studio...) để chúng thực sự biên dịch.

> 💡 **Ví dụ dễ hiểu:**  
> CMake giống như một **kiến trúc sư** vẽ bản thiết kế (blueprint).  
> Make/Ninja giống như **đội thợ xây** dùng bản thiết kế đó để xây nhà.  
> Bạn chỉ cần nói chuyện với kiến trúc sư — không cần biết đội thợ nào đang xây.

### Đặc điểm nổi bật của CMake

- **Cross-platform thực sự**: Viết `CMakeLists.txt` một lần, chạy được trên Windows/Linux/macOS
- **Generator-agnostic**: Cùng một config, có thể sinh ra Makefile, Ninja files, hay Visual Studio solution
- **Scalable**: Từ project nhỏ 1 file đến project khổng lồ như Zephyr (hàng nghìn file)
- **Industry standard**: Được dùng rộng rãi trong ngành: LLVM, OpenCV, Qt, Zephyr...

---

## 3. CMake hoạt động như thế nào?

### Luồng hoạt động 2 bước

```
┌─────────────────────────────────────────────────────────────┐
│                    BƯỚC 1: CONFIGURE                        │
│                                                             │
│   CMakeLists.txt  ──→  cmake -S . -B build  ──→  build/    │
│   (bạn viết)              (bạn chạy)          Makefile      │
│                                               Ninja files   │
│                                               VS solution   │
│                                               build.ninja   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     BƯỚC 2: BUILD                           │
│                                                             │
│   build/Makefile  ──→  cmake --build build  ──→  binary    │
│   (CMake tạo ra)        (bạn chạy)           my_program    │
│                                               libmy_lib.a   │
└─────────────────────────────────────────────────────────────┘
```

### Tại sao lại tách 2 bước?

- **Bước 1 (Configure)** chỉ cần chạy **một lần** khi thay đổi cấu trúc project
- **Bước 2 (Build)** chạy **mỗi lần** bạn sửa code → chỉ recompile những file thay đổi (**incremental build**)

Điều này giúp tiết kiệm thời gian rất nhiều khi project lớn.

### Quá trình đầy đủ từ source đến binary

```
CMakeLists.txt
      │
      ▼ cmake -S . -B build
  ┌─────────┐
  │  build/ │
  │ ├─ Makefile / build.ninja   ← build instructions
  │ ├─ CMakeCache.txt           ← cached variables
  │ └─ CMakeFiles/              ← internal CMake files
  └─────────┘
      │
      ▼ cmake --build build  (gọi make hoặc ninja)
  ┌─────────────────────┐
  │  Compiler (gcc/clang) biên dịch từng .c → .o  │
  │  Linker link các .o → executable hoặc library  │
  └─────────────────────┘
      │
      ▼
  build/hello_world    ← file binary cuối cùng
  build/libmy_lib.a    ← static library
```

---

## 4. Ví dụ Hello World với CMake

Bài 2 dùng project "Hello World" đơn giản để minh họa CMake. Đây là cấu trúc thư mục:

```
02_demo_cmake/
├── include/
│   └── my_lib.h        ← Header file: khai báo hàm
├── src/
│   ├── my_lib.c        ← Source file: định nghĩa hàm
│   └── main.c          ← Source file: chương trình chính
└── CMakeLists.txt      ← Cấu hình CMake
```

### Tại sao tổ chức thư mục như vậy?

Đây là **convention phổ biến** trong C/C++ projects:

| Thư mục | Mục đích | Giải thích |
|---|---|---|
| `include/` | Header files (`.h`) | Chứa **khai báo** (declarations) — cho compiler biết hàm tồn tại |
| `src/` | Source files (`.c`) | Chứa **định nghĩa** (implementations) — code thực sự |
| `build/` | Output của CMake | Tự động tạo ra, **không commit lên git** |

---

## 5. Phân tích từng file trong project

### 5.1 Header file: `include/my_lib.h`

```c
#ifndef MY_LIB_H
#define MY_LIB_H

void say_hello();

#endif
```

**Giải thích chi tiết từng dòng:**

#### `#ifndef MY_LIB_H` — Header Guard (Bảo vệ header)

Đây là **include guard** — một kỹ thuật cực kỳ quan trọng trong C.

**Vấn đề nó giải quyết:** Nếu file `main.c` include cả `a.h` và `b.h`, mà cả hai đều include `my_lib.h`:

```
main.c
├── #include "a.h"     → a.h include my_lib.h  → compiler thấy my_lib.h lần 1
└── #include "b.h"     → b.h include my_lib.h  → compiler thấy my_lib.h lần 2 ❌ LỖI!
```

Compiler sẽ báo lỗi "redefinition" vì thấy cùng một khai báo 2 lần.

**Header guard ngăn chặn điều này:**

```c
#ifndef MY_LIB_H    // Nếu MY_LIB_H CHƯA được define...
#define MY_LIB_H    // ...thì define nó (đánh dấu "đã xử lý rồi")

// ... nội dung header ...

#endif              // Kết thúc khối #ifndef
```

Lần đầu include: `MY_LIB_H` chưa tồn tại → vào trong, define nó, xử lý nội dung  
Lần hai include: `MY_LIB_H` đã tồn tại → bỏ qua toàn bộ ✅

**Convention đặt tên:** Tên macro thường là `FILENAME_H` viết hoa, dùng `_` thay `/` và `.`

#### `void say_hello();` — Khai báo hàm

Đây là **function declaration** (khai báo), không phải **definition** (định nghĩa).

```c
// KHAI BÁO (trong .h) — chỉ nói "hàm này tồn tại"
void say_hello();

// ĐỊNH NGHĨA (trong .c) — code thực sự
void say_hello() {
    printf("Hello, world!\r\n");
}
```

Compiler cần khai báo để biết hàm này có tồn tại khi gặp `say_hello()` trong `main.c`. Linker sau đó sẽ tìm định nghĩa thực sự trong `my_lib.c`.

---

### 5.2 Library source: `src/my_lib.c`

```c
#include <stdio.h>
#include "my_lib.h"

void say_hello() {
    printf("Hello, world!\r\n");
}
```

**Phân biệt hai cách include:**

| Cú pháp | Ý nghĩa | Dùng khi nào |
|---|---|---|
| `#include <stdio.h>` | Dùng dấu `< >` | Include **system/standard library** — compiler tự tìm trong system paths |
| `#include "my_lib.h"` | Dùng dấu `" "` | Include **file local của project** — tìm trong thư mục hiện tại trước |

**`\r\n` vs `\n`:**

| Ký tự | Tên | Ý nghĩa | Dùng khi nào |
|---|---|---|---|
| `\n` | Line Feed (LF) | Xuống dòng (Unix/Linux/macOS) | Hầu hết terminal hiện đại |
| `\r\n` | Carriage Return + Line Feed (CRLF) | Xuống dòng (Windows) | Serial terminal của vi điều khiển |

Trong embedded/Zephyr, dùng `\r\n` vì serial terminal thường cần cả hai ký tự.

---

### 5.3 Main executable: `src/main.c`

```c
#include "my_lib.h"

int main() {
    say_hello();
    return 0;
}
```

**Tại sao `return 0`?**

Quy ước trong C: `main()` trả về `int` để báo cho hệ điều hành biết chương trình kết thúc thành công hay thất bại:
- `return 0` → thành công (exit code 0)
- `return 1` (hoặc bất kỳ số khác) → có lỗi

Trong embedded systems (Zephyr), `main()` thường không bao giờ return vì chương trình chạy vô tận, nhưng quy ước vẫn giữ nguyên.

---

## 6. Phân tích CMakeLists.txt từng dòng

```cmake
# Specify a minimum CMake version
cmake_minimum_required(VERSION 3.20.0)

# Name the project
project(
    hello_world
    VERSION 1.0
    DESCRIPTION "The classic"
    LANGUAGES C
)

# Create a static library target named "my_lib"
add_library(my_lib
    STATIC
    src/my_lib.c
)

# Set the include directories for the library
target_include_directories(
    my_lib
    PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# Create an executable target
add_executable(
    ${PROJECT_NAME}
    src/main.c
)

# Link the library to the executable
target_link_libraries(
    ${PROJECT_NAME}
    PRIVATE
    my_lib
)
```

---

### 6.1 `cmake_minimum_required(VERSION 3.20.0)`

```cmake
cmake_minimum_required(VERSION 3.20.0)
```

**Tại sao cần khai báo version tối thiểu?**

CMake thay đổi behavior qua các version. Khai báo version tối thiểu đảm bảo:
- CMake biết dùng "policy" (quy tắc xử lý) của version đó
- Người dùng có CMake cũ hơn sẽ nhận được lỗi rõ ràng thay vì lỗi bí ẩn

```
Nếu máy bạn có CMake 3.18 mà project yêu cầu 3.20:
→ CMake báo lỗi ngay: "CMake 3.20 or higher is required"
→ Thay vì lỗi mơ hồ sau đó
```

**Cú pháp version:**

```cmake
cmake_minimum_required(VERSION major.minor.patch)
# Ví dụ: 3.20.0 = major 3, minor 20, patch 0
```

---

### 6.2 `project(...)`

```cmake
project(
    hello_world
    VERSION 1.0
    DESCRIPTION "The classic"
    LANGUAGES C
)
```

**Lệnh `project()` làm những gì?**

1. Đặt tên project → tạo biến `${PROJECT_NAME}` = `"hello_world"`
2. Đặt version → tạo các biến:
   - `${PROJECT_VERSION}` = `"1.0"`
   - `${PROJECT_VERSION_MAJOR}` = `"1"`
   - `${PROJECT_VERSION_MINOR}` = `"0"`
3. Khai báo ngôn ngữ → CMake tìm và verify compiler phù hợp

**`LANGUAGES C` quan trọng vì:**

| Nếu khai báo | CMake làm gì |
|---|---|
| `LANGUAGES C` | Tìm C compiler (gcc), set biến `CMAKE_C_COMPILER` |
| `LANGUAGES CXX` | Tìm C++ compiler (g++) |
| `LANGUAGES C CXX` | Tìm cả hai |
| Không khai báo | Mặc định tìm cả C và C++ (chậm hơn không cần thiết) |

---

### 6.3 `add_library()`

```cmake
add_library(my_lib
    STATIC
    src/my_lib.c
)
```

**Library là gì?**

Library là một tập hợp code được compile sẵn, có thể tái sử dụng cho nhiều executable khác nhau.

```
Không có library:                  Có library:
──────────────────                 ──────────────────
main_a.c                           my_lib.c
  └── copy code say_hello          └── say_hello()  ← compile một lần
main_b.c                                  ↑
  └── copy code say_hello          main_a.c          main_b.c
main_c.c                             └── link          └── link
  └── copy code say_hello
```

**Tham số của `add_library()`:**

```cmake
add_library(<tên_target> <loại> <file_nguồn...>)
```

| Tham số | Ý nghĩa |
|---|---|
| `my_lib` | Tên target — sẽ dùng tên này để reference ở nơi khác |
| `STATIC` | Loại library (xem mục 9 để hiểu chi tiết) |
| `src/my_lib.c` | File(s) source cần compile vào library |

---

### 6.4 `target_include_directories()`

```cmake
target_include_directories(
    my_lib
    PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

**Vấn đề lệnh này giải quyết:**

Khi compiler gặp `#include "my_lib.h"` trong `my_lib.c`, nó cần biết **tìm file đó ở đâu**. Lệnh này chỉ định đường dẫn.

```
my_lib.c có: #include "my_lib.h"
                           ↑
                  Tìm ở đâu?
                           ↓
target_include_directories → /workspace/02_demo_cmake/include/
                           ↓
                  Tìm thấy: include/my_lib.h ✅
```

**Tham số:**

| Tham số | Ý nghĩa |
|---|---|
| `my_lib` | Target nào được áp dụng |
| `PUBLIC` | Phạm vi (xem mục 8) |
| `${CMAKE_CURRENT_SOURCE_DIR}/include` | Đường dẫn thư mục chứa headers |

**`${CMAKE_CURRENT_SOURCE_DIR}` là gì?**

Đây là **biến nội trang** (built-in variable) của CMake, tự động được gán = đường dẫn tuyệt đối đến thư mục chứa file `CMakeLists.txt` đang được xử lý. Xem thêm tại mục 7.

---

### 6.5 `add_executable()`

```cmake
add_executable(
    ${PROJECT_NAME}
    src/main.c
)
```

**Tạo target executable** — file chạy được cuối cùng.

| Tham số | Ý nghĩa |
|---|---|
| `${PROJECT_NAME}` | Tên executable = tên project = `hello_world` |
| `src/main.c` | File(s) source cho executable |

Sau khi build, file `build/hello_world` (Linux/macOS) hoặc `build/hello_world.exe` (Windows) sẽ được tạo ra.

**Dùng `${PROJECT_NAME}` thay vì hardcode tên** là best practice — nếu đổi tên project, không cần sửa nhiều chỗ.

---

### 6.6 `target_link_libraries()`

```cmake
target_link_libraries(
    ${PROJECT_NAME}
    PRIVATE
    my_lib
)
```

**Linker là gì và tại sao cần link?**

Compiler chỉ dịch từng file `.c` thành file object `.o` riêng lẻ. Linker **kết nối** chúng lại:

```
Compiler:
  main.c     →  main.o
  my_lib.c   →  my_lib.o (hoặc libmy_lib.a nếu là library)

Linker:
  main.o + my_lib.o  →  hello_world (executable)
```

`target_link_libraries()` nói với CMake: "Khi build `hello_world`, hãy link thêm `my_lib` vào."

---

## 7. Biến nội trang trong CMake

CMake có nhiều biến tự động được tạo ra. Đây là những biến quan trọng nhất:

| Biến | Ý nghĩa | Ví dụ giá trị |
|---|---|---|
| `${PROJECT_NAME}` | Tên project đã khai báo | `hello_world` |
| `${PROJECT_VERSION}` | Version project | `1.0` |
| `${CMAKE_CURRENT_SOURCE_DIR}` | Thư mục chứa CMakeLists.txt **đang xử lý** | `/home/user/02_demo_cmake` |
| `${CMAKE_CURRENT_BINARY_DIR}` | Thư mục build tương ứng | `/home/user/02_demo_cmake/build` |
| `${CMAKE_SOURCE_DIR}` | Thư mục chứa CMakeLists.txt **gốc (top-level)** | `/home/user/02_demo_cmake` |
| `${CMAKE_BINARY_DIR}` | Thư mục build gốc | `/home/user/02_demo_cmake/build` |
| `${CMAKE_C_COMPILER}` | Đường dẫn C compiler | `/usr/bin/gcc` |
| `${CMAKE_BUILD_TYPE}` | Kiểu build | `Debug`, `Release`, `MinSizeRel` |

**`CMAKE_CURRENT_SOURCE_DIR` vs `CMAKE_SOURCE_DIR`:**

```
project_root/              ← CMAKE_SOURCE_DIR
├── CMakeLists.txt         ← top-level
├── src/
│   └── main.c
└── libs/
    └── my_lib/
        ├── CMakeLists.txt ← CMAKE_CURRENT_SOURCE_DIR khi xử lý file này
        └── my_lib.c
```

Khi CMake đang xử lý `libs/my_lib/CMakeLists.txt`:
- `${CMAKE_SOURCE_DIR}` = `project_root/` (luôn là gốc)
- `${CMAKE_CURRENT_SOURCE_DIR}` = `project_root/libs/my_lib/` (thư mục hiện tại)

---

## 8. PUBLIC vs PRIVATE vs INTERFACE

Đây là khái niệm **quan trọng nhất** trong "Modern CMake". Nó kiểm soát **dependency propagation** — tức là khi A phụ thuộc B, liệu C phụ thuộc A có tự động "thấy" B hay không.

### Hình dung bằng sơ đồ

```
          app (executable)
           │
           │ link to
           ▼
        my_lib (library)
           │
           │ depends on
           ▼
      third_party_lib
```

**Câu hỏi:** Khi `app` link vào `my_lib`, `app` có tự động link vào `third_party_lib` không?

→ **Phụ thuộc vào keyword bạn dùng khi khai báo dependency của `my_lib`.**

### Bảng so sánh

| Keyword | my_lib dùng được? | app tự động thấy? | Dùng khi nào |
|---|---|---|---|
| `PRIVATE` | ✅ Có | ❌ Không | Dependency chỉ dùng trong **implementation** của target, không expose ra ngoài |
| `PUBLIC` | ✅ Có | ✅ Có | Dependency dùng trong **cả implementation và interface** (header) của target |
| `INTERFACE` | ❌ Không | ✅ Có | Dependency chỉ cần cho **người dùng** target, không cần trong implementation |

### Ví dụ thực tế

**Trường hợp PRIVATE** (dùng trong bài 2):
```cmake
target_link_libraries(hello_world PRIVATE my_lib)
```
→ `hello_world` dùng `my_lib` trong code, nhưng nếu có `app2` link vào `hello_world`, `app2` **không** tự động thấy `my_lib`.

**Trường hợp PUBLIC:**
```cmake
# my_lib dùng math_lib trong header của nó
# → bất kỳ ai link my_lib đều cần math_lib
target_link_libraries(my_lib PUBLIC math_lib)
```
Ví dụ: `my_lib.h` có `#include <math.h>` và dùng kiểu `complex_t` từ `math_lib`. Khi `app` include `my_lib.h`, nó cũng cần `math_lib`.

**Quy tắc đơn giản để nhớ:**
- Dependency chỉ xuất hiện trong file `.c` → dùng **PRIVATE**
- Dependency xuất hiện trong file `.h` → dùng **PUBLIC**

---

## 9. Static Library vs Shared Library

Trong `add_library()`, bạn có thể chọn loại library:

### Static Library (`STATIC`)

```cmake
add_library(my_lib STATIC src/my_lib.c)
# → tạo ra: libmy_lib.a (Linux/macOS) hoặc my_lib.lib (Windows)
```

**Cách hoạt động:**

```
Compile time:
  my_lib.c → my_lib.o → libmy_lib.a
  main.c   → main.o

Link time:
  main.o + libmy_lib.a → hello_world
                              ↑
               Code của my_lib được COPY VÀO executable
```

### Shared Library (`SHARED`)

```cmake
add_library(my_lib SHARED src/my_lib.c)
# → tạo ra: libmy_lib.so (Linux) / libmy_lib.dylib (macOS) / my_lib.dll (Windows)
```

**Cách hoạt động:**

```
Compile time:
  my_lib.c → libmy_lib.so (file riêng biệt)

Link time:
  main.o → hello_world (chỉ lưu "tham chiếu" đến libmy_lib.so)

Runtime:
  hello_world khởi động → OS tìm và load libmy_lib.so vào RAM
```

### So sánh

| Tiêu chí | Static (`.a`) | Shared (`.so/.dll`) |
|---|---|---|
| **Kích thước executable** | Lớn hơn (code lib nằm trong) | Nhỏ hơn |
| **Kích thước RAM** | Mỗi process có bản riêng | Nhiều process chia sẻ 1 bản |
| **Deploy** | Đơn giản — 1 file duy nhất | Phức tạp — phải ship kèm `.so` |
| **Khởi động** | Nhanh hơn | Chậm hơn (cần load lib) |
| **Embedded/Zephyr** | ✅ **Dùng static** | ❌ Không hỗ trợ |

> 🎯 **Trong Zephyr/embedded:** Luôn dùng **STATIC** vì không có runtime dynamic linker trên vi điều khiển.

---

## 10. Lệnh build và các tham số

### Bước 1: Configure (sinh build files)

```bash
cmake -S . -B build
```

| Tham số | Ý nghĩa |
|---|---|
| `-S .` | `--source`: Thư mục chứa `CMakeLists.txt` gốc. `.` = thư mục hiện tại |
| `-B build` | `--build`: Thư mục **output** sẽ chứa build files. Tự động tạo nếu chưa có |

**Output của bước này trong thư mục `build/`:**

```
build/
├── CMakeCache.txt       ← Cache các biến CMake (quan trọng!)
├── CMakeFiles/          ← Internal CMake files
├── Makefile             ← Nếu dùng Make generator
├── build.ninja          ← Nếu dùng Ninja generator
└── cmake_install.cmake  ← Script để install
```

**`CMakeCache.txt` là gì?**

CMake lưu cache để lần sau không phải tìm lại compiler, libraries... Nếu gặp vấn đề kỳ lạ, hãy xóa `build/` và chạy lại từ đầu (giống `-p always` của west).

### Bước 2: Build (compile thực sự)

```bash
cmake --build build
```

| Tham số | Ý nghĩa |
|---|---|
| `--build build` | Thư mục chứa build files (sinh ra ở bước 1) |

**Các tùy chọn thêm:**

```bash
# Build với nhiều CPU cores (nhanh hơn)
cmake --build build --parallel 4
cmake --build build -j 4        # cú pháp ngắn gọn

# Chỉ build một target cụ thể
cmake --build build --target my_lib

# Xóa toàn bộ build artifacts (clean)
cmake --build build --target clean

# Build với verbose output (xem lệnh compiler thực sự chạy)
cmake --build build --verbose
```

### Bước 3: Chạy executable

```bash
./build/hello_world
# Output: Hello, world!
```

### Các tham số khi Configure thêm

```bash
# Set build type (ảnh hưởng optimization flags)
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

# Truyền biến CMake bất kỳ bằng -D
cmake -S . -B build -DMY_VARIABLE=value

# Dùng compiler cụ thể
cmake -S . -B build -DCMAKE_C_COMPILER=clang
```

**Các `CMAKE_BUILD_TYPE` phổ biến:**

| Type | Optimization | Debug Info | Dùng khi |
|---|---|---|---|
| `Debug` | Không (`-O0`) | Có (`-g`) | Đang phát triển, debugging |
| `Release` | Cao (`-O3`) | Không | Build cuối, production |
| `RelWithDebInfo` | Vừa (`-O2`) | Có (`-g`) | Production nhưng vẫn có thể debug |
| `MinSizeRel` | Tối thiểu size (`-Os`) | Không | Embedded, flash nhỏ |

---

## 11. Generators — Make vs Ninja

CMake hỗ trợ nhiều **generator** — mỗi generator sinh ra build files cho một build system khác nhau.

### Xem danh sách generators

```bash
cmake -h
# Phần cuối output sẽ liệt kê các generators
```

### So sánh Make và Ninja

| Tiêu chí | Unix Makefiles | Ninja |
|---|---|---|
| **Tốc độ build** | Chậm hơn | **Nhanh hơn** (thiết kế tối ưu cho parallel) |
| **Parallel build** | `make -j4` | Mặc định parallel |
| **Output** | Verbose, nhiều chữ | Gọn gàng, tiến trình rõ ràng |
| **Có sẵn trên** | Linux/macOS mặc định | Cần cài thêm (có sẵn trong Docker Zephyr) |
| **Zephyr dùng** | Không | ✅ **Ninja** |

### Chỉ định generator

```bash
# Dùng Unix Makefiles (mặc định trên Linux/macOS)
cmake -S . -B build -G "Unix Makefiles"
cmake --build build

# Dùng Ninja
cmake -S . -B build -G "Ninja"
cmake --build build

# Dùng Visual Studio (Windows)
cmake -S . -B build -G "Visual Studio 17 2022"
```

> 💡 **Điểm mấu chốt:** Cùng một `CMakeLists.txt`, chỉ cần thay `-G` là có thể build bằng tool khác nhau. Đây chính là sức mạnh cross-platform của CMake.

---

## 12. Tổ chức project lớn

Khi project lớn lên, bạn cần tổ chức code tốt hơn. CMake hỗ trợ điều này qua:

### 12.1 Nested CMakeLists.txt với `add_subdirectory()`

```
big_project/
├── CMakeLists.txt          ← Top-level (orchestrator)
├── src/
│   ├── CMakeLists.txt      ← Build app executable
│   └── main.c
└── libs/
    ├── CMakeLists.txt      ← Build tất cả libraries
    ├── network/
    │   ├── CMakeLists.txt  ← Build network library
    │   └── network.c
    └── sensor/
        ├── CMakeLists.txt  ← Build sensor library
        └── sensor.c
```

**Top-level `CMakeLists.txt`:**
```cmake
cmake_minimum_required(VERSION 3.20.0)
project(big_project LANGUAGES C)

# Nói với CMake: hãy vào thư mục libs/ và xử lý CMakeLists.txt ở đó
add_subdirectory(libs)

# Sau đó vào src/
add_subdirectory(src)
```

**`libs/CMakeLists.txt`:**
```cmake
add_subdirectory(network)
add_subdirectory(sensor)
```

**`src/CMakeLists.txt`:**
```cmake
add_executable(big_project main.c)
target_link_libraries(big_project PRIVATE network sensor)
```

### 12.2 Target-based approach (Modern CMake)

**Cách cũ (tránh dùng):** Set global flags
```cmake
# ❌ KHÔNG NÊN — ảnh hưởng toàn bộ project
include_directories(include/)
add_definitions(-DDEBUG)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
```

**Cách mới (nên dùng):** Gắn properties vào từng target
```cmake
# ✅ NÊN — chỉ ảnh hưởng target cụ thể
target_include_directories(my_lib PRIVATE include/)
target_compile_definitions(my_lib PRIVATE DEBUG=1)
target_compile_options(my_lib PRIVATE -Wall -Wextra)
```

**Lý do:** Khi project lớn, global settings gây ra xung đột khó debug. Target-based approach giữ mọi thứ tách biệt và rõ ràng.

---

## 13. CMake với Zephyr

Zephyr **mở rộng** CMake với các functions và variables riêng của mình. Đây là những điểm khác biệt quan trọng:

### 13.1 `find_package(Zephyr)`

```cmake
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
```

**Lệnh này làm gì?**

`find_package()` là lệnh CMake chuẩn để tìm và load một "package" (tập hợp CMake functions/variables).

| Tham số | Ý nghĩa |
|---|---|
| `Zephyr` | Tên package cần tìm |
| `REQUIRED` | Nếu không tìm thấy → báo lỗi và dừng (không phải warning) |
| `HINTS $ENV{ZEPHYR_BASE}` | Gợi ý đường dẫn tìm kiếm: lấy từ biến môi trường `ZEPHYR_BASE` |

Sau lệnh này, CMake đã load toàn bộ Zephyr CMake infrastructure:
- Tìm đúng compiler cho board target
- Load tất cả Zephyr-specific CMake functions
- Set hàng trăm biến liên quan đến board, SoC, architecture

### 13.2 Target `app` — Target đặc biệt của Zephyr

```cmake
# ❌ KHÔNG DÙNG TRONG ZEPHYR
add_executable(my_app src/main.c)

# ✅ ĐÚNG CÁCH TRONG ZEPHYR
target_sources(app PRIVATE src/main.c)
```

Khi `find_package(Zephyr)` được gọi, Zephyr **tự động tạo ra** một target tên là `app`. Bạn **không tạo lại** — chỉ thêm sources vào đó.

**Lý do:** Zephyr cần kiểm soát cách build executable (linking với kernel, bootloader...). Họ tạo sẵn target `app` và bạn chỉ cần "đăng ký" source files của mình.

### 13.3 west gọi CMake như thế nào

Khi bạn chạy:
```bash
west build -p always -b esp32s3_devkitc/esp32s3/procpu
```

west thực chất chạy một chuỗi lệnh CMake như thế này:
```bash
# west tự động chạy điều này (không cần bạn gõ)
cmake \
  -DBOARD=esp32s3_devkitc/esp32s3/procpu \
  -DZEPHYR_BASE=/opt/toolchains/zephyr \
  -GNinja \
  -S /workspace/apps/my_app \
  -B /workspace/apps/my_app/build

cmake --build /workspace/apps/my_app/build
```

west giúp bạn không cần nhớ và gõ các tham số dài dòng này.

---

## 14. So sánh CMakeLists.txt thuần và Zephyr

| | CMake thuần (Hello World) | CMake + Zephyr (Blink) |
|---|---|---|
| **find_package** | Không cần | `find_package(Zephyr REQUIRED ...)` |
| **project()** | Đặt tên project | Đặt tên project |
| **Executable target** | `add_executable(my_app ...)` | Không tạo executable — dùng target `app` có sẵn |
| **Thêm source** | `add_executable(my_app src/main.c)` | `target_sources(app PRIVATE src/main.c)` |
| **Link library** | `target_link_libraries(my_app ...)` | `target_link_libraries(app ...)` |
| **Build command** | `cmake -S . -B build && cmake --build build` | `west build -b <board>` |

**CMakeLists.txt đầy đủ của một Zephyr app:**

```cmake
cmake_minimum_required(VERSION 3.20.0)

# Bước quan trọng nhất: load Zephyr CMake infrastructure
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

project(blink_app)

# Thêm source files vào target "app" (Zephyr đã tạo sẵn target này)
target_sources(app PRIVATE src/main.c)

# Nếu có library riêng, vẫn tạo bình thường:
# add_library(my_lib STATIC src/my_lib.c)
# target_include_directories(my_lib PUBLIC include/)
# target_link_libraries(app PRIVATE my_lib)
```

---

## 15. Challenge — Hello Blink

**Mục tiêu:** Kết hợp Hello World (Bài 2) với LED Blink (Bài 1).

Mỗi lần LED toggle, in ra `"Hello!!!"` qua Serial.

### Cấu trúc thư mục đề xuất

```
02_solution_hello_blink/
├── include/
│   └── say_hello.h     ← Khai báo hàm say_hello()
├── src/
│   ├── say_hello.c     ← Định nghĩa hàm say_hello()
│   └── main.c          ← Blink + gọi say_hello()
└── CMakeLists.txt      ← Thêm library vào Zephyr project
```

### Gợi ý `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20.0)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(hello_blink)

# Tạo library say_hello
add_library(say_hello_lib STATIC src/say_hello.c)
target_include_directories(say_hello_lib PUBLIC include/)

# Link library vào Zephyr app target
target_sources(app PRIVATE src/main.c)
target_link_libraries(app PRIVATE say_hello_lib)
```

### Gợi ý `main.c`

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include "say_hello.h"

// ... (GPIO setup từ Bài 1)

while (1) {
    gpio_pin_toggle_dt(&led);
    say_hello();           // ← Thêm dòng này
    k_msleep(1000);
}
```

**Xem solution:** [02_solution_hello_blink](https://github.com/ShawnHymel/introduction-to-zephyr/tree/main/workspace/apps/02_solution_hello_blink)

---

## 16. Lỗi thường gặp

### ❌ `CMakeLists.txt` viết sai tên (case sensitive)

```
CMakeLists.txt  ← ĐÚNG (C và M viết hoa)
cmakeLists.txt  ← SAI
cmakelists.txt  ← SAI
Cmakelists.txt  ← SAI
```

CMake tìm đúng tên `CMakeLists.txt` — sai một chữ hoa/thường sẽ không tìm thấy.

### ❌ Không tìm thấy compiler

```
CMake Error: No CMAKE_C_COMPILER could be found
```

**Giải pháp:**
```bash
# Linux
sudo apt install gcc build-essential

# macOS
xcode-select --install

# Hoặc chỉ định thẳng:
cmake -S . -B build -DCMAKE_C_COMPILER=/usr/bin/gcc
```

### ❌ Header file không tìm thấy

```
fatal error: my_lib.h: No such file or directory
```

**Kiểm tra:** Đường dẫn trong `target_include_directories()` có đúng không?

```cmake
# Sai nếu thư mục thực tế là include/ nhưng bạn viết:
target_include_directories(my_lib PUBLIC includes/)  # ← thừa 's'

# Đúng:
target_include_directories(my_lib PUBLIC include/)
```

### ❌ Lỗi version mismatch

```
CMake Error: CMake 3.25 or higher is required. You are running version 3.18
```

**Giải pháp:**
```bash
# Cài CMake mới hơn, hoặc hạ version trong CMakeLists.txt
cmake_minimum_required(VERSION 3.18.0)  # hạ xuống version bạn đang có
```

### ❌ Build directory cũ gây lỗi lạ

Đôi khi `CMakeCache.txt` có thông tin cũ gây conflict.

**Giải pháp:** Xóa build directory và configure lại từ đầu:
```bash
rm -rf build/
cmake -S . -B build
cmake --build build
```

---

## 17. Bảng tra cứu nhanh

### Lệnh CMake thường dùng

```bash
# Configure project
cmake -S . -B build

# Configure với Ninja generator
cmake -S . -B build -G "Ninja"

# Configure với Release build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release

# Build project
cmake --build build

# Build parallel (nhanh hơn)
cmake --build build -j 4

# Build verbose (xem lệnh compiler)
cmake --build build --verbose

# Clean build
cmake --build build --target clean

# Xóa hoàn toàn và configure lại
rm -rf build/ && cmake -S . -B build
```

### CMake functions quan trọng

```cmake
# Khai báo
cmake_minimum_required(VERSION 3.20.0)
project(my_project LANGUAGES C)

# Tạo targets
add_executable(my_app src/main.c src/other.c)
add_library(my_lib STATIC src/lib.c)

# Cấu hình targets
target_include_directories(my_lib PUBLIC include/)
target_sources(my_app PRIVATE src/extra.c)
target_link_libraries(my_app PRIVATE my_lib)
target_compile_options(my_lib PRIVATE -Wall)
target_compile_definitions(my_lib PRIVATE MY_DEFINE=1)

# Tổ chức project lớn
add_subdirectory(libs/my_lib)
```

### CMakeLists.txt tối giản cho Zephyr

```cmake
cmake_minimum_required(VERSION 3.20.0)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_zephyr_app)
target_sources(app PRIVATE src/main.c)
```

---

## 📚 Tài liệu tham khảo

| Tài liệu | Link |
|---|---|
| CMake Official Documentation | [cmake.org/documentation](https://cmake.org/documentation/) |
| Modern CMake (free book) | [cliutils.gitlab.io/modern-cmake](https://cliutils.gitlab.io/modern-cmake/) |
| CMake Tutorial (official) | [cmake.org/cmake/help/latest/guide/tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html) |
| Zephyr CMake Extensions | [Zephyr west build docs](https://docs.zephyrproject.org/latest/develop/west/build-flash-debug.html) |
| Solution Bài 2 | [02_solution_hello_blink](https://github.com/ShawnHymel/introduction-to-zephyr/tree/main/workspace/apps/02_solution_hello_blink) |

---

> ✍️ *Ghi chú học tập — dựa trên bài giảng của ShawnHymel, bổ sung giải thích chi tiết.*  
> ⬅️ **Bài trước:** Bài 1 — Cài đặt & Blink | ➡️ **Bài tiếp theo:** Bài 3 — Kconfig
