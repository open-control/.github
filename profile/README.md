# Open Control

**A modular C++ framework for building MIDI controllers and embedded audio interfaces.**

Open Control provides a clean, type-safe architecture for Teensy-based hardware projects with:

- **Fluent Binding API** — Declarative input handling with `onButton().press().then(...)`
- **Context System** — Application modes with lifecycle management and hot-switching
- **Hardware Abstraction** — Write once, run on Teensy 4.x (more platforms coming)
- **Optional LVGL Integration** — DMA-accelerated displays with flicker-free rendering

## Repositories

| Repository | Description |
|------------|-------------|
| [framework](https://github.com/open-control/framework) | Core framework: contexts, bindings, events, APIs |
| [hal-common](https://github.com/open-control/hal-common) | Platform-agnostic hardware definitions |
| [hal-teensy](https://github.com/open-control/hal-teensy) | Teensy 4.x drivers (encoders, buttons, MIDI, display) |
| [ui-lvgl](https://github.com/open-control/ui-lvgl) | LVGL 9.x integration layer |

## Examples

| Example | Description |
|---------|-------------|
| [example-teensy41-minimal](https://github.com/open-control/example-teensy41-minimal) | 4 encoders + 2 buttons → MIDI CC (no display) |
| [example-teensy41-lvgl](https://github.com/open-control/example-teensy41-lvgl) | Full demo with ILI9341 display and LVGL UI |

## Quick Example

```cpp
class MyContext : public oc::context::IContext {
public:
    static constexpr oc::context::Requirements REQUIRES{
        .button = true, .encoder = true, .midi = true
    };

    bool initialize() override {
        // Encoder → MIDI CC
        onEncoder(ENC_1).turn().then([this](float value) {
            midi().sendCC(0, 16, static_cast<uint8_t>(value * 127));
        });

        // Button press/release
        onButton(BTN_1).press().then([this]() {
            midi().sendCC(0, 20, 127);
        });
        onButton(BTN_1).release().then([this]() {
            midi().sendCC(0, 20, 0);
        });

        return true;
    }
};
```

## Getting Started

```bash
# Clone an example
git clone https://github.com/open-control/example-teensy41-minimal.git
cd example-teensy41-minimal

# Build and upload with PlatformIO
pio run -e release -t upload
```

## License

All repositories are licensed under **Apache 2.0**.

---

*Built with music in mind by [petitechose.audio](https://petitechose.audio)*
