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

Open Control is a **platform-agnostic** framework for building any type of controller. The core compiles and tests natively on desktop (200+ unit tests), while HAL layers provide hardware drivers.

**Platforms:**
- **Teensy 4.x** - Supported and tested
- **Daisy Seed** - Planned
- **ESP32** - Planned
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
┌──────────────────────────────────────────────────────────────────┐
│                        Your Application                          │
│  IContext lifecycle · Fluent input bindings · Scoped cleanup     │
├──────────────────────────────────────────────────────────────────┤
│                  Framework (platform-agnostic)                   │
│                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────┐ ┌──────────────┐  │
│  │InputBinding │ │ ContextMgr  │ │ EventBus  │ │  Signal<T>   │  │
│  │  gestures   │ │  lifecycle  │ │  pub/sub  │ │   reactive   │  │
│  └─────────────┘ └─────────────┘ └───────────┘ └──────────────┘  │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────┐ ┌──────────────┐  │
│  │  Result<T>  │ │ Settings<T> │ │CobsCodec  │ │SignalWatcher │  │
│  │ error types │ │ persistence │ │  framing  │ │  coalesce    │  │
│  └─────────────┘ └─────────────┘ └───────────┘ └──────────────┘  │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────┐ ┌──────────────┐  │
│  │   Encoder   │ │  HAL APIs   │ │  Logging  │ │SignalVector  │  │
│  │   3 modes   │ │ Btn/Enc/Ser │ │  colored  │ │ fixed-cap    │  │
│  └─────────────┘ └─────────────┘ └───────────┘ └──────────────┘  │
│                                                                  │
│                 Compiles native · 200+ unit tests                │
├──────────────────────────────────────────────────────────────────┤
│                        HAL Interface                             │
│  IMidi · IEncoder · IButton · IDisplay · IStorage · ISerial      │
├──────────────────────────────────────────────────────────────────┤
│                     HAL Implementations                          │
│  ┌────────────────────────┐   ┌────────────────────────┐         │
│  │      hal-teensy        │   │   hal-daisy (planned)  │         │
│  │  USB MIDI · USB Serial │   │                        │         │
│  │  Encoders · Buttons    │   │                        │         │
│  │  ILI9341 DMA · Mux     │   │                        │         │
│  │  EEPROM · LittleFS     │   │                        │         │
│  └────────────────────────┘   └────────────────────────┘         │
└──────────────────────────────────────────────────────────────────┘
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
onButton(BTN_A).combo(BTN_B).then([]() { /* both pressed */ });

// Latch (toggle) behavior
onButton(1).press().latch().then([]() { /* toggle on/off */ });

// Conditional bindings
onEncoder(1).turn()
    .when(button(BTN_SHIFT).pressed())
    .then([](float v) { /* fine adjust */ });

// Encoder modes with quantization
encoder(1).setMode(EncoderMode::NORMALIZED);
encoder(1).setBounds(0.0f, 127.0f);
encoder(1).setDiscreteSteps(128);  // quantize to steps
```

### Reactive State (Signal Family)

```cpp
// Basic signal
Signal<float> volume{0.5f};
volume.subscribe([](float v) { updateUI(v); });
volume.set(0.8f);  // triggers callback

// Fixed-capacity string (zero heap allocation)
SignalString<32> paramName;
paramName.subscribe([](const char* s) { label.setText(s); });

// Fixed-capacity vector
SignalVector<std::string, 32> deviceNames;
deviceNames.subscribe([]() { rebuildList(); });
deviceNames.set(names.data(), names.size());

// Coalesce multiple signals into one callback
SignalWatcher watcher;
watcher.watchAll(
    [this]() { updateUI(); },
    state.name, state.type, state.enabled
);
```

### Multi-Protocol Communication

```cpp
// MIDI (native USB)
midi().sendNoteOn(channel, note, velocity);
midi().sendCC(channel, cc, value);

// Serial8 (via bridge for high-bandwidth)
serial().send(buffer, length);  // COBS-encoded frames
serial().setOnReceive([](const uint8_t* data, size_t len) {
    processFrame(data, len);
});
```

### Persistent Settings

```cpp
struct MySettings {
    uint8_t midiChannel = 0;
    float gain = 1.0f;
};

Settings<MySettings> settings(storageBackend, 0x0000, /*version=*/1);
settings.load();  // CRC32 validated, auto-defaults on corruption
settings.modify([](auto& s) { s.gain = 0.8f; });  // dirty tracking
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
app.contexts().switchTo(ContextID::SETTINGS);
```

### Scoped Bindings

```cpp
// Bindings grouped for bulk cleanup
constexpr ScopeID MENU_SCOPE = 1;
onButton(BTN_1).press().scope(MENU_SCOPE).then([]{ /* ... */ });
onButton(BTN_2).press().scope(MENU_SCOPE).then([]{ /* ... */ });
buttons().clearScope(MENU_SCOPE);  // clear all at once
```

---

## High-Bandwidth Communication

### The Bridge

[oc-bridge](https://github.com/open-control/bridge) enables high-bandwidth bidirectional communication between hardware and DAW extensions.

| | MIDI SysEx | USB Serial + UDP |
|---|---|---|
| Bandwidth | ~31.25 kbit/s | Full USB speed (480 Mbit/s) |
| Encoding | 7-bit (overhead) | 8-bit native |
| Latency | ~10-50ms | ~2-3ms |

```
┌──────────────┐    USB Serial     ┌────────────┐      UDP       ┌─────────────┐
│   Teensy     │◄─────────────────►│  oc-bridge │◄──────────────►│   Bitwig    │
│  Controller  │   COBS framing    │    (TUI)   │   :9000        │  Extension  │
└──────────────┘                   └────────────┘                └─────────────┘
```

**Bridge Features:**
- Terminal UI with real-time stats
- Auto-detect Teensy port
- Log filtering (Proto/Debug/All) and export
- Windows service mode
- TOML configuration

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
| [framework](https://github.com/open-control/framework) | Platform-agnostic core (contexts, bindings, events, signals, settings, COBS codec) |
| [hal-common](https://github.com/open-control/hal-common) | HAL interfaces and type definitions |
| [hal-teensy](https://github.com/open-control/hal-teensy) | Teensy 4.x: USB MIDI, USB Serial, encoders, buttons, ILI9341, storage, multiplexers |

### Communication

| Repository | Description |
|------------|-------------|
| [bridge](https://github.com/open-control/bridge) | Rust Serial-to-UDP bridge with TUI, auto-detect, Windows service |
| [protocol-codegen](https://github.com/open-control/protocol-codegen) | Python code generator: Serial8/SysEx → C++ / Java |

### UI

| Repository | Description |
|------------|-------------|
| [ui-lvgl](https://github.com/open-control/ui-lvgl) | LVGL 9.x integration (Bridge, IWidget, IComponent, scope system) |
| [ui-lvgl-components](https://github.com/open-control/ui-lvgl-components) | Widgets: Knob, Button, Label, Enum, ListItem, VirtualList, StateIndicator |
| [ui-lvgl-cli-tools](https://github.com/open-control/ui-lvgl-cli-tools) | `oc-icons` (SVG→LVGL), font/image converters |

### Build Tools

| Repository | Description |
|------------|-------------|
| [cli-tools](https://github.com/open-control/cli-tools) | `oc-build` `oc-upload` `oc-monitor` `oc-clean` with memory bars and dependency graph |

---

## Hardware Support (hal-teensy)

| Feature | Description |
|---------|-------------|
| **MIDI** | USB MIDI with active note tracking |
| **Serial** | USB Serial for Serial8 protocol (via bridge) |
| **Encoders** | Quadrature (ISR-driven), 3 modes, quantization |
| **Buttons** | Debounced, direct GPIO or multiplexed, latch support |
| **Multiplexers** | CD74HC4067 (16ch), 4051 (8ch), 4052 (4ch) |
| **Display** | ILI9341 320x240, DMA, differential buffering |
| **Storage** | EEPROM (4KB) or LittleFS (flash) |

### Simplified Teensy AppBuilder

```cpp
#include <oc/teensy/Teensy.hpp>

app = oc::teensy::AppBuilder()
    .midi()
    .encoders(Config::ENCODERS)
    .buttons(Config::BUTTONS)
    .inputConfig(Config::INPUT);  // no .build() needed
```

---

## UI Components

### Widgets

| Widget | Description |
|--------|-------------|
| **KnobWidget** | Rotary arc with value indicator, auto-flash on change |
| **ButtonWidget** | Toggle button with ON/OFF states |
| **Label** | Text with optional auto-scroll for overflow |
| **EnumWidget** | Dropdown/selector for enum values |
| **ListItemWidget** | Container with indicator line, manual flash |
| **VirtualList** | Virtualized list for large datasets |
| **StateIndicator** | Circular LED-style indicator |

### Theme System

8 macro colors + UI palette + layout constants for consistent styling.

### Desktop Development (SDL)

Develop UI without hardware using the SDL demo with cross-platform Zig build system:

```bash
cd ui-lvgl-components
./build_demo.sh   # auto-downloads Zig, CMake, Ninja (~200MB first run)
./run_demo.sh
```

---

## Protocol Code Generation

[protocol-codegen](https://github.com/open-control/protocol-codegen) generates type-safe protocol code from Python definitions.

### Dual Protocol Support

| Method | Encoding | Use Case | Bandwidth |
|--------|----------|----------|-----------|
| **serial8** | 8-bit binary | USB Serial + Bridge | High |
| **sysex** | 7-bit MIDI | Direct MIDI SysEx | Lower |

### Optimized Types

```python
# Bandwidth-efficient normalized floats
parameter_value = PrimitiveField('value', type_name=Type.NORM8)   # 1 byte, ~0.8% precision
parameter_value = PrimitiveField('value', type_name=Type.NORM16)  # 2 bytes, high precision
```

### Features

- Composable data structures and messages
- Nested composite types and arrays
- Synchronized C++ and Java code generation
- Zero-allocation Java streaming encoder
- Incremental generation (only changed files)
- Auto message ID allocation

---

## CLI Tools

### Build Tools (cli-tools)

```bash
oc-build              # Build with auto-detected env
oc-upload release     # Build and upload
oc-monitor            # Build, upload, open serial monitor
```

**Output features:**
- Spinner animation during build
- Memory usage bars (FLASH, RAM1, RAM2, PSRAM)
- Dependency graph display
- Warning/error summary with file:line

### Asset Tools (ui-lvgl-cli-tools)

```bash
oc-icons              # Build icon font from SVG files
```

**Icon pipeline:** SVG → Inkscape clean → FontForge TTF → LVGL binary fonts

```cpp
// Generated Icon namespace
#include "Icon.hpp"
Icon::set(label, Icon::PLAY, Icon::Size::M);
```

---

## Configuration

```ini
# platformio.ini
build_flags =
    -DOC_LOG                  # colored timestamps via Serial
    -DOC_MAX_BUTTONS=32       # override compile-time limits
    -DOC_MAX_ENCODERS=8
    -DOC_ENABLE_STATS=1       # enable statistics
```

Examples use `// ADAPT:` comments for hardware-specific pin values.

---

## Development

- **Tested:** Windows + Linux (native builds)
- **Tests:** 200+ unit tests (Unity framework)
- **CI:** Native platform builds via PlatformIO
- **Documentation:** [Wiki](https://github.com/open-control/framework/wiki)

---

## License

Apache 2.0

---

<sub>Built for music by [petitechose.audio](https://petitechose.audio)</sub>
