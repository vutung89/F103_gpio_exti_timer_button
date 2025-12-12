# F103_gpio_exti_timer_button

## 📋 Mô tả

Dự án này là một ứng dụng nhúng (Embedded) sử dụng vi điều khiển **STM32F103C8T6**, tập trung vào việc xử lý nút bấm (Button) với các kỹ thuật nâng cao bao gồm khử nhiễu (debouncing), phát hiện nhấn dài (long press), và nhấn đôi (double press).

Dự án được phát triển với **STM32CubeMX** và sử dụng **STM32 HAL (Hardware Abstraction Layer)** để tương tác với phần cứng.

---

## ✨ Tính năng chính

- **GPIO Control** - Điều khiển các chân I/O của vi điều khiển
- **EXTI (External Interrupt)** - Xử lý ngắt ngoài từ các sự kiện trên chân GPIO
- **Timer Management** - Sử dụng bộ định thời (Timer) để đo thời gian
- **Advanced Button Handling:**
  - **Short Press Detection** - Nhấn ngắn (<2 giây)
  - **Long Press Detection** - Nhấn và giữ (>=2 giây)
  - **Double Press Detection** - Nhấn hai lần liên tiếp (trong 1 giây)
  - **Software Debouncing** - Khử nhiễu mềm (100ms)
- **UART Debug** - In thông tin debug qua UART

---

## 🔧 I. CẤU HÌNH PHẦN CỨNG CHI TIẾT

### 1. GPIO Configuration

#### 1.1 Button Input Pin
```
Port: PA0
Mode: GPIO_MODE_IT_RISING 
Pull: GPIO_PULLDOWN (button mặc định kéo về GND khi không nhấn, tránh hiện tượng floating)
Speed: GPIO_SPEED_FREQ_LOW
```

#### 1.2 LED Output Pin
```
Port: PB2
Mode: GPIO_MODE_OUTPUT_PP
Pull: GPIO_NOPULL
Speed: GPIO_SPEED_FREQ_LOW
```

#### 1.2 UART Debug pin
```
Port: Uart2 (PA2(TX), PA3(RX))
Mode: 115200 8N1
- Sử dụng printf() để in thông tin debug
- Hàm _write() được override để gửi qua UART
- Timeout: 100ms cho mỗi lần truyền
```

### 2. External Interrupt (EXTI) Configuration

#### 2.1 EXTI Line Setup
```
EXTI Line: EXTIn (tùy theo GPIO pin được dùng)
Trigger Event: GPIO_IT_FALLING (Falling Edge)
Interrupt Priority (NVIC): 2
Sub-priority: 0
Enable: Yes
```

#### 2.2 EXTI Handler
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == BUTTON_PIN)
    {
        btn_pin_state = HAL_GPIO_ReadPin(BUTTON_PORT, BUTTON_PIN);
        exti_flag = 1;
        btn_state = BTN_DEBOUNCE;
        debounce_cnt = 0;
    }
}
```

### 3. Timer Configuration (TIM2)

#### 3.1 Timer Setup
```
Timer: TIM2 (32-bit timer)
Clock Source: Internal Clock (APB1: 72MHz)
Mode: Up-counting
Prescaler: 71 (chia 72MHz thành 1MHz)
Period (ARR): 999 (tạo chu kỳ 1ms)
Frequency: 1MHz / 1000 = 1kHz → 1ms per tick
Update Interrupt: Enabled
Priority: 1
```

#### 3.2 Tính toán Prescaler và Period

**Công thức:**
```
Interrupt Frequency = (Clock Speed) / ((Prescaler + 1) × (ARR + 1))
Clock Speed = 72MHz (STM32F103 APB1)
```

**Tính toán chi tiết:**
```
Timer Frequency = 72MHz / 72 = 1MHz
Interrupt Frequency = 1MHz / 1000 = 1kHz
Timer Tick Interval = 1ms
```

#### 3.3 Timer Handler
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        debounce_cnt++;
        if (btn_state == BTN_DEBOUNCE || 
            btn_state == BTN_PRESSED ||
            btn_state == BTN_WAIT_RELEASE)
        {
            press_time++;
        }
    }
}
```
---

## 🎯 II. CÁC KỸ THUẬT SỬ DỤNG

### 1. Finite State Machine (FSM)

#### 1.1 Định nghĩa Trạng thái
```c
typedef enum {
    BTN_IDLE,                    // Chờ sự kiện
    BTN_DEBOUNCE,                // Khử nhiễu khi nhấn
    BTN_PRESSED,                 // Nút đang được nhấn
    BTN_DEBOUNCE_RELEASE,        // Khử nhiễu khi nhả
    BTN_WAIT_RELEASE,            // Chờ nhả (trong long press)
    BTN_WAIT_SECOND_PRESS,       // Chờ nhấn thứ 2 (double press)
    BTN_LONG_PRESS_DETECTED      // Long press đã được phát hiện
} ButtonState;
```

#### 1.2 Sơ đồ Chuyển Trạng thái

```
                    ┌──────────────────────────────────────┐
                    ▼                                      │
            ┌──────────────┐                               │
            │   BTN_IDLE   │◄──────────────────────────────┘
            └───┬──────────┘
                │ EXTI: Button pressed
                │ debounce_cnt = 0
                ▼
            ┌──────────────────┐
            │  BTN_DEBOUNCE    │
            └────┬─────────────┘
                 │ Timer: debounce_cnt >= 100ms
                 │ & button still pressed
                 ▼
            ┌──────────────────┐
            │  BTN_PRESSED     │
            └────┬──┬──────────┘
                 │  │
        Press>=2s│  │ EXTI: Button released
                 │  └──────────────────┐
                 │                     │
                 ▼                     ▼
        ┌─────────────────┐    ┌──────────────────────┐
        │ BTN_WAIT_RELEASE│    │ BTN_DEBOUNCE_RELEASE │
        └────┬────────────┘    └──┬───────────────────┘
             │ EXTI: Released      │ Timer: debounce_cnt >= 100ms
             │                     │
             ▼                     ▼
        ┌──────────────┐    ┌──────────────────┐
        │ Long Press   │    │WAIT_SECOND_PRESS │
        │  Detected    │    └──┬───────────────┘
        └──────┬───────┘       │
               │               │ Timer: Timeout (>1s)
               │               │ Short Press Detected
               │               │
               └───────┬───────┘
                       │ EXTI: 2nd Press (within 1s)
                       │ Double Press Detected
                       ▼
                   BTN_IDLE
```

### 2. Software Debouncing Technique

#### 2.1 Nguyên lý
- **Vấn đề:** Chân button có thể có nhiễu cơ học từ rung động
- **Giải pháp:** Chỉ xác nhận sự kiện sau khi tín hiệu ổn định 100ms

#### 2.2 Timeline Debouncing
```
Thời gian  │ Tín hiệu  │ Trạng thái         │ Hành động
───────────┼───────────┼────────────────────┼──────────────────
0ms        │ 1→0→1→0   │ BTN_IDLE           │ EXTI triggered
           │ (nhiễu)   │                    │
───────────┼───────────┼────────────────────┼──────────────────
0-50ms     │ 0         │ BTN_DEBOUNCE       │ debounce_cnt++
           │ (not stable)                   │
───────────┼───────────┼────────────────────┼──────────────────
50-100ms   │ 0         │ BTN_DEBOUNCE       │ debounce_cnt++
           │ (stable)  │                    │
───────────┼───────────┼────────────────────┼──────────────────
100ms      │ 0         │ BTN_PRESSED        │ Short/Long Press
           │ (confirmed)                    │ Detection starts
───────────┼───────────┼────────────────────┼──────────────────
```

### 3. Button Event Detection

#### 3.1 Single/Double/Triple Press Detection
```
Điều kiện: 
  • Debounce xong (tín hiệu ổn định)
  • Thời gian giữa các lần nhấn < 1 giây
  • Nhả nút

Flow:
  BTN_IDLE → EXTI → BTN_DEBOUNCE → BTN_PRESSED
  → BTN_DEBOUNCE_RELEASE → BTN_WAIT_SECOND_PRESS
  → BTN_PRESSED or Timeout → Single/Double/Triple Press Event → BTN_IDLE
```

#### 3.2 Long Press Detection
```
Điều kiện:
  • Nhấn giữ ≥ 2 giây
  • Nhả nút

Flow:
  BTN_PRESSED (press_time ≥ 2000ms)
  → BTN_LONG_PRESS_DETECTED → Long Press Event
  → BTN_WAIT_RELEASE → BTN_IDLE
```

### 4. Interrupt Handling Strategy

#### 4.1 EXTI Interrupt (Priority: 2)
```
Sự kiện: Cạnh xuống (button nhấn hoặc nhả)
Công việc:
  • Đọc trạng thái chân GPIO
  • Set EXTI flag
  • Chuyển sang BTN_DEBOUNCE
  • Reset debounce_cnt
```

#### 4.2 Timer Interrupt (Priority: 1)
```
Sự kiện: Mỗi 1ms
Công việc:
  • Tăng debounce_cnt
  • Tăng press_time
  • Kiểm tra điều kiện long press
  • Kiểm tra timeout double press
```

---

## 📊 III. FLOWCHART CHƯƠNG TRÌNH

### 1. Main Program Flowchart

```
        ┌─────────────────────┐
        │ SystemClock_Config()│
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │  MX_GPIO_Init()     │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │ MX_USART2_UART_Init│
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │  MX_TIM2_Init()     │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────────────┐
        │ HAL_TIM_Base_Start_IT()     │
        │ (Bắt đầu Timer Interrupt)   │
        └──────────┬──────────────────┘
                   │
        ┌──────────▼──────────────────┐
        │ Infinite Loop (Main Loop)   │
        └──────────┬──────────────────┘
                   │
          ┌────────┴─────────┐
          │                  │
    ┌─────▼──────┐    ┌──────▼─────┐
    │ Check      │    │ Handle     │
    │ btn_state  │    │ Events     │
    │            │    │            │
    └─────┬──────┘    └──────┬─────┘
          │                  │
          └────────┬─────────┘
                   │
        ┌──────────▼──────────┐
        │  Print Debug Info   │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │  Loop back          │
        └─────────────────────┘
```

### 2. Button State Machine Flowchart

```
        ┌──────────────────────┐
        │ Timer Interrupt      │
        │ (Every 1ms)          │
        └──────────┬───────────┘
                   │
        ┌──────────▼───────────────────┐
        │ if (btn_state == BTN_DEBOUNCE)
        └──────────┬───────────────────┘
                   │
             ┌─────▼──────┐
             │debounce_cnt│ >= 100ms?
             │ >= 100     │
             └────┬───────┘
                  │
          ┌──Yes──┴──No──┐
          ▼              ▼
        ┌─────┐        ┌──────┐
        │ btn │        │ Wait │
        │=    │        │ more │
        │PRES │        └──────┘
        │SED  │
        └──┬──┘
            │
            ▼
        Check if press_time >= 2000ms
            │
        ┌────┴─────┐
        │          │
    Yes         No
        │          │
        ▼          ▼
    Long Press  Continue...
```

### 3. EXTI Interrupt Handler

```
        ┌─────────────────────────────┐
        │ GPIO_EXTI_Callback          │
        │ (Button edge detected)      │
        └──────────┬──────────────────┘
                   │
        ┌──────────▼──────────────────┐
        │ Read GPIO pin state         │
        │ btn_pin_state = ReadPin()   │
        └──────────┬──────────────────┘
                   │
        ┌──────────▼──────────────────┐
        │ Set EXTI flag               │
        │ exti_flag = 1               │
        └──────────┬──────────────────┘
                   │
        ┌──────────▼──────────────────────────┐
        │ if (btn_state == IDLE or            │
        │     WAIT_SECOND_PRESS)              │
        └──────────┬──────────────────────────┘
                   │
            ┌──────┴──────┐
            │             │
       Yes  │             │  No
        ┌───▼──────┐  ┌───▼────────────┐
        │ btn_state│  │ Ignore         │
        │ =        │  │ (debounce)     │
        │ DEBOUNCE │  └────────────────┘
        │ cnt = 0  │
        └──────────┘
```

---

## 📋 IV. BẢNG THAM SỐ VÀ Ý NGHĨA

### 4.1 Hằng số Thời gian (Timing Constants)

| Tham số | Giá trị | Đơn vị | Ý nghĩa | Tác động Khi Thay Đổi |
|---------|--------|--------|---------|----------------------|
| `DEBOUNCE_MS` | 100 | ms | Thời gian khử nhiễu | ↓: Nhanh hơn nhưng dễ nhạy. ↑: Ổn định hơn nhưng chậm |
| `LONG_PRESS_MS` | 2000 | ms | Ngưỡng long press | ↓: Dễ kích hoạt. ↑: Cần giữ lâu hơn |
| `DOUBLE_PRESS_MS` | 1000 | ms | Khoảng thời gian double press | ↓: Phải nhấn nhanh. ↑: Có thời gian hơn |

### 4.2 Cấu hình Timer

| Tham số | Giá trị | Đơn vị | Ý nghĩa |
|---------|--------|--------|---------|
| Clock Frequency | 72 | MHz | Tần số xung nhập Timer |
| Prescaler | 71 | - | Chia 72MHz → 1MHz |
| Period (ARR) | 999 | - | Chu kỳ 1ms |
| Timer Frequency | 1 | MHz | Tần số Timer |
| Interrupt Freq | 1 | kHz | Interrupt 1000x/giây = 1ms |
| Tick Duration | 1 | ms | Mỗi tick = 1ms |

### 4.3 UART Configuration

| Tham số | Giá trị | Ý nghĩa |
|---------|--------|---------|
| Baud Rate | 115200 | bits per second |
| Data Bits | 8 | Số bit dữ liệu |
| Stop Bits | 1 | Bit dừng |
| Parity | None | Không kiểm tra parity |

---

## 🔢 V. CÔNG THỨC TÍNH TOÁN

### 5.1 Tính Prescaler
```
Prescaler = (Clock_Frequency / Desired_Frequency) - 1
Prescaler = (72MHz / 1MHz) - 1 = 71
```

### 5.2 Tính Period (ARR)
```
ARR = (Timer_Frequency / Desired_Interrupt_Frequency) - 1
ARR = (1MHz / 1kHz) - 1 = 999
```

### 5.3 Chuyển Tick thành Milliseconds
```
Time_ms = tick_count × (1 / Interrupt_Frequency)
Ví dụ:
  debounce_cnt = 100 → 100ms
  press_time = 2000 → 2000ms = 2 giây
```

---

## ⚙️ VI. HƯỚNG DẪN CHỈNH SỬA THAM SỐ

### 6.1 Thay đổi Debounce Time

**File:** `Core/Src/main.c`

```c
#define DEBOUNCE_MS  100  // Thay đổi giá trị này
```

**Khuyến nghị:** 50-200ms

---

### 6.2 Thay đổi Long Press Threshold

**File:** `Core/Src/main.c`

```c
#define LONG_PRESS_MS  2000  // Thay đổi giá trị này
```

**Khuyến nghị:** 1000-3000ms

---

### 6.3 Thay đổi Double Press Window

**File:** `Core/Src/main.c`

```c
#define DOUBLE_PRESS_MS  1000  // Thay đổi giá trị này
```

**Khuyến nghị:** 300-1500ms

---

### 6.4 Thay đổi Timer Frequency

**Trong STM32CubeMX:**

1. Mở file `.ioc` 
2. Vào **Timers → TIM2**
3. Đổi **Prescaler** hoặc **Period (ARR)**

**Ví dụ:** Muốn interrupt mỗi 2ms:
```
Prescaler = 71
Period (ARR) = 1999
→ Interrupt frequency = 500Hz → 2ms
```

---

## 📁 VII. CẤU TRÚC THƯ MỤC

```
F103_gpio_exti_timer_button/
├── Core/
│   ├── Inc/
│   │   ├── main.h              # Hằng số, prototype hàm
│   │   ├── stm32f1xx_hal_conf.h # Cấu hình HAL
│   │   └── stm32f1xx_it.h       # Handler ngắt
│   ├── Src/
│   │   ├── main.c              # Logic chính, state machine
│   │   ├── stm32f1xx_hal_msp.c  # Khởi tạo MSP
│   │   ├── stm32f1xx_it.c       # Handler EXTI & Timer
│   │   ├── syscalls.c
│   │   ├── sysmem.c
│   │   └── system_stm32f1xx.c
│   └── Startup/
│       └── startup_stm32f103c8tx.s
├── Drivers/
│   ├── CMSIS/
│   └── STM32F1xx_HAL_Driver/
├── Debug/                       # Build output
├── F103_gpio_exti_timer_button.ioc # CubeMX config
├── STM32F103C8TX_FLASH.ld      # Linker script
├── Makefile                     # Build script
└── README.md                    # Tài liệu
```

---

## 📚 VIII. THAM KHẢO

- [STM32F103 Datasheet](https://www.st.com/resource/en/datasheet/stm32f103c8.pdf)
- [STM32F1 Reference Manual](https://www.st.com/resource/en/reference_manual/cd00171190-stm32f10xxx_20xx_21xxx_l1xxxx_full_reference_manual.pdf)
- [STM32 HAL Documentation](https://www.st.com/en/embedded-software/stm32cube-embedded-software.html)

---

**Tác giả:** [Nhập tên của bạn]  
**Cập nhật:** December 2025  
**License:** STMicroelectronics - AS-IS
