# 嵌入式 C 解耦架构 Skills

一个面向 **Cortex-M 裸机嵌入式 C** 固件的双 skill 包，两者设计为可组合使用：

- **`embedded-oop-architecture`** —— 编译期的项目结构与 HAL 隔离。
- **`embedded-fnptr-register`** —— 运行时模块间事件通知。

两个 skill 各自解决"解耦"的不同维度，互不重叠，专门设计为一起使用。可以通过下面的决策矩阵选择合适的 skill。

---

## 该用哪个 Skill？

| 你的问题 | 用这个 skill |
|---|---|
| "我想换芯片（STM32 → GD32）而不用重写 App 层" | `embedded-oop-architecture` |
| "我想要一个清晰的 5 层项目结构：Interface / Adapter / Device / Board / App" | `embedded-oop-architecture` |
| "我想在 PC 上单元测试 Device 驱动，不需要 MCU" | `embedded-oop-architecture` |
| "我的 ISR 触发后，N 个模块都需要响应" | `embedded-fnptr-register` |
| "我想用周期 tick 驱动软调度（不用 RTOS）" | `embedded-fnptr-register` |
| "我想让一次按键分发到 LED + 电机 + 蜂鸣器 + 日志" | `embedded-fnptr-register` |
| "我想实现 UART 字节 → 协议解析器 → UI，且彼此不耦合" | `embedded-fnptr-register` |
| "我都想要：可移植的 Device 层 + 事件订阅能力" | **两个都用** —— 见下文 *两个 Skill 的组合* |

---

## 1. `embedded-oop-architecture` —— 5 层虚表 OOP

**编译期解耦**模式。5 层结构，严格规定每一层可包含的头文件：

```
┌────────────────────────────────────────────────────────┐
│  App        main.c / *_app.c        业务逻辑            │
├────────────────────────────────────────────────────────┤
│  Board      board_config.c         引脚映射、装配        │
├────────────────────────────────────────────────────────┤
│  Device     led.c / uart_comm.c    可移植驱动            │
├────────────────────────────────────────────────────────┤
│  Adapter    stm32_gpio.c / gd32_gpio.c  芯片 → 接口      │
├────────────────────────────────────────────────────────┤
│  Interface  igpio.h / iuart.h      头文件虚表            │
└────────────────────────────────────────────────────────┘
   ▲ 依赖只允许向下流动。Vendor 头文件只存在于 Adapter + Platformdefine.h。
```

**核心特性：**

- 虚函数表（`_ops_t`）实现运行时多态，零 `malloc`。
- **单文件平台隔离** —— `Platformdefine.h` 加上 `platform/<chip>/` 构建槽位。
- **通过 `board_config` 做依赖注入** —— Device 代码里不出现具体芯片名。
- **切芯片 3 步搞定**：改 `CURRENT_PLATFORM`、换构建槽位、重新编译。**Device / Board / App 三层代码 0 改动**。
- 在 PC 上对 Device 做主机侧单元测试，无需 MCU。
- 支持多平台构建的 CMake 模板。
- 接口版本管理规则（语义化版本）。

完整 5 步骨架、命名规范、平台切换流程见 [`skills/embedded-oop-architecture/SKILL.md`](./skills/embedded-oop-architecture/SKILL.md)。

---

## 2. `embedded-fnptr-register` —— 函数指针注册模式

**运行时解耦**模式。生产者暴露注册 API；一个或多个消费者提交函数指针。当事件触发时，生产者通过指针调用 —— **两端互不包含对方的头文件、互不因对方变更而重编译**。

**5 种模式变体**，按事件形态选择：

| # | 模式 | 适用场景 |
|---|---|---|
| 1 | 单回调注册 | 一个事件一个消费者 |
| 2 | 多回调注册表 | N 个独立订阅者，固定上限 |
| 3 | 带 `void *ctx` 的回调 | 同一处理函数被多个有状态对象复用 |
| 4 | ISR-safe 事件队列 | 重活绝对不能跑在 ISR 上下文 |
| 5 | 带 topic 过滤的发布订阅 | 事件种类多，每个订阅者只关心子集 |

**核心特性：**

- **ISR 安全**内置：ISR 只置标志位 / 入队，主循环处理。
- `volatile` 纪律：ISR 与主循环共享状态必须加 `volatile`。
- 重入规则：避免回调触发的事件再次分发出栈。
- 用桩回调在主机上对分发器做单元测试。
- `volatile bool` + 环形缓冲 = 单生产者/单消费者队列，无 RTOS、无锁。

完整模式库、示例（UART / ADC DMA / 按键矩阵 / 软调度）、ISR→队列→分发器→订阅者端到端示例见 [`skills/embedded-fnptr-register/SKILL.md`](./skills/embedded-fnptr-register/SKILL.md)。

---

## 两个 Skill 的组合

两个模式本身就是为组合而设计的。典型 OOP 项目的 Device 层通过回调注册暴露硬件事件，App 层订阅这些事件，两层都不知道芯片、两层都不知道解析器内部细节。

```
platform/stm32/stm32_uart.c     ── adapter: 实现 iuart_ops
        │
        ▼
device/uart_comm.c              ── device:  拥有状态，调用 bus->ops->read()，
        │                                 同时: uart_comm_register_rx(cb, ctx)
        │                                 (← fnptr-register API)
        ▼
app/protocol.c                  ── 通过 register_rx 订阅；从不接触 HAL
        │
        ▼
app/main.c                      ── 主循环、软调度、系统 tick
```

OOP 项目中**何时引入** `embedded-fnptr-register`：

- Device 拥有"数据就绪"或"帧完成"事件，且有多个消费者。
- 周期 tick 需要把工作分派到多个独立子系统。
- 希望解析器可以在主机侧做单元测试（订阅一个 mock 回调，不涉及 UART）。

---

## Skill 实施时调用的工具

两个 skill 都按以下工具链设计。结构性问题优先用 CodeGraph，`Read` / `Edit` / `Write` / `Bash` 用于确认和实际改动。

| 任务 | 工具 | 用途 |
|---|---|---|
| 查找 HAL 符号或 ISR 的所有调用方 | `mcp__codegraph__codegraph_callers` | 重构前定位泄漏点 |
| 追踪数据流（ISR → 回调，或引脚 → GPIO 寄存器） | `mcp__codegraph__codegraph_trace` | 一次调用返回完整链路 |
| 确认虚表 / 回调签名 | `mcp__codegraph__codegraph_node` | 验证 typedef 和字段顺序 |
| 审计目录结构 | `mcp__codegraph__codegraph_files` | 验证 5 层目录树 |
| 重构现有文件 | `Read` 然后 `Edit` | 外科手术式改动 |
| 新建层 / 模块 | `Write` | 一层一个文件 |
| 编译 / 烧录 / 调试 | `Bash`（arm-none-eabi-gcc, openocd, st-flash） | 编译和烧录 |
| 主机侧单元测试 | `Bash`（主机 gcc + Unity / CMock） | 不需要 MCU |
| 静态分析 | `Bash`（cppcheck, clang-tidy） | NULL 解引用 / 漏 `volatile` 守卫 |

---

## 安装

### Hermes

```bash
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-oop-architecture/SKILL.md
hermes skills install https://raw.githubusercontent.com/gsiliconk/embedded-skills/master/skills/embedded-fnptr-register/SKILL.md
```

### Claude Code

对 Claude 说：

> 请安装这两个 skill（来自 <https://github.com/gsiliconk/embedded-skills>）：
>
> 1. `skills/embedded-oop-architecture/SKILL.md` —— 把它的 5 层规则应用为我嵌入式项目的架构规范。
> 2. `skills/embedded-fnptr-register/SKILL.md` —— 把它的回调注册模式应用到本项目所有事件驱动代码。

### Codex CLI

对 Codex 说：

> 请读取并应用 <https://github.com/gsiliconk/embedded-skills> 中的两个 skill：
>
> 1. `skills/embedded-oop-architecture/SKILL.md` —— 5 层架构。
> 2. `skills/embedded-fnptr-register/SKILL.md` —— 用于事件的函数指针注册。

---

## 仓库结构

```
embedded-skills/
├── README.md
├── LICENSE
└── skills/
    ├── embedded-oop-architecture/
    │   └── SKILL.md        ← 5 层 OOP、平台隔离
    └── embedded-fnptr-register/
        └── SKILL.md        ← 回调注册、事件分发
```

---

## 版本管理

每个 skill 在 frontmatter 中维护自己的 `version`。版本号变更语义如下：

- **Major**（3.0.0 → 4.0.0）—— 破坏性规则变更（重命名规范、层模型变更、移除模式变体）。
- **Minor**（3.0.0 → 3.1.0）—— 新增模式、新增示例、扩展章节，不破坏现有规则。
- **Patch**（3.0.0 → 3.0.1）—— 错别字、链接、代码注释修复。

两个 skill **独立**版本号 —— 升级其中一个不要求升级另一个。

---

## 许可证

本仓库基于 [MIT License](./LICENSE) 发布。
