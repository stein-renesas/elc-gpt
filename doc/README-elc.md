# ELC Software Building Block

## Overview

This Software Building Block (SBB) demonstrates basic use of the Renesas Event Link Controller (ELC) to connect a software-generated ELC event with timer events. The resulting event chain controls an LED without the application explicitly starting and stopping each timer in response to the generated event.

This document covers only:

- FPB-RA0L1
- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

All five boards are listed as supported by the supplied `readme.txt`.

The underlying timer implementation depends on the MCU capabilities. FPB-RA0L1 uses TAU timers. The supplied source selects GPT timers when `BSP_PERIPHERAL_GPT_PRESENT` is defined.

## What use case does this SBB demonstrate?

The SBB demonstrates peripheral-to-peripheral event routing using ELC. A terminal keypress causes the application to generate `ELC_SOFTWARE_EVENT_0`; the timer behavior that follows is controlled by ELC links configured in the generated FSP project.

### FPB-RA0L1: TAU event chain

For FPB-RA0L1, the supplied documentation describes this sequence:

1. The application generates an ELC software event.
2. The software event starts TAU1.
3. TAU1 runs in Delay Counter mode.
4. The end-of-counter event from TAU1 starts TAU0.
5. TAU0 runs in One-Shot Pulse Output mode.
6. LED1 turns on after 1 second.
7. LED1 remains on for 2 seconds until the requested pulse width expires.

The source maps the first timer to `g_timer1_*` and the second timer to `g_timer0_*` when TAU is selected.

### GPT-based reference configuration

For the GPT timer path, the supplied documentation describes this sequence:

1. The application generates an ELC software event.
2. The software event is the start source for GPT0 and GPT5.
3. GPT0 operates in One-Shot mode.
4. GPT5 operates in Saw-wave PWM mode.
5. GPT0 counter overflow is the stop source for GPT5.
6. The resulting PWM activity blinks the LED.
7. LED blinking stops after 5 seconds when GPT0 expires.

The source maps these roles to `g_timer_oneshot_*` and `g_timer_pwm_*` when GPT is selected.

## Hardware requirements

- 1 x FPB-RA0L1, FPB-RA2E3, FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1
- 1 x appropriate USB cable for programming and debugging

Connect the board USB debug port to the host PC.

### FPB-RA0L1

The supplied README specifies:

- Place jumper **J9 on pins 1-2** to use the on-board debug functionality.
- Close **E20** to use P002 for LED1.
- Connect **P409 (J1:10)** to **P002 (J2:22)** for LED1.

### FPB-RA2E3

The supplied README specifies:

- Close **E25, E28, E46, and E47** to use J-Link On-Board VCOM for the UART terminal.
- Connect **P102 (J4:10)** to **P213 (J3:7)** for LED1.

### FPB-RA4E2

The supplied README lists FPB-RA4E2 as supported but does not specify an additional FPB-RA4E2 hardware connection.

### FPB-RA6E2

The supplied README lists FPB-RA6E2 as supported but does not specify an additional FPB-RA6E2 hardware connection.

### FPB-RA8E1

The supplied README lists FPB-RA8E1 as supported but does not specify an additional FPB-RA8E1 hardware connection.

## What does the SBB core code do?

The reusable SBB layer is implemented by:

- `elc_ep.c`
- `elc_ep.h`

The SBB core:

- Opens the configured ELC instance.
- Enables the configured ELC links.
- Generates `ELC_SOFTWARE_EVENT_0`.
- Closes the ELC instance.
- Initializes the first timer used by the event chain.
- Initializes the second timer used by the event chain.
- Enables external event triggers for both timers.
- Closes the timer instances.
- Abstracts TAU and GPT implementations behind the same SBB functions.

The timer abstraction is selected at compile time from BSP peripheral feature macros.

## Demo/reference application versus reusable SBB code

### Reusable SBB code

`elc_ep.c` and `elc_ep.h` provide the reusable ELC/timer abstraction. This layer owns the ELC open/enable/software-event calls and maps the generic timer helper functions onto TAU or GPT APIs.

### Demo/reference application code

`elc_example_app.c` and `elc_example_app.h` provide the terminal-driven demonstration. They:

- Initialize the UART console.
- Retrieve and display the FSP version.
- Display timer-mode information appropriate to the selected timer implementation.
- Initialize and enable ELC.
- Initialize and enable both timer instances.
- Wait for terminal input.
- Generate an ELC software event for every accepted keypress.
- Print the user prompt again after each generated event.
- Report errors, close the timer and ELC instances, flush the console, and enter a breakpoint.

The demo entry point is:

```c
void elc_example_app_run(void);
```

The source identifies it as the single function called from `hal_entry.c`.

## Which public functions should I look at first?

### `elc_init()`

```c
fsp_err_t elc_init(void);
```

Opens the generated ELC instance through `R_ELC_Open()` using `g_elc_ctrl` and `g_elc_cfg`.

### `elc_enable()`

```c
fsp_err_t elc_enable(void);
```

Enables the configured ELC links using `R_ELC_Enable()`.

### `elc_event_gen()`

```c
fsp_err_t elc_event_gen(void);
```

Generates `ELC_SOFTWARE_EVENT_0` using `R_ELC_SoftwareEventGenerate()`. This is the application-visible trigger used to start the demonstrated event chain.

### `oneshot_timer_init()`

```c
fsp_err_t oneshot_timer_init(void);
```

Initializes the first timer role. With TAU it opens `g_timer1_ctrl/g_timer1_cfg`; with GPT it opens `g_timer_oneshot_ctrl/g_timer_oneshot_cfg`.

### `oneshot_timer_enable()`

```c
fsp_err_t oneshot_timer_enable(void);
```

Enables the external-event triggers configured for the first timer role.

### `pwm_timer_init()`

```c
fsp_err_t pwm_timer_init(void);
```

Initializes the second timer role. With TAU it uses the TAU PWM interface for `g_timer0_*`; with GPT it opens `g_timer_pwm_*` through the GPT interface.

### `pwm_timer_enable()`

```c
fsp_err_t pwm_timer_enable(void);
```

Enables the external-event triggers configured for the second timer role.

### Cleanup functions

```c
void elc_deinit(void);
void oneshot_timer_deinit(void);
void pwm_timer_deinit(void);
```

These close the ELC and timer instances when open.

## What configuration assumptions were made for the reference boards?

### ELC configuration

The SBB expects generated ELC objects:

```text
g_elc_ctrl
g_elc_cfg
```

The actual event links are provided by the generated ELC configuration. The hand-written layer opens that configuration, enables it, and generates software event 0.

### TAU configuration

When `BSP_PERIPHERAL_TAU_PRESENT` is selected, `elc_ep.h` defines:

- Timer 1: **TAU1**, Delay Counter mode.
- Timer 2: **TAU0**, One-Shot Pulse Output mode.
- First timer configuration: `g_timer1_ctrl` / `g_timer1_cfg`.
- Second timer configuration: `g_timer0_ctrl` / `g_timer0_cfg`.

The supplied README explicitly identifies FPB-RA0L1 as using this TAU behavior.

### GPT configuration

When `BSP_PERIPHERAL_GPT_PRESENT` is selected, `elc_ep.h` defines:

- Timer 1: **GPT0**, One-Shot mode.
- Timer 2: **GPT5**, Saw-wave PWM mode.
- First timer configuration: `g_timer_oneshot_ctrl` / `g_timer_oneshot_cfg`.
- Second timer configuration: `g_timer_pwm_ctrl` / `g_timer_pwm_cfg`.

The generated configuration supplies the external start/stop event relationships used when the timer `Enable` calls are made.

### Demonstrated timing

The documentation states:

- TAU path: LED1 turns on after **1 second** and remains on for **2 seconds**.
- GPT path: the LED blinks for **5 seconds** and then stops when the One-Shot timer expires.

These values describe the supplied timer configuration.

### Terminal input

The demonstration accepts any available terminal character as the request to generate another ELC software event. The source removes one character with `getchar()` before calling `elc_event_gen()`.

## What must I change for my board, pins, clocks, timing, memory, task priorities, or middleware setup?

### ELC links

The SBB depends on the links in `g_elc_cfg`. If the producer event, target peripheral, start event, or stop event changes, update the generated ELC configuration so that it matches the intended event chain.

### Timer implementation

The source supports TAU and GPT through compile-time BSP feature selection. The generated timer instances must match the macros selected by `elc_ep.h`.

For the TAU path the SBB expects `g_timer1_*` and `g_timer0_*`. For the GPT path it expects `g_timer_oneshot_*` and `g_timer_pwm_*`.

### Pins and LED routing

FPB-RA0L1 and FPB-RA2E3 require the external connections documented above. If the timer output or LED pin changes, update the board pin configuration and any required external connection accordingly.

The supplied README does not provide additional pin-routing instructions for FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1.

### Timer timing

The 1-second delay, 2-second output pulse, and 5-second GPT sequence are supplied reference behaviors. If different timing is needed, change the applicable generated timer configuration while preserving the required ELC start/stop relationships.

### Clocks

The supplied files do not state board-specific TAU/GPT source-clock frequencies or clock-divider settings for the five boards. Changes to clock configuration must therefore be validated against the generated timer configuration and desired timing.

### Memory

The supplied source does not provide a measured RAM or ROM footprint for the SBB.

### Task priorities and execution model

The reference application does not define an RTOS or task priorities. It remains in a foreground loop waiting for terminal input. The demonstrated timer-to-timer behavior is configured through ELC rather than being sequenced by foreground application calls after the initial software event.

### Console middleware

The console is used only by the reference application to initiate demonstrations and display information/errors. The reusable ELC/timer layer does not require the terminal menu itself.

## Dependencies

### Renesas FSP ELC driver

The SBB directly uses:

- `R_ELC_Open()`
- `R_ELC_Enable()`
- `R_ELC_SoftwareEventGenerate()`
- `R_ELC_Close()`

### TAU path

When `BSP_PERIPHERAL_TAU_PRESENT` is selected, the abstraction uses:

- `R_TAU_Open()`
- `R_TAU_Close()`
- `R_TAU_Enable()`
- `R_TAU_PWM_Open()`
- `R_TAU_PWM_Close()`
- `R_TAU_PWM_Enable()`

### GPT path

When `BSP_PERIPHERAL_GPT_PRESENT` is selected, the abstraction uses:

- `R_GPT_Open()`
- `R_GPT_Close()`
- `R_GPT_Enable()`

### Generated FSP configuration

The SBB includes `hal_data.h` and relies on generated ELC and timer control/configuration objects.

### Console/reference application

The demo uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- `printf()`
- `getchar()`
- `R_FSP_VersionGet()`
- `__BKPT()` for its error trap

## Serial terminal configuration

The supplied README specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## What behavior and performance were demonstrated or measured?

### Demonstrated behavior

The supplied source and documentation demonstrate:

- Opening and enabling the ELC.
- Generating `ELC_SOFTWARE_EVENT_0` in response to terminal input.
- Starting timer activity from an ELC software event.
- Chaining timer events through ELC.
- Using TAU Delay Counter and One-Shot Pulse Output modes on the TAU path.
- Using GPT One-Shot and Saw-wave PWM modes on the GPT path.
- Controlling visible LED behavior as the output of that event chain.

### Demonstrated timing

For FPB-RA0L1's documented TAU path, LED1 turns on after 1 second and stays on for 2 seconds.

For the documented GPT path, LED blinking stops after 5 seconds when GPT0 expires.

### Performance measurements

The supplied files do **not** report measured ELC propagation latency, timer accuracy, PWM frequency, jitter, interrupt latency, CPU utilization, or memory footprint.

The 1-second, 2-second, and 5-second values describe configured demonstration behavior, not ELC performance measurements. This SBB should therefore be treated as a functional event-routing reference rather than a performance benchmark.

## What files are safe to modify and what files are generated, preset, or demo-only?

| File | Classification | Guidance |
|---|---|---|
| `elc_ep.c` | Reusable SBB implementation | Modify when intentionally changing the ELC wrapper, software-event selection, timer abstraction, or enable/cleanup behavior. |
| `elc_ep.h` | Reusable SBB interface and timer abstraction | Modify carefully when changing public functions, TAU/GPT mappings, timer instances, or timer modes. |
| `elc_example_app.c` | Demo/reference application | Appropriate location for changing console interaction, application sequencing, presentation, and error handling. |
| `elc_example_app.h` | Demo/reference application interface | Contains the demo entry point and TAU/GPT-specific informational/status text. |
| `readme.txt` | Existing example documentation | Source for supported boards, board-specific hardware setup, demonstrated ELC relationships, timing, and terminal settings. |

The SBB depends on generated configuration through `hal_data.h`. The supplied file set does not enumerate the generated project files themselves, so those generated files should be treated separately from the hand-maintained SBB and reference-application sources.

## What happens if I modify the source and later adopt a newer SBB release?

Local modifications to `elc_ep.c` or `elc_ep.h` create differences in the reusable SBB layer that may overlap a future SBB update.

To reduce upgrade conflicts:

- Keep terminal prompts, display strings, and application-level sequencing in `elc_example_app.*`.
- Prefer the generated FSP configuration for ELC link selection, timer modes, timer periods, channels, and pin configuration when those are configuration concerns.
- Keep changes to the TAU/GPT abstraction in `elc_ep.h` focused and documented.
- Keep generated FSP files separate from hand-maintained SBB code.
- When adopting a newer SBB release, compare the new `elc_ep.c` and `elc_ep.h` with the locally modified versions before carrying changes forward.
- Revalidate ELC software-event routing, timer start/stop event links, timer implementation selection, hardware pin routing, and demonstrated LED timing after integration.

The supplied files do not define an automated source-migration or compatibility mechanism for later SBB releases. Compatibility must therefore be validated by the integrating project.

## Recommended reading order

1. **`elc_ep.h`** to understand the TAU/GPT abstraction, generated timer object mappings, modes, and public SBB functions.
2. **`elc_ep.c`** to see the ELC open/enable/software-event operations and timer initialization/enable wrappers.
3. **`elc_example_app.c`** to understand initialization order and how terminal input causes a software ELC event.
4. **`elc_example_app.h`** to see the TAU/GPT-specific demonstration behavior and application entry point.
5. **`readme.txt`** for the original supported-board list, FPB-RA0L1/FPB-RA2E3 hardware connections, timing description, and terminal configuration.
