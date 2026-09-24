+++
title = "Rust keyboard firmware"
description = "Fixing KVM Freezes on a Split Keyboard"
date = "2026-09-24"
tags = ["Rust", "embassy", "rmk", "rust mechanical keyboard", "qmk", "quantum mechanical keyboard"]

[extra]
image = "/images/posts/qmk_rmk.jpg"
+++

{{ img(path="images/posts/qmk_rmk.jpg", alt="qmk to rmk and embassy") }}

> Note on AI usage: The concepts, experiences, and technical analogies in this post are entirely my own.
> I utilized an AI collaborator to help structure the outline and clean up the prose.

If you use a split mechanical keyboard alongside a hardware KVM switch to jump between workstation machines, you might have run into a frustrating edge case: switching devices abruptly cuts the USB connection, causing custom firmware to freeze or lock up its USB state machine.

Having to physically yank the USB-C cable or perform a hard reset every time you switch PCs completely ruins the seamless workflow a KVM is meant to provide. Here is how diagnosing a USB enumeration issue on a Dilemma Max keyboard led to porting the board’s firmware from QMK to RMK (Rust Mechanical Keyboard framework) powered by Embassy.

## The Root Cause: Sudden USB Disconnects & Microcontroller Loops

When a KVM switches host ports, it doesn't always signal an orderly USB suspend or graceful teardown. Instead, the USB data lines drop abruptly or get re-routed while VBUS power behavior can vary depending on the switch.

Under QMK (which relies on C-based USB stack implementations like LUFA or ChibiOS), a sudden drop in USB state without a clean bus reset can cause the host-side communication driver or split inter-half communication tasks to lock up in a blocking loop or crash the endpoint handler.

Rather than chasing edge-case bug fixes through legacy C macro abstractions, this was a perfect excuse to explore RMK, a modern keyboard framework written in Rust built on top of the Embassy async embedded ecosystem.

## Non-Blocking USB Enumeration with Embassy

The key advantage of RMK lies in its underlying runtime: Embassy. Traditional C firmware often relies on blocking loops or interrupts that can stall if an expected USB handshaking signal never arrives.

In Embassy:

1. **Async USB Stack**: The embassy-usb driver operates asynchronously. When the KVM toggles and severs the USB bus mid-packet, the USB task yields execution rather than spinning in a hard wait lock.

1. **Graceful Re-enumeration**: When the KVM connects to the destination PC, the USB state machine observes the VBUS state change and initiates a clean enumeration sequence automatically.

1. **Isolated Tasks**: Key matrix scanning and split UART communication run as independent async tasks, keeping the local peripheral state responsive regardless of host USB connectivity.

## Decoding QMK Sources for Hardware Pinout Mapping

Porting a custom split keyboard to a new firmware framework can seem daunting, but if you already have existing QMK source files, you hold the entire hardware blueprint. By referencing the keyboard's QMK config files (keyboard.json, config.h, and halconf.h), extracting the physical matrix mapping and microcontroller pin assignments is straightforward

```
QMK config.h / keyboard.json          RMK TOML Configuration
┌─────────────────────────┐           ┌─────────────────────────┐
│ "matrix_pins": {        │           │ [matrix]                │
│   "cols": ["D0", "D1"], │  ───────► │ input_pins = ["PD0"]    │
│   "rows": ["B0", "B1"]  │           │ output_pins = ["PB0"]   │
│ }                       │           └─────────────────────────┘
└─────────────────────────└
```

Because QMK defines row/column pins, split communication protocols (such as UART/serial), and matrix diode directions in plain C headers or JSON schemas, mapping these directly into RMK's configuration format took minutes.

## Declarative Firmware with keyboard.toml

RMK uses a structured TOML format to declare the board metadata, GPIO pin matrix, keymap layers, and split transport options without writing boilerplate firmware logic. Here is an actual representative configuration for a split keyboard setup in RMK

**keyboard.toml**

```toml

[keyboard]
name = "Dilemma_3X6_Clone"
vendor_id = 0xAFC5
product_id = 0xBFC5
manufacturer = "lucky_studio"
chip = "rp2040"

[split]
connection = "serial"

[split.central]
rows = 4
cols = 6
row_offset = 0
col_offset = 0
serial = [
    { instance = "PIO0", tx_pin = "PIN_0", rx_pin = "PIN_1" }
]

[split.central.matrix]
row_pins = ["PIN_6", "PIN_12", "PIN_18", "PIN_17"]
col_pins = ["PIN_10", "PIN_8", "PIN_7", "PIN_5", "PIN_13", "PIN_9"]
row2col = true

[[split.peripheral]]
rows = 4
cols = 6
row_offset = 0
col_offset = 6
serial = [
    { instance = "PIO0", tx_pin = "PIN_0", rx_pin = "PIN_1" }
]

[split.peripheral.matrix]
row_pins = ["PIN_12", "PIN_13", "PIN_17", "PIN_18"]
col_pins = ["PIN_10", "PIN_9", "PIN_8", "PIN_7", "PIN_6", "PIN_5"]
row2col = true

[[input_device.encoder]]
pin_a = "PIN_14"
pin_b = "PIN_16"
resolution = 4

[storage]
enabled = true

[layout]
rows = 4
cols = 12
map = """
    (0,0) (0,1) (0,2) (0,3) (0,4) (0,5) (0,6) (0,7) (0,8) (0,9) (0,10) (0,11)
    (1,0) (1,1) (1,2) (1,3) (1,4) (1,5) (1,6) (1,7) (1,8) (1,9) (1,10) (1,11)
    (2,0) (2,1) (2,2) (2,3) (2,4) (2,5) (2,6) (2,7) (2,8) (2,9) (2,10) (2,11)
    (3,0) (3,1) (3,2) (3,3) (3,4) (3,5) (3,6) (3,7) (3,8) (3,9) (3,10) (3,11)
"""

[keymap]
[[keymap.layer]]
keys = """
    Q       W       E       R       T       Y       U       I       O       P       No      No
    A       S       D       F       G       H       J       K       L       SemiColon  No      No
    Z       X       C       V       B       N       M       Comma   Dot     Slash   No      No
    LCtrl   LShift  Space   Tab     Enter   BackSpace  LAlt    LGUI    No      No      No      No
"""

[[keymap.layer]]
keys = """
    1       2       3       4       5       6       7       8       9       0       No      No
    No      Right   Up      Down    Left    VolU    No      No      No      No      No      No
    No      No      No      No      No      VolD    No      No      No      No      No      No
    LAlt    TRNS    No     No      No      No      No      No      No      No      No      No
"""

```

## Outgrowing TOML and Moving to Pure Rust & Embassy

My keyboard (the Dilemma Max) includes WS2812B RGB underglow LEDs. Because RMK's TOML configuration schema did not yet support custom underglow peripherals out of the box, driving the LEDs meant stepping past declarative config files and writing custom firmware in pure Rust.

```Rust

// Spawning a background task in Embassy for manual WS2812B LED control
# [embassy_executor::task]
async fn run_underglow(mut ws2812: Ws2812Spi<...>) {
  loop {
    // Drive RGB state independently without blocking matrix scanning
    ws2812.write(render_lighting_frame()).await.ok();
    Timer::after_millis(30).await;
  }
}
```

Moving to a full Rust codebase allowed initializing hardware peripherals directly using Embassy drivers:

1. **Direct Peripheral Ownership**: Using Embassy’s HAL for the microcontroller, SPI or bit-banged PWM peripherals were initialized manually to handle the strict timing protocols required by WS2812B LEDs.

1. **Concurrent Async Tasks**: By spawning an independent background task via Embassy's #[embassy_executor::task], the underglow lighting animations run concurrently without blocking key scanning or USB polling tasks.

1. **Deep Customization**: Bypassing high-level abstractions provided direct control over energy efficiency, custom layer status indicators, and custom hardware state handling.

## Results & Takeaways

1. **Zero Cable Yanks**: KVM switching between workstations is seamless—the keyboard re-enumerates within milliseconds of target selection.

1. **Maintainable Configuration**: Moving from C macros and header definitions to Rust's type-checked abstractions and TOML configs makes future keymap and hardware tweaks simple and safe.Modern Embedded Stack: Leveraging async Rust in embedded contexts demonstrates how modern language tools eliminate entire classes of concurrency and state-machine deadlock bugs common in C.

1) **Practical Embedded Rust**: isn't just about memory safety. It offers structured concurrency and async paradigms that simplify complex state management across USB, split-serial communication, and hardware peripherals.
