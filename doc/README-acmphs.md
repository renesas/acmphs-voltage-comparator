# ACMPHS (Voltage Comparator)

## Overview

This Software Building Block (SBB), referred to in the source as the ACMPHS Example Project, demonstrates use of the Renesas Flexible Software Package (FSP) high-speed analog comparator (ACMPHS) APIs on the FPB-RA8E1. The reference application uses the ACMPHS internal reference voltage as the comparison reference and DAC0 to generate the analog signal presented to AN000. The comparator result is reported over the console and reflected on the on-board LED.

On FPB-RA8E1, the internal reference voltage is configured to **1.18 V** (described by the original example as equivalent to digital value **1465**). DAC0 provides the variable input. When the input voltage is above the reference, the comparator output is HIGH and the LED is turned ON. When it is below the reference, the comparator output is LOW and the LED is turned OFF.

## What use case does this SBB demonstrate?

The SBB demonstrates a basic threshold-detection use case using the FPB-RA8E1 high-speed analog comparator:

1. Initialize DAC0 and the ACMPHS peripheral.
2. Use DAC0 to generate a controllable analog voltage.
3. Route DAC0 output externally to the ACMPHS AN000 input.
4. Compare AN000 against the ACMPHS internal 1.18 V reference.
5. Read the comparator state after its required stabilization interval.
6. Report the result through the UART-backed standard I/O console.
7. Drive the FPB-RA8E1 on-board LED according to the comparator output.

This provides a concrete reference for applications that need to detect whether an analog signal is above or below a configured threshold.

## Hardware setup

### Required hardware

- 1 x FPB-RA8E1 board
- 1 x USB cable for programming and debugging
- 1 x jumper wire

### FPB-RA8E1 connection

Connect:

- **DA0 P014 (J1:34)** to **IVCMP0_2 / AN000 P004 (J1:20)**

This connection routes the DAC0 output to the comparator analog input used by the reference application.

## Demonstrated operation

The reference application prompts for a DAC0 value in either of these ranges:

- `0` to `1400`
- `1600` to `4095`

Values strictly between `1400` and `1600` are treated as a fluctuating-voltage range and are not used for the comparison operation. The example identifies this exclusion as a way to avoid comparator-output changes caused by electrical noise near the reference threshold.

For an accepted DAC value, the application:

1. Writes the value to DAC0.
2. Calculates and displays its nominal voltage using a **3.3 V** scale and a 12-bit DAC range.
3. Waits for the minimum ACMPHS stabilization time reported by the driver.
4. Refreshes the comparator status.
5. Prints whether the comparator output is HIGH or LOW.
6. Updates the on-board LED.

The example accepts a maximum terminal input buffer of 16 bytes, and the original instructions state that user-entered data should not exceed 15 bytes. Operation is not guaranteed for non-integer terminal input.

## SBB core code

The reusable SBB layer is implemented by `acmphs_ep.c` and `acmphs_ep.h`. It wraps the FSP DAC, ACMPHS, and I/O operations needed by the example instead of placing those driver calls directly in the application loop.

For FPB-RA8E1, the core provides:

- DAC0 initialization, initial write, and start.
- Subsequent DAC0 writes.
- ACMPHS open, information retrieval, and output enable.
- Comparator status refresh.
- On-board LED update from comparator state.
- Peripheral cleanup.
- An ACMPHS callback that marks comparator status for refresh when channel 0 generates a callback.

The core also contains compile-time abstraction macros that map its DAC and comparator operations onto the applicable FSP driver implementation.

## Demo/reference application versus reusable SBB code

### Reusable SBB code

`acmphs_ep.c` and `acmphs_ep.h` contain the SBB-facing peripheral functions and abstractions. These are the files to study first when reusing the comparator behavior in another application.

### Demo/reference application code

`acmphs_example_app.c` and `acmphs_example_app.h` implement the interactive demonstration. This layer is responsible for:

- Initializing the UART console.
- Printing version and example information.
- Prompting for DAC values.
- Parsing terminal input.
- Enforcing the demo's DAC-value ranges.
- Printing the calculated DAC voltage and comparator state.
- Waiting for the comparator stabilization time before checking status.
- Handling errors for the demonstration.
- Running the demonstration continuously.

The application exposes one entry point, `acmphs_example_app_run()`, which is intended to be called by `hal_entry.c`.

The console UI, continuous input loop, printed banner, and demonstration-specific input handling are reference-application behavior rather than the comparator SBB interface itself.

## Public functions to look at first

### `acmphs_ep_comparator_init()`

```c
fsp_err_t acmphs_ep_comparator_init(comparator_info_t * p_stabilize_time);
```

Opens the configured ACMPHS instance, retrieves comparator information including the minimum stabilization wait time, and enables comparator output.

### `acmphs_ep_status_refresh()`

```c
fsp_err_t acmphs_ep_status_refresh(comparator_status_t * p_status);
```

Obtains the comparator status when the internal comparator-state flag indicates that status should be refreshed. After a successful read, the function clears the flag.

### `acmphs_ep_dac0_init()`

```c
fsp_err_t acmphs_ep_dac0_init(uint16_t initial_val);
```

Opens DAC0, writes the initial value, and starts conversion. The FPB-RA8E1 reference application initializes DAC0 to zero before enabling normal interactive operation.

### `acmphs_ep_dac0_write()`

```c
fsp_err_t acmphs_ep_dac0_write(uint16_t dac_val);
```

Writes a new DAC0 value. On FPB-RA8E1, DAC0 supplies the analog signal routed to AN000.

### `acmphs_ep_led_update()`

```c
fsp_err_t acmphs_ep_led_update(comparator_state_t state);
```

Writes the board LED according to the comparator output state. For FPB-RA8E1, `LED_PIN` is configured as `BSP_IO_PORT_04_PIN_08`.

### `acmphs_ep_close()`

```c
void acmphs_ep_close(void);
```

Closes DAC0 if it is open and closes the ACMPHS instance if it is open.

### `acmphs_user_callback()`

```c
void acmphs_user_callback(comparator_callback_args_t * p_args);
```

ACMPHS callback. For comparator channel 0 it sets the internal state flag so that the status can subsequently be refreshed.

## FPB-RA8E1 reference configuration assumptions

The supplied implementation makes the following explicit FPB-RA8E1 assumptions:

- The target is selected with `BOARD_RA8E1_FPB`.
- The ACMPHS internal reference voltage is **1.18 V**.
- Comparator input is **AN000 / IVCMP0_2 on P004**.
- DAC0 output is **DA0 on P014**.
- P014 is externally jumpered to P004.
- DAC0 is used as the controllable analog input source.
- The DAC is treated as 12-bit with maximum value `4095`.
- The application uses `3.3f` as `V_REF` when converting the DAC value to a displayed voltage.
- Values between `1400` and `1600` are excluded by the reference application to avoid the stated noise-sensitive region.
- The LED is `BSP_IO_PORT_04_PIN_08`.
- Comparator control/configuration objects are `g_comparator_ctrl` and `g_comparator_cfg`.
- The application waits for `min_stabilization_wait_us` returned by the comparator driver before checking status.
- Application printing and input use standard I/O through the UART console.

## What must change when porting to another board or application?

The supplied files establish the FPB-RA8E1 reference implementation, but they do not specify a complete generic board-porting procedure. When adapting this SBB, review every application-specific assumption rather than copying the FPB-RA8E1 settings unchanged.

### Pins and analog routing

The FPB-RA8E1 reference depends on P014/DA0 being connected to P004/AN000. A different hardware design must provide an appropriate analog source and comparator input routing. The LED pin must also match the target hardware if `acmphs_ep_led_update()` is retained.

### Comparator configuration

The SBB uses the generated `g_comparator_ctrl` and `g_comparator_cfg` objects. The target project's generated FSP configuration must therefore describe the comparator instance, reference/input selection, callback, and associated pin configuration required by that hardware design.

### DAC configuration

The FPB-RA8E1 demonstration relies on DAC0. If the target design uses a different analog source, the DAC-dependent demonstration code and configuration must be adapted accordingly. If DAC0 remains the source, the target project must supply the appropriate generated DAC control/configuration objects expected by the abstraction macros.

### Voltage and threshold assumptions

Do not carry the FPB-RA8E1 values `V_REF = 3.3f`, the 1.18 V comparator reference, or the demo's excluded `1400..1600` region into another hardware design without validating them against that board and its FSP/peripheral configuration.

### Timing

The application does not hard-code a comparator stabilization delay. It calls `InfoGet`, retains the driver's `min_stabilization_wait_us`, and uses that returned value before reading comparator status. Preserve that relationship if the comparison sequence is reused.

### Memory and task priorities

The supplied files do **not** define memory sizing requirements or RTOS task-priority requirements for this SBB. The reference application shown here is a continuous foreground loop. Any memory or task-priority changes required by a larger application must be determined in that application's integration context.

### Console/middleware setup

The demo calls `uart_console_init()`, uses `printf()`/`fgets()` for interaction, and calls `uart_console_flush()` during error handling. If the reusable comparator code is integrated without the demo application, this console behavior is not inherently required by the SBB core.

## Dependencies

From the supplied source, the SBB/reference application depends on the following interfaces and resources:

### Renesas FSP / BSP

- ACMPHS HAL driver, selected through the source abstraction as `r_acmphs` or `r_acmphs_b` according to BSP feature macros.
- DAC HAL driver, selected as `r_dac` or `r_dac_b` according to BSP feature macros.
- IOPORT through `R_IOPORT_PinWrite()` for the on-board LED.
- BSP software delay through `R_BSP_SoftwareDelay()`.
- FSP version API through `R_FSP_VersionGet()` in the reference application.
- Generated HAL declarations made available through `hal_data.h`.

### Application/console

- `console/uart_console.h`
- C standard I/O/string/conversion functions used by the demo (`printf`, `snprintf`, `fgets`, `atoi`, `memset`, `strcspn`).

### Hardware resources on FPB-RA8E1

- ACMPHS channel 0
- DAC0
- AN000 / P004 comparator input
- DA0 / P014 DAC output
- P408 on-board LED GPIO as represented by `BSP_IO_PORT_04_PIN_08`
- One jumper wire between the DAC output and comparator input
- J-Link On-Board serial connection for the demo console

## Serial terminal settings

The original example documentation specifies these terminal settings:

- Speed: **115200 baud**
- Data: **8 bit**
- Parity: **none**
- Stop bits: **1 bit**
- Flow control: **none**
- Port: the port provided by J-Link On-Board

For Tera Term, the original instructions also specify:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## Demonstrated behavior and performance

### Demonstrated behavior

The supplied reference demonstrates that:

- DAC0 can be initialized and updated through the SBB wrapper.
- The configured ACMPHS instance can be opened, queried for stabilization information, and enabled.
- The DAC-generated analog signal can be compared against the configured 1.18 V internal reference.
- The comparator HIGH/LOW result can be obtained and displayed.
- The FPB-RA8E1 on-board LED can reflect the comparator state.
- Comparator callbacks can trigger a later status refresh through the SBB's internal flag.

### Performance data

The supplied documentation and source do **not** provide measured latency, interrupt response time, CPU load, memory footprint, accuracy, throughput, or other benchmark results. The only explicit timing behavior is that the application waits for the minimum stabilization interval returned by the ACMPHS driver before refreshing comparator status.

Consequently, this example should be treated as a functional reference, not as a performance benchmark.

## File ownership and modification guidance

The supplied files naturally separate into reusable SBB code and demo/reference application code:

| File | Role | Modification guidance |
|---|---|---|
| `acmphs_ep.c` | Reusable SBB implementation | Modify only when intentionally changing or extending the reusable ACMPHS/DAC/LED abstraction. |
| `acmphs_ep.h` | Reusable SBB public interface and compile-time mappings | Modify when the reusable interface or board/peripheral mapping needs to change. Keep application-only behavior out of this interface. |
| `acmphs_example_app.c` | Demo/reference application | Safe place to customize the demonstration workflow, console interaction, prompts, input policy, and application-specific sequencing. |
| `acmphs_example_app.h` | Demo/reference application interface and banner text | Demo-oriented. Customize when changing the reference application's entry point, displayed information, or example metadata. |
| `readme.txt` | Original example documentation | Source documentation used to derive this SBB guide. It is not executable code. |

The supplied files reference FSP-generated objects from `hal_data.h`, but the provided material does not enumerate the generated configuration files themselves. Files generated by FSP configuration tools should be treated separately from the hand-maintained SBB and demo files; do not classify a particular unprovided file as generated unless that is established by the project that contains it.

## Adopting a newer SBB release after making local changes

If `acmphs_ep.c` or `acmphs_ep.h` are modified locally, those changes become a local fork of the reusable SBB implementation. A later SBB update may modify the same functions, macros, or interfaces, so the local changes may need to be reconciled during an upgrade.

To reduce upgrade friction:

- Prefer keeping product-specific behavior in the application/integration layer rather than changing the SBB core when the core API already supports the required behavior.
- Keep board/application policy such as prompts, input ranges used only by a demo, and presentation logic out of reusable code where possible.
- Record intentional changes made to `acmphs_ep.c` and `acmphs_ep.h` so they can be reviewed against a newer SBB version.
- When adopting a newer release, compare the new SBB core against the locally modified version and reapply only the customizations that are still required.
- Revalidate comparator configuration, DAC behavior, physical signal routing, timing, callback operation, and LED behavior after integration.

The supplied files do not define an automated migration or compatibility mechanism, so source-level compatibility with a future SBB release should not be assumed.

## Quick-start reading order

For a customer evaluating this SBB, the recommended code-reading order is:

1. `acmphs_ep.h` to understand the public SBB functions, constants, and FSP mappings.
2. `acmphs_ep.c` to see how the SBB wraps DAC0, ACMPHS, callback/status handling, LED output, and cleanup.
3. `acmphs_example_app.c` to see one complete integration sequence on FPB-RA8E1.
4. `acmphs_example_app.h` to understand the demo entry point and displayed example information.

This ordering keeps the reusable SBB interface separate from the reference application's console-driven demonstration behavior.
