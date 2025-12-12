<h1>
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/open-control/.github/main/logo-white.svg">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/open-control/.github/main/logo.svg">
  <img alt="" src="https://raw.githubusercontent.com/open-control/.github/main/logo.svg" width="36" valign="middle">
</picture>
&nbsp;Open Control
</h1>

**A modular C++17 framework for building hardware controllers.**

> **Alpha** - API subject to change. Not recommended for production.

---

## Overview

Open Control is a **platform-agnostic** framework for building any type of controller. The core compiles and tests natively on desktop (74+ unit tests), while HAL layers provide hardware drivers.

**Platforms:**
- **Teensy 4.x** - Supported and tested
- **Daisy Seed** - Planned
- **Desktop** - Native builds for development/testing

Lightweight with minimal dynamic allocation. Designed for **PlatformIO**. Open to contributions.

---

## Quick Start

```bash
git clone https://github.com/open-control/example-teensy41-minimal.git
cd example-teensy41-minimal
pio run -e release -t upload
```

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                         Your Application                           │
│   IContext lifecycle · Fluent input bindings · Scoped cleanup      │
├────────────────────────────────────────────────────────────────────┤
│                   Framework (platform-agnostic)                    │
│                                                                    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────────┐ │
│  │ InputBinding│ │ContextMgr  │ │  EventBus   │ │  Signal<T>   │ │
│  │ gestures    │ │ lifecycle   │ │  pub/sub    │ │  reactive    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └──────────────┘ │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────────┐ │
│  │ Result<T>   │ │ Settings<T> │ │EncoderLogic │ │   APIs       │ │
│  │ error types │ │ persistence │ │ 3 modes     │ │ Button/Enc/  │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ │ MIDI         │ │
│                                                   └──────────────┘ │
│                   Compiles native · 74+ unit tests                 │
├────────────────────────────────────────────────────────────────────┤
│                         HAL Interface                              │
│   IMidi · IEncoder · IButton · IDisplay · IStorage · IMultiplexer  │
├────────────────────────────────────────────────────────────────────┤
│                      HAL Implementations                           │
│   ┌──────────────────────┐     ┌──────────────────────┐           │
│   │      hal-teensy      │     │  hal-daisy (planned) │           │
│   │  USB MIDI · Encoders │     │                      │           │
│   │  Buttons · ILI9341   │     │                      │           │
│   │  EEPROM · LittleFS   │     │                      │           │
│   │  Multiplexers        │     │                      │           │
│   └──────────────────────┘     └──────────────────────┘           │
└────────────────────────────────────────────────────────────────────┘
```

---

## Core Features

### Fluent Input Binding

```cpp
// Button gestures
onButton(1).press().then([this]() { midi().sendCC(0, 20, 127); });
onButton(1).release().then([this]() { midi().sendCC(0, 20, 0); });
onButton(1).longPress(500).then([]() { /* alt action */ });
onButton(1).doubleTap(300).then([]() { /* reset */ });

// Button combos
onCombo(BTN_1, BTN_2).then([]() { /* both pressed */ });

// Encoder modes
encoder(1).setMode(EncoderMode::NORMALIZED);  // bounded angle → 0.0-1.0
encoder(1).setMode(EncoderMode::RAW);         // unbounded raw ticks
encoder(1).setMode(EncoderMode::RELATIVE);    // ±delta per detent

onEncoder(1).turn().then([this](float v) {
    midi().sendCC(0, 16, static_cast<uint8_t>(v * 127));
});
```

### Reactive State (Signal)

```cpp
Signal<float> volume{0.5f};
volume.subscribe([](float v) { updateUI(v); });
volume.set(0.8f);  // triggers callback

SignalString<32> paramName;  // zero-allocation, fixed buffer
paramName.subscribe([](const char* s) { label.setText(s); });
```

### Persistent Settings

```cpp
struct MySettings {
    uint8_t midiChannel = 0;
    float gain = 1.0f;
};

Settings<MySettings> settings(storageBackend);
settings.load();  // CRC32 validated, auto-defaults on corruption
settings.data().gain = 0.8f;
settings.save();  // only writes if dirty
```

### Context System

```cpp
class MyContext : public oc::context::IContext {
public:
    static constexpr Requirements REQUIRES{
        .button = true, .encoder = true, .midi = true
    };

    bool initialize() override {
        // Bindings auto-cleanup on context switch
        onEncoder(1).turn().then([this](float v) { ... });
        return true;
    }

    void update() override { }
    void cleanup() override { }
};

// Deferred switching (safe from within handlers)
switchTo<ContextID::SETTINGS>();
```

### Scoped Bindings

```cpp
// Bindings tied to UI visibility
auto scope = ui::scope(myDialog);
onButton(1, scope).press().then([]() { /* only when dialog visible */ });
// Auto-cleanup when dialog hidden
```

---

## Teensy 4.1 Examples

| # | Example | Concepts |
|---|---------|----------|
| 01 | [example-teensy41-01-midi-output](https://github.com/open-control/example-teensy41-01-midi-output) | Raw UsbMidi driver |
| 02 | [example-teensy41-02-encoders](https://github.com/open-control/example-teensy41-02-encoders) | AppBuilder, IContext, encoder bindings |
| 03 | [example-teensy41-03-buttons](https://github.com/open-control/example-teensy41-03-buttons) | press, release, longPress, doubleTap |
| — | [example-teensy41-minimal](https://github.com/open-control/example-teensy41-minimal) | Full controller (encoders + buttons) |
| — | [example-teensy41-lvgl](https://github.com/open-control/example-teensy41-lvgl) | ILI9341 + LVGL + DMA |

---

## Repositories

### Core

| Repository | Description |
|------------|-------------|
| [framework](https://github.com/open-control/framework) | Platform-agnostic core (contexts, bindings, events, signals, settings) |
| [hal-common](https://github.com/open-control/hal-common) | HAL interfaces and type definitions |
| [hal-teensy](https://github.com/open-control/hal-teensy) | Teensy 4.x: USB MIDI, encoders, buttons, ILI9341, EEPROM, LittleFS, multiplexers |

### UI

| Repository | Description |
|------------|-------------|
| [ui-lvgl](https://github.com/open-control/ui-lvgl) | LVGL 9.x bridge (DMA display, scope system) |
| [ui-lvgl-components](https://github.com/open-control/ui-lvgl-components) | Widgets (Knob, Button, Label, Enum, StateIndicator) + SDL desktop viewer |
| [ui-lvgl-cli-tools](https://github.com/open-control/ui-lvgl-cli-tools) | TTF/IMG → C array converters |

### Tools

| Repository | Description |
|------------|-------------|
| [cli-tools](https://github.com/open-control/cli-tools) | `oc-build` `oc-monitor` `oc-upload` `oc-clean` |
| [protocol-codegen](https://github.com/open-control/protocol-codegen) | Schema-driven protocol generator (MIDI SysEx → C++ / Java) |

---

## Hardware Support

| Feature | hal-teensy |
|---------|------------|
| **MIDI** | USB MIDI with active note tracking |
| **Encoders** | Quadrature (ISR-driven), 3 modes |
| **Buttons** | Debounced, direct GPIO or multiplexed |
| **Multiplexers** | CD74HC4067 (16ch), 4051 (8ch), 4052 (4ch) |
| **Display** | ILI9341 320x240, DMA, differential buffering |
| **Storage** | EEPROM (4KB) or LittleFS (flash) |

---

## UI Components

**Widgets:** KnobWidget, ButtonWidget, Label (auto-scroll), EnumWidget, StateIndicator

**Components:** ParameterKnob, ParameterEnum, ParameterSwitch

**Theme:** 8 macro colors + UI palette + layout constants

**Desktop development:** SDL viewer (no hardware needed)
```bash
cd ui-lvgl-components && ./build_demo.sh && ./run_demo.sh
```

---

## Protocol Code Generation

[protocol-codegen](https://github.com/open-control/protocol-codegen) generates strongly-typed protocol code from simple schema definitions:

- Define composable data structures and messages once
- Generate synchronized C++ and Java code
- Currently supports **MIDI SysEx** (OSC planned)
- Goal: focus on your data, not protocol implementation

---

## Configuration

```ini
# platformio.ini
build_flags =
    -D OC_LOG    ; colored timestamps via Serial (remove for production)
```

Examples use `// ADAPT:` comments for hardware-specific pin values.

---

## Development

- **Tested:** Windows + Linux (native builds)
- **Tests:** 74+ unit tests (Unity framework)
- **CI:** Native platform builds via PlatformIO

---

## License

Apache 2.0

---

<sub>Built for music by [petitechose.audio](https://petitechose.audio)</sub>
