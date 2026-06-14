---
name: embedded-oop-architecture
description: "Compile-time OOP 5-layer architecture for Cortex-M bare-metal C projects (STM32/GD32/APM32/HC32). Virtual-table polymorphism, dependency injection, single-header platform isolation, and zero code change when swapping chips. Use when starting a new embedded project, designing a multi-platform firmware framework, or doing HAL abstraction. NOT for event notification between modules — use embedded-fnptr-register for that."
version: 3.0.0
author: ZHAO Yankun + Hermes
platforms: [universal]
tags: [embedded, mcu, oop, architecture, vtable, hal, stm32, gd32, apm32, cortex-m, dependency-injection]
metadata:
  hermes:
    tags: [embedded, oop, vtable, hal, stm32, gd32, apm32, cortex-m, dependency-injection]
    category: embedded
  install:
    hermes: "hermes skills install <url>"
    claude-code: "Tell Claude: 'Install skill from <url> and save to CLAUDE.md'"
    codex: "Tell Codex: 'Read and apply this skill from <url>'"
---

# OOP 5-Layer Architecture (Virtual-Table Pattern, Embedded C)

A **compile-time decoupling** pattern for Cortex-M bare-metal firmware written in C. Five layers — `Interface → Adapter → Device → Board → App` — with virtual function tables (`_ops_t`) and dependency injection through `board_config`. The Device, Board, and App layers never include a vendor header. **Switching STM32 → GD32 → APM32 is a one-line macro change and a build directory swap, with zero edits to the three platform-independent layers.**

> **Scope.** This skill is about *project layout, file ownership, and platform isolation*. It is **not** an event-dispatch or ISR-notification pattern. If your goal is "an ISR fires and N modules need to know", use `embedded-fnptr-register` instead. The two skills compose: an OOP project's Device layer commonly registers a callback (fnptr-register) so that the App layer can subscribe to hardware events without depending on the chip.

## When to Use This Skill

Use it when **any** of the following is true:

- You are starting a new MCU project and want a long-lived structure (the project will outlive the first chip you pick).
- The same firmware must build for two or more chips (STM32F4 today, GD32F4 tomorrow, an in-house ASIC the year after).
- You want a board swap (different pin map, different clock tree) without rewriting business logic.
- You want unit-testable Device drivers on a host gcc with no MCU.
- You are tired of `#ifdef STM32 … #elif GD32 …` smeared across business code.

**Do not use it** for: one-off blinky demos, projects that will never leave one chip, or anything where "move fast" beats "live long".

## The 5-Layer Model

```
┌────────────────────────────────────────────────────────────┐
│  App Layer       main.c / motor_app.c / sensor_app.c       │  Business logic, no HW deps
├────────────────────────────────────────────────────────────┤
│  Board Layer     board_config.h / board_config.c           │  Pin map, clock tree, wiring
├────────────────────────────────────────────────────────────┤
│  Device Layer    led.c / uart_comm.c / motor_drv.c         │  Drivers, hardware-shape agnostic
├────────────────────────────────────────────────────────────┤
│  Adapter Layer   stm32_gpio.c / gd32_gpio.c / apm32_uart.c │  Chip-peripheral → interface
├────────────────────────────────────────────────────────────┤
│  Interface Layer igpio.h / iuart.h / ii2c.h / ipwm.h      │  Pure virtual, headers only
└────────────────────────────────────────────────────────────┘
   ▲ Dependencies flow DOWN only (App → Board → Device → Adapter → Interface).
   ▲ Each layer is replaceable independently.
```

**Why 5 layers, not 3?** Because the Adapter layer is what makes multi-platform possible without `#ifdef` blocks. If you collapse Adapter into Device, every Device file ends up `#ifdef`-ing the chip — which is exactly what this skill exists to avoid.

| Layer | Allowed to include | Forbidden |
|---|---|---|
| App | Board, Device, stdlib | Any `*_hal.h`, vendor register header |
| Board | Device, Interface, Platformdefine | App headers, vendor HAL |
| Device | Interface | Platformdefine, vendor HAL |
| Adapter | Platformdefine, vendor HAL, Interface | Device, Board, App |
| Interface | stdint, stdbool, stddef | Everything else |

---

## Tools to Invoke While Applying This Skill

| Stage | Tool | Why |
|---|---|---|
| Audit current file layout | `mcp__codegraph__codegraph_files` | Verify the 5-layer directory tree is intact. |
| Find all callers of a HAL symbol | `mcp__codegraph__codegraph_callers` | Locate the Device files that still leak `HAL_GPIO_WritePin` before refactoring. |
| Verify an interface's signature | `mcp__codegraph__codegraph_node` | Confirm `_ops_t` field order and return types. |
| Trace a pin from board to GPIO register | `mcp__codegraph__codegraph_trace from=<board_init> to=<vendor_register>` | One call proves the path goes Board → Device → Interface → Adapter. |
| Refactor an existing file | `Read` then `Edit` | Surgical change to add `if (p_dev == NULL) return;` etc. |
| Create new layer files | `Write` | One file per layer role. |
| Build per platform | `Bash` (`cmake -DCURRENT_PLATFORM=…`, `make`) | Verify the same App compiles on two chips. |
| Host-side unit test | `Bash` (`gcc -DHOST_TEST` on Device + Interface + mock adapter) | Run Device tests on the build server, no MCU required. |
| Static analysis | `Bash` (`cppcheck --enable=warning,style`) | Catch missing NULL checks, unused ops fields. |

---

## Naming Conventions (Enforced)

These are not suggestions — they make the layer visible at a glance.

| Element | Rule | Example |
|---|---|---|
| Interface files | `i` prefix, lowercase peripheral | `igpio.h`, `iuart.h`, `ii2c.h`, `ipwm.h`, `ispi.h` |
| Interface types | `_if_t` suffix | `gpio_if_t`, `uart_if_t` |
| Operation tables | `_ops_t` suffix | `gpio_ops_t`, `uart_ops_t` |
| Adapter files | `<chip>_<peripheral>.c` | `stm32_gpio.c`, `gd32_uart.c`, `apm32_i2c.c` |
| Adapter exported instances | `<peripheral>_adapter` | `gpio_adapter`, `uart1_adapter` |
| Device types | `_dev_t` suffix | `led_dev_t`, `motor_dev_t` |
| Device files | peripheral noun | `led.c`, `motor.c`, `uart_comm.c` |
| App files | `_app` suffix | `motor_app.c`, `sensor_app.c` |
| Board file | `board_config.c` / `board_config.h` | (single canonical name) |
| Platform macro file | `Platformdefine.h` | (single canonical name) |
| Booleans / state | `0u`, `1u`, `0x00u` — explicit unsigned | `if (count >= MAX) return;` |
| NULL guards | always check both `p_dev` and `p_dev->ops` | `if (p_dev == NULL \|\| p_dev->ops == NULL) return;` |
| Doxygen | `/** @brief ... */` in C-comment style; Chinese or English OK | `/** @brief 初始化 LED 设备 */` |

---

## Platform Isolation — The Single-File Contract

**`Platformdefine.h` is the *only* file that knows the chip.** Every other file in the project may include it, but only the Adapter layer is *required* to.

```c
/* Platformdefine.h */
#ifndef PLATFORM_DEFINE_H
#define PLATFORM_DEFINE_H

/* ---------- Platform selection (change ONLY this) ---------- */
#define PLATFORM_STM32  1u
#define PLATFORM_GD32   2u
#define PLATFORM_APM32  3u
#define PLATFORM_HC32   4u

#define CURRENT_PLATFORM  PLATFORM_STM32    /* flip this to swap chips */

#if CURRENT_PLATFORM == PLATFORM_STM32
  #include "stm32f4xx_hal.h"
  #define PLATFORM_NAME  "STM32_HAL"
#elif CURRENT_PLATFORM == PLATFORM_GD32
  #include "gd32f4xx.h"
  #define PLATFORM_NAME  "GD32_STD"
#elif CURRENT_PLATFORM == PLATFORM_APM32
  #include "apm32f4xx.h"
  #define PLATFORM_NAME  "APM32"
#elif CURRENT_PLATFORM == PLATFORM_HC32
  #include "hc32f4xx.h"
  #define PLATFORM_NAME  "HC32"
#else
  #error "CURRENT_PLATFORM is not set to a supported value"
#endif

#endif /* PLATFORM_DEFINE_H */
```

**Directory layout enforces the contract at build time, not by `#ifdef`:**

```
project/
├── platform/
│   ├── stm32/        ← only built when CURRENT_PLATFORM == PLATFORM_STM32
│   ├── gd32/         ← only built when CURRENT_PLATFORM == PLATFORM_GD32
│   ├── apm32/
│   └── hc32/
├── interface/        ← always built, no chip headers
├── device/           ← always built, no chip headers
├── board/            ← always built, no chip headers (only the board's pin map)
├── app/              ← always built, no chip headers
├── Platformdefine.h
├── CMakeLists.txt
└── Makefile
```

**Why both the macro AND the directory split?** Belt and suspenders. The directory split means even if you forget to set the macro, only one platform's `.c` is compiled. The macro means Platformdefine.h knows which chip's header to expose for the Adapter to use.

---

## Complete Skeleton — Walkthrough

### Step 1 — Interface Layer (`interface/igpio.h`)

The interface is a **header-only contract**. It contains no platform code, no static data, no inline function bodies that depend on a chip.

```c
/* interface/igpio.h */
#ifndef IGPIO_H
#define IGPIO_H
#include <stdint.h>
#include <stdbool.h>

typedef enum {
    GPIO_LOW  = 0u,
    GPIO_HIGH = 1u
} gpio_level_t;

typedef enum {
    GPIO_DIR_INPUT  = 0u,
    GPIO_DIR_OUTPUT = 1u
} gpio_dir_t;

typedef struct {
    void          (*write)(uint16_t pin, gpio_level_t level);
    gpio_level_t  (*read) (uint16_t pin);
    void          (*toggle)(uint16_t pin);
    void          (*set_dir)(uint16_t pin, gpio_dir_t dir);
} gpio_ops_t;

typedef struct {
    const gpio_ops_t *ops;        /* resolved at link time */
} gpio_if_t;

#endif /* IGPIO_H */
```

The pin is encoded as `(port << 8) | mask`. A small inline helper in the interface header (still header-only) makes call sites readable:

```c
/* still in igpio.h, still platform-agnostic */
#define GPIO_PIN(port, mask)  ((uint16_t)(((port) << 8u) | (mask)))
#define GPIO_PORT(p)          ((uint8_t)(((p) >> 8u) & 0x0Fu))
#define GPIO_MASK(p)          ((uint16_t)((p) & 0xFFu))
```

### Step 2 — Adapter Layer (`platform/stm32/stm32_gpio.c`)

The adapter **owns the vtable** for its chip. Nothing else exports a `gpio_ops_t`.

```c
/* platform/stm32/stm32_gpio.c */
#include "Platformdefine.h"      /* pulls in stm32f4xx_hal.h */
#include "igpio.h"

static GPIO_TypeDef *const k_ports[] = {
    GPIOA, GPIOB, GPIOC, GPIOD, GPIOE, GPIOF, GPIOG, GPIOH
};

static void stm32_write(uint16_t pin, gpio_level_t lvl) {
    HAL_GPIO_WritePin(k_ports[GPIO_PORT(pin)], (uint16_t)GPIO_MASK(pin),
        (lvl == GPIO_HIGH) ? GPIO_PIN_SET : GPIO_PIN_RESET);
}
static gpio_level_t stm32_read(uint16_t pin) {
    return (HAL_GPIO_ReadPin(k_ports[GPIO_PORT(pin)], (uint16_t)GPIO_MASK(pin))
            == GPIO_PIN_SET) ? GPIO_HIGH : GPIO_LOW;
}
static void stm32_toggle(uint16_t pin) {
    HAL_GPIO_TogglePin(k_ports[GPIO_PORT(pin)], (uint16_t)GPIO_MASK(pin));
}
static void stm32_set_dir(uint16_t pin, gpio_dir_t dir) {
    /* init done in board_config; set_dir is a no-op on STM32 HAL after MX_GPIO_Init */
    (void)pin; (void)dir;
}

static const gpio_ops_t k_stm32_gpio_ops = {
    .write   = stm32_write,
    .read    = stm32_read,
    .toggle  = stm32_toggle,
    .set_dir = stm32_set_dir
};

const gpio_if_t gpio_adapter = { .ops = &k_stm32_gpio_ops };
```

The `gd32_gpio.c` file is structurally identical with `gd32f4xx_gpio.h` calls in place of `HAL_GPIO_*`. Same shape, different bodies. Same `gpio_adapter` symbol.

### Step 3 — Device Layer (`device/led.c`)

The Device layer **never includes a chip header**. It only knows the interface.

```c
/* device/led.c */
#include "led.h"
#include "igpio.h"

typedef struct {
    const gpio_if_t *bus;
    uint16_t         pin;
    bool             active_low;
    bool             inited;
} led_dev_t;

static led_dev_t g_led;

void led_init(const gpio_if_t *bus, uint16_t pin, bool active_low) {
    if (bus == NULL) return;
    g_led.bus        = bus;
    g_led.pin        = pin;
    g_led.active_low = active_low;
    g_led.inited     = true;
}

static void led_set(bool on) {
    if (!g_led.inited || g_led.bus == NULL || g_led.bus->ops == NULL) return;
    bool high = g_led.active_low ? !on : on;
    g_led.bus->ops->write(g_led.pin, high ? GPIO_HIGH : GPIO_LOW);
}

void led_on (void) { led_set(true);  }
void led_off(void) { led_set(false); }
void led_toggle(void) {
    if (!g_led.inited || g_led.bus == NULL || g_led.bus->ops == NULL) return;
    g_led.bus->ops->toggle(g_led.pin);
}
```

`led.h` is the Device's public surface — it does **not** expose `gpio_if_t *` to the App; the App receives a `led_dev_t *` (or just calls the global `led_*` functions and never sees the type at all).

### Step 4 — Board Layer (`board/board_config.c`)

The board **owns the wiring**. Pin map, clock, and which adapter instance goes with which device. It is the *only* place where the linker names like `gpio_adapter` appear.

```c
/* board/board_config.c */
#include "Platformdefine.h"
#include "igpio.h"
#include "iuart.h"
#include "led.h"
#include "uart_comm.h"

/* Adapter symbols come from the platform/* build slot */
extern const gpio_if_t gpio_adapter;
extern const uart_if_t uart1_adapter;

void board_init(void) {
    /* Pin map: PE5, active-low LED; USART1 @ 115200 8N1 */
    led_init(&gpio_adapter, GPIO_PIN(4u, (1u << 5u)), true);
    uart_comm_init(&uart1_adapter, 115200u);
    /* Per-chip HAL/system clock init is the Adapter's board.c (MX_* / SystemInit) */
}
```

`board/board_config.h` exposes only the prototypes, not the adapter externs:

```c
/* board/board_config.h */
#ifndef BOARD_CONFIG_H
#define BOARD_CONFIG_H
void board_init(void);
#endif
```

### Step 5 — App Layer (`app/main.c`)

The App **never names an adapter**. It calls only Device and Board functions.

```c
/* app/main.c */
#include "Platformdefine.h"        /* allowed: only to call HAL_Init/SystemClock_Config */
#include "board_config.h"
#include "led.h"
#include "uart_comm.h"

int main(void) {
    HAL_Init();
    SystemClock_Config();          /* chip-specific, but only because App is the entry */
    board_init();

    uart_comm_print("boot\r\n");
    for (;;) {
        led_toggle();
        HAL_Delay(500u);
    }
}
```

> The App *does* include `Platformdefine.h`, but **only** because `HAL_Init()` / `SystemClock_Config()` are mandatory at the entry point on most HALs. If you want the App to be 100% chip-clean, wrap the chip init in a `chip_init()` function exported by the Adapter, and have the App call that instead.

---

## Dependency Injection — The Whole Point

```
board_config.c
    ├── holds extern adapter references  (extern const gpio_if_t gpio_adapter;)
    └── calls Device init with adapter pointer
            led_init(&gpio_adapter, …)
            │
            ▼
led.c stores the pointer:    g_led.bus = &gpio_adapter
            │
            ▼
at call time:                g_led.bus->ops->write(pin, level);
            │
            ▼
resolved at link time to:    stm32_write()  or  gd32_write()  or  apm32_write()
```

**The Device layer doesn't know or care which chip is in the build.** The compiler doesn't even see `stm32_write` — only the linker does, and only the Adapter `.c` that matches `CURRENT_PLATFORM` is in the link line.

This is **dependency inversion**: high-level policy (Device) does not depend on low-level detail (chip HAL); both depend on the abstraction (Interface).

---

## Switching Platforms — The 3-Step Procedure

1. **Edit `Platformdefine.h`** — change `CURRENT_PLATFORM` to the new value.
2. **Swap the build slot** — in CMake/Make, change `PLATFORM_DIR := platform/stm32` to `platform/gd32`. Only that directory's `.c` files are compiled.
3. **Recompile** — the Device, Board, and App source files are **byte-for-byte identical** between builds.

If the new platform lacks a function (say, GD32's UART driver doesn't have `HAL_UARTEx_ReceiveToIdle`), add the **missing** op to the Interface as a default-no-op or `_Optional` and have all Adapters implement it. **Do not** add `#ifdef GD32` inside the Device.

---

## Extending the Architecture

### Adding a New Peripheral

Suppose you need SPI. Four small changes, all in the Adapter slot and the Board:

```c
/* 1. interface/ispi.h — new interface */
typedef struct { … } spi_ops_t;
typedef struct { const spi_ops_t *ops; } spi_if_t;

/* 2. platform/stm32/stm32_spi.c — new adapter */
const spi_if_t spi1_adapter = { .ops = &k_stm32_spi_ops };

/* 3. device/flash.c — new device, no chip code */
void flash_init(const spi_if_t *bus, …);

/* 4. board/board_config.c — wire it */
extern const spi_if_t spi1_adapter;
flash_init(&spi1_adapter, …);
```

App, other Devices, and the rest of the Board file: untouched.

### Adding a New Chip

Suppose you need to support APM32. Three small changes, all in a new platform slot:

```c
/* 1. platform/apm32/apm32_gpio.c — mirror of stm32_gpio.c, apm32 HAL calls */
const gpio_if_t gpio_adapter = { .ops = &k_apm32_gpio_ops };

/* 2. Platformdefine.h — add the #elif branch */
#elif CURRENT_PLATFORM == PLATFORM_APM32
  #include "apm32f4xx.h"

/* 3. Build system — add platform/apm32 to the platform list */
```

All Adapters you have so far must have an `apm32_*` counterpart. If you ship a new chip without a counterpart, the link will fail loudly — **which is the right failure mode**. A missing adapter is a compile-time problem, not a runtime surprise.

### Composing with `embedded-fnptr-register`

The Device layer often needs to notify the App of an event (ADC done, frame received, key press). The clean way is the function-pointer registration from the companion skill:

```c
/* device/uart_comm.c — uses fnptr-register for events, oop-architecture for HAL */
typedef void (*uart_rx_cb_t)(uint8_t b, void *ctx);
void uart_comm_register_rx(uart_rx_cb_t cb, void *ctx);

/* The Device internally does: bus->ops->read(); if (g_rx_cb) g_rx_cb(b, g_rx_ctx); */
```

The App uses the *event* skill to subscribe; the Device uses the *architecture* skill to talk to the chip. The two patterns do not overlap.

---

## Interface Versioning

When the Interface evolves, follow semantic rules:

| Change | Action |
|---|---|
| Add a new op at the end of `_ops_t` | All Adapters must add a function. Compile error is OK — it forces the update. |
| Add a new op in the middle | **Forbidden.** Breaks every Adapter's initializer. |
| Remove an op | Mark as `_Optional` first; only delete after every Adapter is updated. |
| Rename an op | Add a `#define OLD_NAME NEW_NAME` shim, ship the rename, remove the shim in a later major. |
| Change a signature | Major version bump on the Interface header guard (`IGPIO_H → IGPIO_H_V2`). |

A single guard macro per interface plus a `CHANGELOG` at the top of the file is enough for most projects.

---

## Build System Skeleton (CMake)

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(firmware C)

set(PLATFORM stm32 CACHE STRING "stm32|gd32|apm32|hc32")
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Platform-independent sources
set(IPIFACE_SRC
    interface/igpio.c
    interface/iuart.c
    interface/ii2c.c
)
# Note: interfaces are header-only in this skill; .c files exist only if you
#       need a default no-op implementation. The list is empty by default.

set(DEVICE_SRC
    device/led.c
    device/uart_comm.c
    device/motor.c
)
set(BOARD_SRC  board/board_config.c)
set(APP_SRC    app/main.c)

# Platform slot — exactly one
set(PLATFORM_SRC
    platform/${PLATFORM}/${PLATFORM}_gpio.c
    platform/${PLATFORM}/${PLATFORM}_uart.c
    platform/${PLATFORM}/${PLATFORM}_i2c.c
    platform/${PLATFORM}/${PLATFORM}_board.c   # MX_*/SystemClock_Config
)

add_executable(firmware.elf
    ${DEVICE_SRC} ${BOARD_SRC} ${APP_SRC} ${PLATFORM_SRC}
)

target_compile_definitions(firmware.elf PRIVATE
    "CURRENT_PLATFORM=PLATFORM_${PLATFORM^^}"
)

# Cross-compile toolchain (set CMAKE_TOOLCHAIN_FILE=arm-none-eabi.cmake)
target_link_options(firmware.elf PRIVATE
    -mcpu=cortex-m4 -mthumb -nostartfiles
    -T${PLATFORM}/linker.ld
)
```

Switch platform: `cmake -B build-stm32 -DPLATFORM=stm32 && cmake -B build-gd32 -DPLATFORM=gd32 && make -C build-stm32 && make -C build-gd32`. Same source tree, two binaries.

---

## Host-Side Unit Testing (No MCU Required)

The Device layer depends only on the Interface. Swap the adapter for a mock at link time:

```c
/* test/mocks/mock_gpio.c */
#include "igpio.h"
#include <string.h>
static uint32_t mock_state = 0u;

static void mock_write(uint16_t pin, gpio_level_t l) {
    if (l == GPIO_HIGH) mock_state |=  (1u << (pin & 0x1Fu));
    else                mock_state &= ~(1u << (pin & 0x1Fu));
}
static gpio_level_t mock_read(uint16_t pin) {
    return (mock_state & (1u << (pin & 0x1Fu))) ? GPIO_HIGH : GPIO_LOW;
}
static void mock_toggle(uint16_t pin) {
    mock_state ^= (1u << (pin & 0x1Fu));
}
static void mock_set_dir(uint16_t p, gpio_dir_t d) { (void)p; (void)d; }

static const gpio_ops_t k_mock_ops = { mock_write, mock_read, mock_toggle, mock_set_dir };
const gpio_if_t gpio_adapter = { .ops = &k_mock_ops };

uint32_t mock_gpio_state(void) { return mock_state; }
```

```c
/* test/test_led.c */
#include "led.h"
#include "mock_gpio.h"

void test_led_on_active_low(void) {
    led_init(&gpio_adapter, 5u, /*active_low=*/true);
    led_on();
    TEST_ASSERT_EQUAL_UINT32(0u, mock_gpio_state());   /* pin low → LED on */
    led_off();
    TEST_ASSERT_EQUAL_UINT32((1u << 5u), mock_gpio_state());
}
```

Compile: `gcc -DHOST_TEST test_led.c device/led.c test/mocks/mock_gpio.c -o test_led && ./test_led`. **No MCU, no HAL, no OpenOCD, no J-Link.** A CI server can run this in 200 ms.

---

## Pitfalls and Prevention

| Pitfall | Symptom | Prevention |
|---|---|---|
| Device calls vendor function directly | Linker pulls in HAL into "platform-independent" layer | `grep -R "HAL_" device/ board/ app/` in CI; must be empty |
| `#ifdef STM32` inside a Device file | Two platforms compile the same file differently | Each chip must have its own Adapter `.c`; never both |
| Adapter exposes raw `GPIO_TypeDef *` | App drags in vendor header transitively | Adapter hands out only `gpio_if_t *`; never the raw struct |
| Calling `dev->ops->fn()` before `init()` | Hard fault on first call | `dev->inited == false` → return; or assert + reset |
| Board file contains business logic | Logic becomes chip-coupled through the back door | Board file: only `*_init(adapter, …)` calls |
| `Platformdefine.h` included by Device | Layer boundary is fake | Compile-time grep: Device/Interface must not include `Platformdefine.h` |
| App includes `extern const gpio_if_t gpio_adapter;` | App becomes a second board file | App must go through Device or Board APIs only |
| Same vtable for two peripherals | All Adapters in one `.c` become unmaintainable | One Adapter `.c` per `(chip, peripheral)` |
| `static inline` in Interface that calls vendor | Interface is no longer header-only-agnostic | Inline helpers in Interface must use only Interface types |
| Forgetting `_adapter` symbol on a new platform | Linker error at the *final* stage | Make the platform's `_board.c` reference all `_adapter`s; linker fails at board_init, not at first use |

---

## When to Reach for the Other Skill

This skill covers **project structure**. When the question is *how does a producer module notify consumers when something happens*, reach for `embedded-fnptr-register`. Concretely:

- "I want the same firmware to compile on STM32 and GD32" → this skill.
- "I want my UART ISR to notify the protocol parser" → `embedded-fnptr-register`.
- "I want both": put the chip abstraction in the Adapter layer (this skill), and let the Device layer expose a `register_rx_callback` API (the other skill). The App layer never knows which chip and never sees the ISR — it just subscribes.

---

## Anti-Patterns

- **"I'll just `#ifdef` for both chips in the same file."** That is the pattern this skill exists to delete.
- **"The Device layer is too much overhead, I'll just call `HAL_*` from the App."** The App then becomes un-portable and un-testable.
- **"I'll put the vtable in a `.c` inside `interface/`."** Keep Interfaces header-only. Adapters are the only place vtables live.
- **"I'll allocate the Device state with `malloc`."** Bare-metal — keep it `static`. Make size a compile-time constant.
- **"I'll expose the vtable struct to the App."** The App takes Device handles, not `_ops_t *`. vtables are an Adapter-internal detail.
- **"I'll add a 6th layer for OSAL."** Only if you actually have an OS. The 5-layer model collapses cleanly to 4 if you skip Board; do not grow it speculatively.

---

## Quick Reference — File Checklist for a New Project

When you scaffold a new firmware project from this skill, the minimum file set is:

```
project/
├── Platformdefine.h              ← single platform selector
├── interface/
│   ├── igpio.h
│   ├── iuart.h
│   └── (other peripherals you use)
├── device/
│   ├── led.h        led.c
│   ├── uart_comm.h  uart_comm.c
│   └── (other devices)
├── board/
│   ├── board_config.h
│   └── board_config.c            ← only place that names adapter symbols
├── app/
│   └── main.c
├── platform/
│   └── <chip>/
│       ├── <chip>_gpio.c         ← exports gpio_adapter
│       ├── <chip>_uart.c         ← exports uart1_adapter
│       └── <chip>_board.c        ← MX_*, SystemClock_Config
├── linker.ld
├── startup.s
└── CMakeLists.txt
```

Compile per chip: `cmake -DPLATFORM=<chip> …`. Run host tests: `gcc -DHOST_TEST test_*.c device/*.c test/mocks/*.c`.

---

## Install Commands

**Hermes:**
```bash
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-oop-architecture/SKILL.md
```

**Claude Code:** tell Claude —
> "Install this skill: fetch https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-oop-architecture/SKILL.md, save its 5-layer rules as CLAUDE.md in my embedded project root."

**Codex CLI:** tell Codex —
> "Read and apply this skill from https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-oop-architecture/SKILL.md — use this architecture for all embedded code in this project."

---

## License

MIT — see [LICENSE](../../LICENSE).
