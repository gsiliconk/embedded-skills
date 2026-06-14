# Embedded C Decoupling Skills

A two-skill pack for **bare-metal Cortex-M firmware in C**, designed to compose:

- **`embedded-oop-architecture`** — compile-time project structure & HAL isolation.
- **`embedded-fnptr-register`** — runtime event notification between modules.

Each skill solves a different axis of "decoupling". They do not overlap; they are designed to be used together. Pick the right one with the decision matrix below.

---

## Which Skill Do I Need?

| Your question | Use this skill |
|---|---|
| "I want to swap STM32 for GD32 without rewriting the App" | `embedded-oop-architecture` |
| "I want a clean 5-layer project: Interface / Adapter / Device / Board / App" | `embedded-oop-architecture` |
| "I want to unit-test a Device driver on my laptop with no MCU" | `embedded-oop-architecture` |
| "My ISR fires and N modules need to react" | `embedded-fnptr-register` |
| "I want a periodic tick to drive a soft scheduler (no RTOS)" | `embedded-fnptr-register` |
| "I want a key press to fan out to LED + motor + buzzer + logger" | `embedded-fnptr-register` |
| "I want UART byte → protocol parser → UI without header coupling" | `embedded-fnptr-register` |
| "I want both: a portable Device layer that also publishes events" | **Both** — see *Composing the two skills* below |

---

## 1. `embedded-oop-architecture` — 5-Layer Virtual-Table OOP

A **compile-time decoupling** pattern. Five layers with strict include rules:

```
┌────────────────────────────────────────────────────────┐
│  App        main.c / *_app.c        Business logic     │
├────────────────────────────────────────────────────────┤
│  Board      board_config.c         Pin map, wiring     │
├────────────────────────────────────────────────────────┤
│  Device     led.c / uart_comm.c    Portable drivers    │
├────────────────────────────────────────────────────────┤
│  Adapter    stm32_gpio.c / gd32_gpio.c  Chip → if       │
├────────────────────────────────────────────────────────┤
│  Interface  igpio.h / iuart.h      Header-only vtable  │
└────────────────────────────────────────────────────────┘
   ▲ Dependencies flow DOWN only. Vendor headers live in Adapter + Platformdefine.h.
```

**Key features:**

- Virtual function tables (`_ops_t`) for runtime polymorphism with zero `malloc`.
- **Single-header platform isolation** — `Platformdefine.h` plus `platform/<chip>/` build slot.
- **Dependency injection** through `board_config` — Device code never names a chip.
- **Switching chips is 3 steps**: change `CURRENT_PLATFORM`, swap build slot, recompile. **Zero edits** to Device / Board / App.
- **Host-side unit testing** of Devices on a PC, no MCU.
- CMake template for multi-platform builds.
- Interface versioning rules (semantic).

See [`skills/embedded-oop-architecture/SKILL.md`](./skills/embedded-oop-architecture/SKILL.md) for the complete walkthrough, naming conventions, and full skeleton.

---

## 2. `embedded-fnptr-register` — Function-Pointer Registration

A **runtime decoupling** pattern. A producer exposes a registration API; one or many consumers hand over a function pointer. When the event fires, the producer invokes the pointer — **neither side includes the other's header, neither side recompiles when the other changes**.

**Five pattern variants**, chosen by event shape:

| # | Pattern | When |
|---|---|---|
| 1 | Single Callback | One consumer per event |
| 2 | Multi-Callback Registry | N independent subscribers, fixed upper bound |
| 3 | Callback with `void *ctx` | One handler reused across many stateful objects |
| 4 | ISR-safe Event Queue | Heavy work must NOT run in ISR context |
| 5 | Pub/Sub with topic filter | Many event kinds, each subscriber wants a subset |

**Key features:**

- **ISR safety** built-in: ISR sets a flag / pushes a queue slot, main loop processes.
- `volatile` discipline for ISR↔loop shared state.
- Reentrancy rules for callbacks that trigger other events.
- Host-side unit tests of dispatchers with stub callbacks.
- `volatile bool` + ring buffer = single-producer / single-consumer queue, no RTOS, no lock.

See [`skills/embedded-fnptr-register/SKILL.md`](./skills/embedded-fnptr-register/SKILL.md) for the full pattern library, examples (UART, ADC DMA, key matrix, soft-scheduler), and the end-to-end ISR→queue→dispatcher→subscriber demo.

---

## Composing the Two Skills

The two patterns are designed to compose. A typical OOP project's Device layer exposes hardware events through callback registration, and the App layer subscribes to those events. Neither layer knows the chip; neither layer knows the parser's internals.

```
platform/stm32/stm32_uart.c     ── adapter: implements iuart_ops
        │
        ▼
device/uart_comm.c              ── device:  owns state, calls bus->ops->read(),
        │                                 also: uart_comm_register_rx(cb, ctx)
        │                                 (← fnptr-register API)
        ▼
app/protocol.c                  ── subscribes via register_rx; never sees HAL
        │
        ▼
app/main.c                      ── poll loop, soft-scheduler, system tick
```

When to introduce `embedded-fnptr-register` **inside** an OOP project:

- A Device has a "data ready" or "frame complete" event with multiple consumers.
- A periodic tick must dispatch work to several independent subsystems.
- You want a parser to be unit-testable on the host (subscribe a mock callback, no UART involved).

---

## Tools Used by These Skills

Both skills are designed to be applied with the following toolchain. The agent should use CodeGraph for structural questions and reserve `Read` / `Edit` / `Write` / `Bash` for confirmation and edits.

| Task | Tool | Why |
|---|---|---|
| Find all callers of a HAL symbol or ISR | `mcp__codegraph__codegraph_callers` | Locate leaks before refactoring |
| Trace a flow (ISR → callback, or pin → GPIO register) | `mcp__codegraph__codegraph_trace` | One call returns the whole chain |
| Verify a vtable / callback signature | `mcp__codegraph__codegraph_node` | Confirm typedefs and field order |
| Audit directory layout | `mcp__codegraph__codegraph_files` | Verify the 5-layer tree |
| Refactor an existing file | `Read` then `Edit` | Surgical change |
| Create a new layer / module | `Write` | One file per role |
| Build / flash / debug | `Bash` (arm-none-eabi-gcc, openocd, st-flash) | Compile and load |
| Host-side unit test | `Bash` (host gcc + Unity / CMock) | No MCU required |
| Static analysis | `Bash` (cppcheck, clang-tidy) | NULL-deref and missing-`volatile` guards |

---

## Install

### Hermes

```bash
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-oop-architecture/SKILL.md
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-fnptr-register/SKILL.md
```

### Claude Code

Tell Claude:

> Install these two skills from <https://github.com/gsiliconk/embedded-skills>:
>
> 1. `skills/embedded-oop-architecture/SKILL.md` — apply its 5-layer rules as my embedded project's architecture.
> 2. `skills/embedded-fnptr-register/SKILL.md` — apply its callback-registration pattern to all event-driven code in this project.

### Codex CLI

Tell Codex:

> Read and apply both skills from <https://github.com/gsiliconk/embedded-skills>:
>
> 1. `skills/embedded-oop-architecture/SKILL.md` — 5-layer architecture.
> 2. `skills/embedded-fnptr-register/SKILL.md` — function-pointer registration for events.

---

## Repository Layout

```
embedded-skills/
├── README.md
├── LICENSE
└── skills/
    ├── embedded-oop-architecture/
    │   └── SKILL.md        ← 5-layer OOP, platform isolation
    └── embedded-fnptr-register/
        └── SKILL.md        ← callback registration, event dispatch
```

---

## Versioning

Each skill tracks its own `version` in its frontmatter. A bump means:

- **Major** (3.0.0 → 4.0.0) — breaking rule changes (renamed naming convention, layer model change, removed pattern variant).
- **Minor** (3.0.0 → 3.1.0) — new pattern, new example, expanded section, no breaking changes to existing rules.
- **Patch** (3.0.0 → 3.0.1) — typo fix, link fix, code-comment fix.

The two skills are versioned **independently** — updating one does not require updating the other.

---

## License

This repository is released under the [MIT License](./LICENSE).
