# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

M5Dial-EspHome is an ESPHome firmware that turns an M5Stack Dial (ESP32-S3, 240x240 circular display, rotary encoder) into a WLED controller integrated with Home Assistant. All logic lives in a single monolithic file `m5dial.yaml` (~1300 lines of YAML + embedded C++ lambdas). There are no custom ESPHome components.

## Build Commands

```bash
# Install ESPHome
pip3 install esphome

# Create secrets from template (required before first build)
cp secrets.yaml.example secrets.yaml

# Compile firmware
esphome compile m5dial.yaml

# Upload via USB (first time) or WiFi (subsequent)
esphome upload m5dial.yaml

# Monitor serial/WiFi logs
esphome logs m5dial.yaml

# Validate configuration without building
esphome config m5dial.yaml
```

There are no tests or linting tools configured for this project.

## Architecture

### Single-File Design

Everything is in `m5dial.yaml`: hardware config, state management (~39 global variables), input handling, display rendering, and Home Assistant service calls. The display UI is rendered entirely within a single `display:` lambda using ESPHome's drawing primitives. An inline C++ lambda (`hsv_to_rgb`) handles HSV-to-RGB conversion within the display lambda.

### Menu State Machine

The UI is driven by a `current_menu` global (int 0-5, 8):

| ID | Menu | Key Input |
|----|------|-----------|
| 0 | Main menu (circular layout, 6 items at 95px radius) | Rotary selects option, wraps around |
| 1 | Device selection (contour-following list) | Rotary scrolls, button toggles checkboxes, Done confirms |
| 2 | Color wheel (HSV picker, 60 segments) | Rotary changes hue (10° per step) |
| 3 | Effect browser | Rotary scrolls effects |
| 4 | Brightness control (arc slider) | Rotary adjusts value |
| 5 | Group selection (contour-following list) | Rotary scrolls, button selects (radio), Done confirms |
| 8 | Save group prompt | 2 options: Use Once / Save as Group |

Main menu items (indices 0-5): Devices, Power, Color, Effect, Brightness, Groups.

### Input Handling

- **Short click** (50ms-1s): Menu-specific action (navigate, toggle, apply)
- **Long press** (1-5s): Return to main menu from any submenu (escape hatch)
- **Button handler ordering**: Service calls execute BEFORE menu navigation to prevent race conditions where UI updates before the command is sent.

### Device Management

Devices sync from Home Assistant via a template sensor (`sensor.wled_device_list`) that provides a pipe-delimited device string parsed into parallel vectors (`device_names`, `device_entity_ids`). A hard limit of 20 devices is enforced. Effects are stored per-device in `effect_lists[20]` arrays, though currently all devices share the same effect list.

### Control Modes

- **Single device** (`control_mode == 0`): Commands target one `entity_id`
- **Group** (`control_mode == 1`): Commands target comma-separated `entity_ids`

Groups 0 ("All Devices") and 1 ("Custom 1") are built-in; up to 10 total groups supported. Groups are not persisted across reboots (only `selected_group_index` and `control_mode` are restored).

### Group Selection Behavior

The group selection screen (menu 5) uses the same visual design as device selection (menu 1):
- Contour-following list layout curving along the circular display edge
- Radio-button checkboxes (single-select, only one group checked at a time)
- "Done" button at the end of the list to confirm selection
- `pending_group_selection` global tracks the tentative choice before Done is pressed
- `control_mode` is only changed to 1 if the user selects a different group than the current one

### Boot Validation

On boot, restored globals are validated to protect against flash corruption:
- `selected_group_index` clamped to valid range [0, num_groups)
- `control_mode` clamped to 0 or 1
- `num_presets` clamped to [0, 10]
- Preset names regenerated (preset_names cannot be restored from flash)
- Group names and group_devices re-initialized (cannot be restored)

### Critical Implementation Details

- **Display color inversion**: Hardware has `invert_colors: true`. Use standard RGB values in code. The HSV color wheel uses a +180 degree hue offset (`hue = ((i * 360/60) + 180) % 360`) to compensate; the center preview circle does NOT apply this offset.
- **9 o'clock positioning**: The selected item in both the main menu circle and list views is always rendered at the 270 degree position (left-middle of screen). Angle formula: `270 + ((item_index - menu_index) * angle_per_item)` where `angle_per_item = 360 / num_items`.
- **Contour-following lists**: List items on device and group screens follow the circular display edge. X position is calculated as `center_x - sqrt(radius^2 - (y - center_y)^2)` with a 12px edge offset and a sqrt guard against negative arguments.
- **Touch is disabled**: The FT6336U touch controller causes I2C NACK errors; touch handling code exists but is inactive.
- **HSV-to-RGB**: Defined as an inline C++ lambda at the top of the display lambda. ESPHome scripts cannot use reference out-parameters, so this must stay inline.

### Hardware Pinout

Display uses SPI (CS=7, DC=4, RST=8, CLK=6, MOSI=5), backlight on GPIO 9 (PWM). Rotary encoder on GPIO 40/41 with button on GPIO 42 (inverted, pullup). Touch controller on I2C (SDA=11, SCL=12, addr 0x38).

### Fonts

Two local Roboto TTF files in `fonts/` (Regular 14pt, Bold 18pt/24pt). Material Design Icons loaded remotely from GitHub as `font_icons` (28pt).

### Home Assistant Services Used

- `light.toggle` -- power toggle
- `light.turn_on` with `hs_color` -- color selection (HSV format `[hue, 100]`)
- `light.turn_on` with `effect` -- effect activation
- `light.turn_on` with `brightness` -- brightness (0-255)

### Known Limitations

- Group mode HA service calls use comma-separated entity_ids; this may not work reliably via the ESPHome native API and may need refactoring to individual calls per device.
- "Use Once" on the save group prompt silently overwrites the "Custom 1" group.
- Groups created at runtime are lost on reboot.
- No way to delete groups once created.
- No feedback when the group limit (10) is reached on "Save as Group".
