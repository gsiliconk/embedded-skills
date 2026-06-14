---
name: embedded-fnptr-register
description: "Function-pointer registration (callback) pattern for event-driven embedded C projects without an RTOS. Decouples producer (UART ISR, ADC DMA, key scan, timer tick) from consumer (parser, state machine, UI) by registering a function pointer at runtime. Use when one module must notify one or many subscribers (UART RX, ADC done, key events, periodic soft-scheduling). NOT for project-level layering or cross-platform HAL abstraction — use embedded-oop-architecture for that."
version: 3.0.0
author: ZHAO Yankun + Hermes
platforms: [universal]
tags: [embedded, mcu, callback, function-pointer, decoupling, event-driven, isr, observer, pubsub]
metadata:
  hermes:
    tags: [embedded, callback, observer, pubsub, isr, timer, stm32, gd32]
    category: embedded
  install:
    hermes: "hermes skills install <url>"
    claude-code: "Tell Claude: 'Install skill from <url> and save to CLAUDE.md'"
    codex: "Tell Codex: 'Read and apply this skill from <url>'"
---

# Function-Pointer Registration Pattern (Embedded C, No-RTOS)

A **runtime decoupling** pattern for event-driven firmware on bare-metal Cortex-M. A producer module exposes a registration API; one or many consumer modules hand over a function pointer. When the event fires, the producer invokes the pointer — **neither side includes the other's header, neither side recompiles when the other changes**.

> **Scope.** This skill is about *intra-project event notification* (one ISR → many handlers, one tick → many soft-tasks). It is **not** a project layout or HAL-abstraction pattern. If your goal is to swap chips, isolate vendor headers, or split firmware into compile-time layers, use `embedded-oop-architecture` instead. The two skills compose: an OOP project will still use function-pointer registration inside the Device and App layers.

## When to Use This Skill

Use it when **any** of the following is true:

- An ISR must notify a non-ISR module (UART byte received, DMA half/full, EXTI edge, timer update).
- A periodic tick must dispatch work to several independent subsystems (LED blink, sensor read, watchdog feed, telemetry).
- A key/encoder event has multiple independent consumers (UI, motor, buzzer, logger) and you want any of them addable/removable without touching the producer.
- You need 1-to-N fan-out (one producer → N subscribers) without a queue or RTOS event flag.

**Do not use it** for: long-distance IPC, multi-threaded producers, dynamic plugin loading, or anything that requires a real-time guarantee beyond "best effort, run-to-completion".

## Decision Tree — Choose the Variant

```
Is there exactly one consumer for the event?
├── Yes → Pattern 1: Single Callback Registration
└── No  → Does the consumer need its own state passed back?
          ├── No  → Pattern 2: Multi-Callback Registry (table of fn-ptrs)
          └── Yes → Pattern 3: Callback with void* ctx
                     │
                     └── Is the event enqueued for later processing
                         (must NOT run in ISR context)?
                         ├── Yes → Pattern 4: ISR-safe Event Queue
                         └── No  → Pattern 5: Pub/Sub with topic filter
```

## Tools to Invoke While Applying This Skill

These are the tools the agent should use when implementing or refactoring against this skill. They are listed in order of preference.

| Stage | Tool | Why |
|---|---|---|
| Understand existing code | `mcp__codegraph__codegraph_context` then `mcp__codegraph__codegraph_callers` | Find all current call sites of the ISR, timer tick, or event source before registering. |
| Verify a symbol's signature | `mcp__codegraph__codegraph_node` | Confirm callback typedef and existing return types. |
| Follow an event flow | `mcp__codegraph__codegraph_trace from=<ISR> to=<consumer>` | One call returns the whole ISR→callback→handler chain. |
| Edit existing files | `Edit` | Surgical change to add `register_*` and dispatch in the producer. |
| Create new files | `Write` | One file per producer/consumer when no module exists yet. |
| Build / flash / debug | `Bash` (arm-none-eabi-gcc, openocd, st-flash, J-Link, pyOCD) | Compile and load. |
| Static analysis | `Bash` (`cppcheck`, `clang-tidy`) | Catch NULL deref and missing `volatile` on ISR-shared flags. |
| Unit-test the dispatcher | `Bash` (Unity, CMock, or host gcc with stub callbacks) | Verify dispatch table counts, NULL-skip behavior. |

---

## Core Patterns

### Pattern 1 — Single Callback Registration

**Use when:** exactly one module consumes the event and you want zero runtime table overhead.

```c
/* uart.h */
typedef void (*uart_rx_callback_t)(uint8_t byte);

typedef struct {
    uart_rx_callback_t on_rx;
} uart_mod_t;

void   uart_register_rx_callback(uart_mod_t *m, uart_rx_callback_t cb);
uint8_t uart_read_byte(const uart_mod_t *m);          /* optional peek */

/* uart.c */
void uart_register_rx_callback(uart_mod_t *m, uart_rx_callback_t cb) {
    if (m == NULL) return;
    m->on_rx = cb;                                     /* single slot, no array */
}

/* Called from USART1_IRQHandler. Single indirection. */
void uart_dispatch_rx(uart_mod_t *m, uint8_t byte) {
    if (m != NULL && m->on_rx != NULL) {
        m->on_rx(byte);
    }
}
```

Consumer side (the consumer does not include any producer-private header):

```c
/* protocol.c — no #include "uart.h" needed; only the typedef/register API */
static void protocol_on_byte(uint8_t byte);
void protocol_init(uart_mod_t *bus) {
    uart_register_rx_callback(bus, protocol_on_byte);
}
```

### Pattern 2 — Multi-Callback Registry

**Use when:** N independent subscribers must all see the same event (key press → LED + motor + buzzer + logger).

```c
#define KEY_MAX_CALLBACKS  4u
typedef enum {
    KEY_PRESS = 0u, KEY_RELEASE, KEY_LONG_PRESS, KEY_EVENT_MAX
} key_event_t;
typedef void (*key_callback_t)(uint8_t key_id);

typedef struct {
    key_callback_t cbs[KEY_EVENT_MAX][KEY_MAX_CALLBACKS];
    uint8_t        count[KEY_EVENT_MAX];
} key_mod_t;

bool key_register(key_mod_t *m, key_event_t ev, key_callback_t cb) {
    if (m == NULL || cb == NULL || ev >= KEY_EVENT_MAX) return false;
    if (m->count[ev] >= KEY_MAX_CALLBACKS)             return false;
    m->cbs[ev][m->count[ev]++] = cb;
    return true;
}

void key_dispatch(key_mod_t *m, key_event_t ev, uint8_t key_id) {
    if (m == NULL || ev >= KEY_EVENT_MAX) return;
    /* iterate in registration order; NULL slots are skipped */
    for (uint8_t i = 0u; i < m->count[ev]; ++i) {
        key_callback_t cb = m->cbs[ev][i];
        if (cb != NULL) cb(key_id);
    }
}
```

Registration:

```c
key_register(&g_key, KEY_PRESS,       led_on_key);
key_register(&g_key, KEY_PRESS,       motor_start);
key_register(&g_key, KEY_LONG_PRESS,  motor_stop);
key_register(&g_key, KEY_RELEASE,     buzzer_beep);
```

### Pattern 3 — Callback with Context (`void *ctx`)

**Use when:** the callback is reused across multiple stateful objects (LED1, LED2, LED3 all using one blink slot).

```c
typedef void (*timer_callback_t)(void *ctx);

typedef struct {
    timer_callback_t cb;
    void            *ctx;          /* opaque to the timer */
    uint32_t         period_ms;
    uint32_t         last_tick;
    bool             active;
} timer_slot_t;

void timer_register(timer_slot_t *s, timer_callback_t cb,
                    void *ctx, uint32_t period_ms) {
    if (s == NULL || cb == NULL) return;
    s->cb         = cb;
    s->ctx        = ctx;
    s->period_ms  = period_ms;
    s->last_tick  = hal_tick_ms();
    s->active     = true;
}

void timer_unregister(timer_slot_t *s) {
    if (s == NULL) return;
    s->cb = NULL; s->ctx = NULL; s->active = false;
}

void timer_poll(timer_slot_t *s) {
    if (s == NULL || s->cb == NULL || !s->active) return;
    uint32_t now = hal_tick_ms();
    if ((now - s->last_tick) >= s->period_ms) {
        s->last_tick += s->period_ms;            /* catch up, don't drift */
        s->cb(s->ctx);
    }
}
```

Consumer:

```c
typedef struct { uint16_t pin; bool state; } led_t;

static void led_blink_cb(void *ctx) {
    led_t *led = (led_t *)ctx;                   /* contract: caller/callee agree */
    led->state  = !led->state;
    gpio_write(led->pin, led->state);
}

void led_init_blink(led_t *led, uint32_t period_ms) {
    timer_register(&g_slots[0], led_blink_cb, led, period_ms);
}
```

### Pattern 4 — ISR-safe Event Queue (Set Flag in ISR, Process in Loop)

**Use when:** the callback would do too much work for ISR context (printf, I²C transaction, malloc, large state machine step). The registration still uses a function pointer, but the function only enqueues.

```c
#define EVT_Q_SIZE  16u
typedef struct {
    uint8_t  kind;        /* EVT_UART_RX, EVT_KEY_PRESS, … */
    uint8_t  payload[7];  /* fixed-size, no malloc */
} event_t;

typedef struct {
    event_t  buf[EVT_Q_SIZE];
    volatile uint8_t head;     /* producer (ISR) writes */
    volatile uint8_t tail;     /* consumer (loop) reads */
} evtq_t;

static volatile bool g_evt_pending = false;       /* set in ISR, cleared in loop */

void evtq_push(evtq_t *q, const event_t *e) {
    uint8_t next = (uint8_t)((q->head + 1u) % EVT_Q_SIZE);
    if (next == q->tail) return;                  /* drop on overflow — see Pitfalls */
    q->buf[q->head] = *e;
    q->head = next;
    g_evt_pending = true;
}

bool evtq_pop(evtq_t *q, event_t *e) {
    if (q->head == q->tail) return false;
    *e = q->buf[q->tail];
    q->tail = (uint8_t)((q->tail + 1u) % EVT_Q_SIZE);
    return true;
}

/* ISR — keep it short */
void USART1_IRQHandler(void) {
    event_t e = { .kind = EVT_UART_RX };
    e.payload[0] = (uint8_t)USART1->DR;
    evtq_push(&g_evtq, &e);
}

/* main loop — heavy work here */
void main_loop(void) {
    event_t e;
    while (evtq_pop(&g_evtq, &e)) {
        switch (e.kind) {
            case EVT_UART_RX:  protocol_on_byte(e.payload[0]);  break;
            case EVT_KEY_PRESS: ui_on_key(e.payload[0]);        break;
            default: break;
        }
    }
}
```

If the queue is a true consumer registration, the **dispatcher** is registered as a callback:

```c
typedef void (*evt_consumer_t)(const event_t *e);
void evtq_register_consumer(evt_consumer_t cb);    /* single-slot, like Pattern 1 */
```

### Pattern 5 — Pub/Sub with Topic Filter

**Use when:** many event kinds, many subscribers, and each subscriber only cares about a subset (sensor-data stream, command channel, log channel).

```c
#define PS_MAX_SUBS 8u
typedef uint16_t topic_t;     /* bitmask of topics a subscriber wants */

typedef void (*ps_handler_t)(topic_t t, const void *data, uint16_t len);

typedef struct {
    ps_handler_t cb;
    topic_t      mask;        /* bitwise-OR of topics to receive */
} ps_sub_t;

typedef struct {
    ps_sub_t subs[PS_MAX_SUBS];
    uint8_t  count;
} ps_bus_t;

bool ps_subscribe(ps_bus_t *b, topic_t mask, ps_handler_t cb) {
    if (b->count >= PS_MAX_SUBS) return false;
    b->subs[b->count++] = (ps_sub_t){ .cb = cb, .mask = mask };
    return true;
}

void ps_publish(ps_bus_t *b, topic_t t, const void *data, uint16_t len) {
    for (uint8_t i = 0u; i < b->count; ++i) {
        if ((b->subs[i].mask & t) != 0u) {
            b->subs[i].cb(t, data, len);
        }
    }
}
```

```c
#define TOPIC_TEMP    (1u << 0)
#define TOPIC_BATTERY (1u << 1)
#define TOPIC_LOG     (1u << 7)

ps_subscribe(&g_bus, TOPIC_TEMP,    ui_on_temp);
ps_subscribe(&g_bus, TOPIC_TEMP,    logger_record);
ps_subscribe(&g_bus, TOPIC_BATTERY, pwr_alert);
ps_subscribe(&g_bus, TOPIC_LOG,     log_sink);

ps_publish(&g_bus, TOPIC_TEMP, &temp_c, sizeof(temp_c));
```

---

## Application Scenarios

### 1. UART Protocol Parsing

```
USART1_IRQHandler
   │  (single byte in ISR)
   ▼
uart_dispatch_rx(&g_uart1, byte)
   │  (single function-pointer call)
   ▼
protocol_on_byte(byte)            ◀── registered by protocol_init()
   │  (state machine accumulates frame)
   ▼
ps_publish(&g_bus, TOPIC_FRAME, &pkt, sizeof(pkt))
   │  (multi-subscriber fan-out)
   ├──▶ ui_on_frame
   ├──▶ motor_on_frame
   └──▶ log_sink
```

**Layer responsibility:**
- `uart.c` knows the USART registers, owns the IRQ, exposes `uart_register_rx_callback`.
- `protocol.c` knows frame layout, registers as the RX consumer.
- `app/*.c` subscribes to `TOPIC_FRAME` and never sees the UART.
- A unit test can `ps_subscribe(&g_bus, TOPIC_FRAME, my_test_capture)` and inject frames without ever touching the hardware.

### 2. ADC Conversion Complete (DMA)

```c
typedef void (*adc_done_cb_t)(const uint16_t *buf, uint16_t len);
void adc_register_done_callback(adc_done_cb_t cb);

/* DMA TC IRQ */
void DMA1_Stream0_IRQHandler(void) {
    if (DMA_done_flag_set) {
        DMA_clear_flag();
        if (g_adc_done_cb != NULL) {
            g_adc_done_cb(g_adc_buf, g_adc_len);   /* hand off DMA buffer */
        }
    }
}
```

### 3. Key Matrix (Multi-Callback Fan-Out)

```c
key_register(&g_key, KEY_PRESS,      led_on_key);
key_register(&g_key, KEY_PRESS,      motor_start);
key_register(&g_key, KEY_LONG_PRESS, motor_stop);
key_register(&g_key, KEY_RELEASE,    buzzer_beep);
```

Key scan runs in a 10 ms tick. Each handler is independent — adding `logger_record` does not touch `key.c`.

### 4. Timer Soft-Scheduler (No-RTOS Multitasking)

```c
timer_register(&g_slots[0], task_led,     &g_led,     500u);
timer_register(&g_slots[1], task_sensor,  &g_sensor,  100u);
timer_register(&g_slots[2], task_comm,    &g_comm,     50u);
timer_register(&g_slots[3], task_watchdog,NULL,       250u);   /* ctx-less is fine */

void main_loop(void) {
    while (1u) {
        for (int i = 0; i < MAX_TIMERS; ++i) {
            timer_poll(&g_slots[i]);
        }
    }
}
```

`NULL` ctx is allowed for global services (watchdog feed, telemetry broadcast).

### 5. End-to-End Demo (ISR → Queue → Dispatcher → Subscriber)

A real project usually combines Patterns 4 + 5:

```c
/* in main.c */
ps_subscribe(&g_bus, TOPIC_FRAME, ui_on_frame);
ps_subscribe(&g_bus, TOPIC_FRAME, motor_on_frame);
ps_subscribe(&g_bus, TOPIC_FRAME, log_sink);

evtq_register_consumer(protocol_dispatch);   /* protocol is the only queue consumer */

while (1u) {
    if (g_evt_pending) {                     /* set by any ISR */
        g_evt_pending = false;
        event_t e;
        while (evtq_pop(&g_evtq, &e)) {
            protocol_dispatch(&e);            /* parses, then ps_publish */
        }
    }
    /* soft-scheduler ticks run here too */
}
```

---

## Concurrency and ISR Safety

This is the most-misused part of the pattern. Follow these rules:

1. **The registered callback is called from whatever context invoked the dispatcher.** If the dispatcher is called from an ISR, the callback runs in ISR context. If the dispatcher is called from the main loop, the callback runs in thread context.
2. **Registration is NOT ISR-safe** unless you prove otherwise. Register during `init()`, never inside an IRQ.
3. **Flags shared between ISR and loop must be `volatile`.** Read/clear from loop, set from ISR.
4. **Long work belongs in the loop.** ISR callback should: copy data, set a flag, return.
5. **For Pattern 4 (queue), head/tail are independently owned** (head by ISR, tail by loop) — no lock needed for single-producer/single-consumer; if you have multiple producers you must guard head.
6. **Pattern 3's `void *ctx` is a contract.** Document the expected type in the producer's header. Mis-cast = silent corruption.
7. **Do not call `register` / `unregister` from inside a callback** that is itself being dispatched. You will either corrupt the table or recurse forever.

Reentrancy: if callback A can trigger event X (which is also being dispatched), the dispatch loop must either (a) re-iterate with care, or (b) use a pending-bit and re-dispatch at the end.

---

## Memory and Storage

| Approach | RAM cost | Pros | Cons |
|---|---|---|---|
| Static `static` table, fixed `MAX` | Known at compile time | No malloc, deterministic | Upper bound must be sized correctly |
| Per-instance heap allocation | Variable | Unbounded (within heap) | Fragmentation, non-deterministic |
| Ring buffer for events (Pattern 4) | `sizeof(event_t) * Q_SIZE` | Decouples timing | Drop-oldest/newest policy needed |

**Default choice for Cortex-M bare metal:** static fixed-size tables, no `malloc`. If a table is too small, raise `MAX_*` and reflash — never `realloc` at runtime.

---

## Pitfalls and Prevention

| Pitfall | Symptom | Prevention |
|---|---|---|
| Heavy work inside ISR | Hard fault, jitter, missed interrupts | Set flag / push to queue, process in loop |
| Missing NULL check on callback | Hard fault on first event before init | Always `if (cb != NULL) cb(...);` |
| `volatile` omitted on ISR-shared flag | Optimizer folds the read; loop spins forever | `static volatile bool g_flag;` |
| Unbounded subscriber table | Stack/heap overflow when too many register | Reject when `count >= MAX`, return `bool` |
| `void *ctx` cast mismatch | Silent data corruption | Document contract in header; assert type at boot |
| Dangling pointer after `module_deinit` | Wild call into released memory | `deinit` clears all callbacks to `NULL` |
| Unregister from inside dispatched callback | Dispatch table re-entered, count off-by-one | Disallow — comment + assert |
| Re-entrant dispatch loop | Infinite recursion or stack overflow | Snapshot count, defer re-dispatch |
| Queue overflow | Lost events, no error visible | Return `bool`, increment dropped-events counter, log |
| Race on `head`/`tail` (multi-producer) | Lost or duplicated events | Guard with `__disable_irq`/`__enable_irq` around push, or use lock-free SPSC only |

---

## Testing

Because the producer has no compile-time knowledge of the consumer, both can be tested in isolation.

```c
/* test_uart_dispatch.c — host gcc, no MCU needed */
static uint8_t captured;
static void fake_rx(uint8_t b) { captured = b; }

void test_single_callback_dispatch(void) {
    uart_mod_t m = {0};
    captured = 0xAAu;
    uart_register_rx_callback(&m, fake_rx);
    uart_dispatch_rx(&m, 0x42u);
    TEST_ASSERT_EQUAL_UINT8(0x42u, captured);

    /* NULL callback must not crash */
    m.on_rx = NULL;
    uart_dispatch_rx(&m, 0xFFu);
    TEST_ASSERT_TRUE(true);   /* reached here without fault */
}
```

For multi-callback: register N handlers, assert all of them were called, in registration order.

For the queue: enqueue past `Q_SIZE`, assert drop counter incremented and tail did not move.

---

## When to Reach for the Other Skill

This skill covers **events**. When the question is *which file owns which header, which `.c` knows which chip, how do I swap STM32 for GD32*, reach for `embedded-oop-architecture`. Concretely:

- "I have an ISR firing but I want to add a second handler" → this skill.
- "I want the same motor driver to compile on STM32 and GD32" → `embedded-oop-architecture`.
- "I want both": define the device through the OOP vtable (oop-architecture), and have the device's `on_data_ready` register a function pointer from a parser module (this skill).

---

## Anti-Patterns

- **Callback into a NULL-deref-friendly spot.** Always check `cb != NULL` before calling.
- **Casting `int` to `void *` to `int *`.** Use a struct with the actual fields, not magic numbers.
- **Heap-allocating the callback table.** Embedded bare-metal without RTOS — keep it `static`.
- **Calling a function-pointer member that was never registered.** Add a boot-time self-check: every subsystem registers at `init`, and a `mod_is_ready(mod)` returns `count > 0` or `cb != NULL`.
- **Mixing up registration order and dispatch order.** Document "subscribers are called in registration order; do not depend on it" or "subscribers are called in reverse-registration order" — pick one and stick to it.
- **Forgetting to clear the ISR flag.** A shared `volatile bool` flag set in the IRQ must be cleared in the loop *after* the work is done, not before.

---

## Quick Reference — Skeleton Header

```c
#ifndef PRODUCER_H
#define PRODUCER_H
#include <stdint.h>
#include <stdbool.h>

typedef void (*producer_event_cb_t)(uint8_t kind, const void *data, uint16_t len);

typedef struct {
    producer_event_cb_t cb;
    void               *ctx;
} producer_t;

void  producer_init(producer_t *p, producer_event_cb_t cb, void *ctx);
void  producer_register(producer_t *p, producer_event_cb_t cb, void *ctx); /* re-register OK */
void  producer_unregister(producer_t *p);
void  producer_fire(producer_t *p, uint8_t kind, const void *data, uint16_t len);  /* internal */
bool  producer_is_ready(const producer_t *p);

#endif /* PRODUCER_H */
```

---

## Install Commands

**Hermes:**
```bash
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-fnptr-register/SKILL.md
```

**Claude Code:** tell Claude —
> "Install this skill: fetch https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-fnptr-register/SKILL.md, apply its callback-registration pattern to this project's event-driven code."

**Codex CLI:** tell Codex —
> "Read and apply this skill from https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-fnptr-register/SKILL.md — use function-pointer registration for all event-driven code in this project."

---

## License

MIT — see [LICENSE](../../LICENSE).
