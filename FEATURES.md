# M5Stack Dial WLED Controller - Feature Design Document

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Menu System](#menu-system)
3. [UI Design Patterns](#ui-design-patterns)
4. [Input Handling](#input-handling)
5. [Device Management](#device-management)
6. [Group Management](#group-management)
7. [State Management](#state-management)
8. [Home Assistant Integration](#home-assistant-integration)
9. [Display Rendering](#display-rendering)
10. [Touch System](#touch-system)

---

## Architecture Overview

### Hardware Components
- **Display**: 240x240 GC9A01A (circular IPS LCD)
- **Touch**: FT6336U capacitive touch controller (I2C 0x38)
- **Encoder**: Rotary encoder with integrated button
- **MCU**: ESP32-S3 (dual-core, WiFi)

### Software Stack
- **Framework**: ESPHome (Arduino-based)
- **Display**: ili9xxx platform with inverted colors
- **Integration**: Home Assistant API
- **Update Rate**: 250ms display refresh, 200ms I2C polling (touch disabled by default)

### Core Principles
1. **Black background** - All menus use black (#000000) background
2. **9 o'clock positioning** - Active/selected items always at 270° (left middle)
3. **Rotation, not scrolling** - Menus rotate around center, not scroll up/down
4. **Color inversion** - Display has `invert_colors: true` so color definitions are standard RGB
5. **Menu state preservation** - Returning to main menu preserves last selected option

---

## Menu System

### Menu State Machine
```
Current Menu ID (global: current_menu):
0 = Main Menu
1 = Device Selection
2 = Color Wheel
3 = Effect Selection
4 = Brightness Control
5 = Group Selection
6 = Group Edit
7 = Preset Selection
8 = Save Group Prompt
```

### Menu Navigation Rules

**Main Menu (menu 0)**:
- `menu_index` tracks selected option (0-7)
- Rotary encoder increments/decrements `menu_index`
- Button press OR touch center executes selected action
- When entering submenu, `menu_index` is preserved
- When returning to main menu, `menu_index` is restored to last position

**Navigation Pattern**:
```
Main Menu → [Submenu] → Main Menu (same index)
Exception: Effect/Group Edit menus reset menu_index to 0 for their own use
```

### Menu Options (Main Menu)

| Index | Icon | Label | Color | Action |
|-------|------|-------|-------|--------|
| 0 | \U000F0335 | Devices | Cyan | Enter device selection (menu 1) |
| 1 | \U000F0425 | Power | Red | Execute toggle_power_action script |
| 2 | \U000F0E0C | Color | Magenta | Enter color wheel (menu 2) |
| 3 | \U000F0766 | Effect | Yellow | Enter effect list (menu 3, reset index) |
| 4 | \U000F00DF | Bright | Orange | Enter brightness (menu 4) |
| 5 | \U000F1254 | Groups | Green | Enter group selection (menu 5) |
| 6 | \U000F1C2B | Presets | Blue | Enter preset selection (menu 7) |
| 7 | \U000F0FE2 | Mode | Purple | Toggle control_mode (0↔1) |

---

## UI Design Patterns

### Circular Menu Layout (Main Menu)

**Visual Structure**:
- 8 icons arranged in circle at 95px radius from center (120, 120)
- Selected item ALWAYS at 270° (9 o'clock position)
- Other items distributed evenly around circle
- Selected item: 22px radius circle
- Unselected items: 18px radius circle

**Rotation Calculation**:
```cpp
float selected_angle = 270.0;  // 9 o'clock
float angle_per_item = 360.0 / 8;  // 45° spacing

// For item i when menu_index is selected:
float angle = selected_angle + ((i - menu_index) * angle_per_item);
```

**Center Display**:
- Device name (if single device mode)
- "Group: [name]" (if group mode)
- Selected menu label below

### List Layout (Device/Effect/Group Selection)

**Contour-Following List** (Device Selection):
- Items arranged vertically on LEFT side of screen
- Follow circular screen edge (calculated X position)
- Active item at y=120 (9 o'clock vertical position)
- Items above/below scroll past active position
- Spacing: 35px between items

**X Position Calculation**:
```cpp
int edge_offset = 12;  // pixels from edge
float dy = y - center_y;
float max_radius = 120.0;
float usable_radius = max_radius - edge_offset;
float dx = sqrt(usable_radius * usable_radius - dy * dy);
int x = center_x - dx;  // Left side of circle
```

**Visibility Range**:
- 3 items above active item
- Active item at center (y=120)
- 3 items below active item
- Total: 7 visible items max

### Color Wheel Design

**Structure**:
- Outer ring: 105px radius
- Inner ring: 80px radius
- Center preview: 65px radius filled circle
- 60 color segments around ring

**Color Mapping** (CRITICAL):
- Wheel segments: `hue = ((i * 360 / 60) + 180) % 360` 
- Indicator position: `angle = (color_hue - 90) * π / 180`
- Center preview: `hue = color_hue` (no offset, inverted display handles it)

**Visual Elements**:
- Colored ring with gradient segments
- White indicator dot (8px) at selected hue position
- Center circle filled with selected color
- White border around center (2px)
- Hue value (0-360°) in center
- Color name below hue value
- Text color: white or black depending on hue for readability

**Color Names**:
```cpp
if (hue >= 345 || hue < 15) → "Red"
else if (hue < 45) → "Orange"
else if (hue < 75) → "Yellow"
else if (hue < 150) → "Green"
else if (hue < 210) → "Cyan"
else if (hue < 270) → "Blue"
else if (hue < 330) → "Purple"
else → "Magenta"
```

---

## Input Handling

### Rotary Encoder

**Configuration**:
- Pin A: GPIO40, Pin B: GPIO41
- Resolution: 1 (one detent = one step)
- Debounce: 0.1s (100ms)

**Behavior by Menu**:

| Menu | Clockwise | Counter-Clockwise |
|------|-----------|-------------------|
| 0 (Main) | menu_index++ (max 7) | menu_index-- (min 0) |
| 1 (Device) | device_select_index++ (max num_devices) | device_select_index-- (min 0) |
| 2 (Color) | color_hue += 3 (wrap 360) | color_hue -= 3 (wrap 0) |
| 3 (Effect) | menu_index++ (max effects-1) | menu_index-- (min 0) |
| 4 (Brightness) | brightness_value += 5 (max 255) | brightness_value -= 5 (min 0) |
| 5 (Group) | group_select_index++ (max groups-1) | group_select_index-- (min 0) |
| 7 (Preset) | preset_select_index++ (max presets-1) | preset_select_index-- (min 0) |
| 8 (Save) | menu_index++ (max 1) | menu_index-- (min 0) |

### Button (Dial Press)

**Configuration**:
- Pin: GPIO42 (inverted, pullup)
- Event: `on_click` with min 50ms, max 1000ms
- No debounce filters (handled by on_click timing)

**Execution Order** (CRITICAL):
1. **FIRST**: Execute service calls (color, effect, brightness, power)
2. **THEN**: Change menu state (navigation lambda)

This prevents race conditions where menu changes before service executes.

**Button Behavior by Menu**:

| Menu | Action |
|------|--------|
| 0, index 0 | Enter device selection (menu 1) |
| 0, index 1 | Execute toggle_power_action script |
| 0, index 2 | Enter color wheel (menu 2) |
| 0, index 3 | Enter effects (menu 3), reset menu_index to 0 |
| 0, index 4 | Enter brightness (menu 4) |
| 0, index 5 | Enter groups (menu 5) |
| 0, index 6 | Enter presets (menu 7) |
| 0, index 7 | Toggle control_mode (0↔1) |
| 1 (Device) | If on "Done": process selection, else toggle device |
| 2 (Color) | Apply color, return to menu 0 index 2 |
| 3 (Effect) | Apply effect, return to menu 0 index 3 |
| 4 (Brightness) | Apply brightness, return to menu 0 index 4 |
| 5 (Group) | Select group, return to menu 0 index 5 |
| 7 (Preset) | Apply preset, return to menu 0 index 6 |
| 8 (Save) | Save or use group, return to menu 0 index 0 |

### Touch System (Currently Disabled)

**Touch Controller**: FT6336U at I2C address 0x38

**IMPORTANT**: Touch is currently disabled due to I2C reliability issues. The interval component is commented out in the YAML.

**When Enabled - Touch Behavior**:

Touch coordinates are read via I2C polling (200ms interval):
```cpp
uint8_t data[16];
i2c_bus.read(0x38, data, 16);
int touch_x = ((data[3] & 0x0F) << 8) | data[4];
int touch_y = ((data[5] & 0x0F) << 8) | data[6];
```

**Validation Rules**:
- Only accept coordinates 5-235 (filter edge noise and 0,0 phantoms)
- Ignore repeated coordinates (debouncing)
- Track last processed coordinates to prevent repeats

**Touch Zones**:

*Color Wheel (menu 2)*:
- Radius 85-110: Select color (update color_hue based on angle)
- Radius < 70: Confirm selection (return to main menu)

*Other Menus*:
- y < 80: Scroll up (decrement appropriate index)
- y > 160: Scroll down (increment appropriate index)
- Center (radius < 100): Execute selection (same as button press)

**Menu-Specific Touch Scrolling**:
- Menu 1: Modify `device_select_index`
- Menu 5: Modify `group_select_index`
- Menu 7: Modify `preset_select_index`
- Other menus: Modify `menu_index`

---

## Device Management

### Data Structures
```cpp
// Maximum 20 devices supported
int num_devices = 0;  // Actual count
std::vector<std::string> device_names;  // Display names
std::vector<std::string> device_entity_ids;  // HA entity IDs
std::vector<std::string> effect_lists[20];  // Effect list per device
int effect_counts[20];  // Effect count per device

// Selection tracking
int selected_device_index = 0;  // Currently selected single device
int device_select_index = 0;  // Scroll position in device list
std::vector<int> selected_devices;  // Multi-select temporary storage
```

### Device Sync from Home Assistant

**Template Sensor** (Home Assistant side):
```yaml
sensor.wled_device_list:
  attributes:
    devices: "Name1|entity.id1,Name2|entity.id2,..."
    effects: "Effect1,Effect2,Effect3,..."
```

**Parsing Logic** (ESPHome side):
1. Receive pipe-delimited device list
2. Split on `|` for name/entity pairs
3. Split on `,` for multiple devices
4. Store in parallel vectors
5. Update `num_devices` count
6. Parse effects list (comma-separated)
7. Copy same effect list to all devices

**Device Display**:
- Main menu shows current device name at top
- Device selection shows full list with checkboxes
- Checked devices show cyan checkbox (\U000F05E1)
- Unchecked show grey checkbox (\U000F0130)

### Multi-Device Selection

**Process**:
1. Enter device selection menu (menu 1)
2. Tap/press devices to toggle selection
3. Selected devices added to `selected_devices` vector
4. Press "Done" button (index == num_devices)

**Done Button Logic**:
```cpp
if (selected_devices.size() == 0) {
  // Nothing selected, return to main
  current_menu = 0; menu_index = 0;
}
else if (selected_devices.size() == 1) {
  // Single device: set as selected device, single mode
  selected_device_index = selected_devices[0];
  control_mode = 0;
  selected_devices.clear();
  current_menu = 0; menu_index = 0;
}
else {
  // Multiple devices: prompt to save or use once
  current_menu = 8; menu_index = 0;
}
```

**"Done" Button Visual**:
- 2x size checkbox (larger icon)
- Always green color
- Text: "Done" in font_large
- Shows selection count: "X selected"

---

## Group Management

### Data Structures
```cpp
int num_groups = 2;  // Starts with 2 default groups
std::string group_names[10];  // Max 10 groups
std::vector<int> group_devices[10];  // Device indices per group

// Selection
int selected_group_index = 0;  // Currently active group
int group_select_index = 0;  // Scroll position
int control_mode = 0;  // 0=single device, 1=group
```

**Default Groups**:
```cpp
group_names[0] = "All Devices";  // Contains all devices
group_names[1] = "Custom 1";     // Temporary group slot
```

### Group Creation

**Save Group Prompt** (menu 8):
Two options after multi-select:

1. **Use Once** (index 0):
   - Store devices in `group_devices[1]` (Custom 1)
   - Set `selected_group_index = 1`
   - Set `control_mode = 1`
   - Clear `selected_devices`
   - Return to main menu

2. **Save as Group** (index 1):
   - Create new permanent group at `group_devices[num_groups]`
   - Name: "Group N" where N = num_groups
   - Increment `num_groups`
   - Set `selected_group_index = num_groups - 1`
   - Set `control_mode = 1`
   - Clear `selected_devices`
   - Return to main menu

### Group Service Calls

When `control_mode == 1`, service calls use comma-separated entity_id:
```cpp
std::string entities = "";
for (int i = 0; i < group_devices[selected_group_index].size(); i++) {
  int dev_idx = group_devices[selected_group_index][i];
  if (dev_idx < num_devices) {
    if (i > 0) entities += ",";
    entities += device_entity_ids[dev_idx];
  }
}
// entities = "light.wled1,light.wled2,light.wled3"
```

**Group Display**:
- Main menu center shows: "Group: [name]" in green
- Group menu shows all saved groups
- Current group marked with ">" indicator

---

## State Management

### Global Variables (Persistent)

**Restored on Boot**:
```cpp
int selected_device_index = 0;  // Last selected device
int selected_group_index = 0;   // Last selected group
int control_mode = 0;            // 0=single, 1=group
int num_presets = 0;             // Number of saved presets
int preset_hues[10];             // Preset color values
int preset_effects[10];          // Preset effect indices
int preset_brightness[10];       // Preset brightness values
```

**Not Restored** (reset on boot):
```cpp
int menu_index = 0;              // Menu position
int current_menu = 0;            // Current menu ID
std::vector<std::string> device_names;
std::vector<std::string> device_entity_ids;
std::vector<int> selected_devices;
std::string group_names[10];
std::vector<int> group_devices[10];
```

### Temporary State

**Per-Menu State**:
```cpp
int device_select_index = 0;     // Device menu scroll
int color_hue = 0;                // Color wheel position
int brightness_value = 128;       // Brightness slider
int group_select_index = 0;       // Group menu scroll
int preset_select_index = 0;      // Preset menu scroll
```

**These reset when entering menu**, ensuring clean state.

### Menu State Preservation

**Rule**: When returning to main menu (menu 0), restore `menu_index` to the option that was selected.

**Examples**:
- Enter Color (index 2) → Pick color → Return to main at index 2
- Enter Effects (index 3) → Pick effect → Return to main at index 3
- Enter Device (index 0) → Select device → Return to main at index 0

**Exception**: Save Group prompt returns to index 0 (Select Device)

---

## Home Assistant Integration

### API Connection
```yaml
api:
  encryption:
    key: !secret encryptionKey
```

### Service Calls

**light.toggle**:
```yaml
service: light.toggle
data_template:
  entity_id: !lambda 'return "light.wled_device";'
```

**light.turn_on** (with parameters):
```yaml
service: light.turn_on
data_template:
  entity_id: !lambda 'return entity_id_string;'
  hs_color: !lambda 'return "[180, 100]";'  # [hue, saturation]
  brightness: !lambda 'return "255";'
  effect: !lambda 'return "Rainbow";'
```

**Group Control** (comma-separated entities):
```yaml
entity_id: !lambda 'return "light.wled1,light.wled2,light.wled3";'
```

### Template Sensor Requirements

**Required Attributes**:
1. `devices` - Pipe and comma delimited: "Name1|entity1,Name2|entity2"
2. `effects` - Comma delimited effect list from any WLED device

**Update Frequency**: Template updates on state change, ESPHome syncs on boot and when template changes

---

## Display Rendering

### Display Configuration
```yaml
display:
  platform: ili9xxx
  model: GC9A01A
  cs_pin: GPIO7
  dc_pin: GPIO4
  reset_pin: GPIO8
  invert_colors: true  # CRITICAL - display is inverted
  rotation: 0
  update_interval: 250ms
```

### Color System

**CRITICAL**: Display has inverted colors enabled, so standard RGB values work correctly.

**Standard Colors** (defined in globals):
```cpp
COLOR_BLACK:    RGB(0, 0, 0)      // Shows as black
COLOR_WHITE:    RGB(255, 255, 255) // Shows as white
COLOR_RED:      RGB(255, 0, 0)
COLOR_GREEN:    RGB(0, 255, 0)
COLOR_BLUE:     RGB(0, 128, 255)
COLOR_CYAN:     RGB(0, 255, 255)
COLOR_MAGENTA:  RGB(255, 0, 255)
COLOR_YELLOW:   RGB(255, 255, 0)
COLOR_ORANGE:   RGB(255, 128, 0)
COLOR_PURPLE:   RGB(191, 0, 255)
COLOR_GREY:     RGB(128, 128, 128)
COLOR_LIGHT_GREY: RGB(191, 191, 191)
COLOR_DARK_GREY:  RGB(64, 64, 64)
```

### HSV to RGB Conversion

**Script**: `convert_hsv_to_rgb`

**Parameters**:
- h: Hue (0-360)
- s: Saturation (0-100)
- v: Value/Brightness (0-100)
- r, g, b: Output references (0-255)

**Algorithm**: Standard HSV→RGB conversion with 6 sectors

**Usage**:
```cpp
uint8_t r, g, b;
id(convert_hsv_to_rgb).execute(180, 100, 100, r, g, b);
Color color = Color(r, g, b);
```

### Font Configuration

**Roboto Fonts** (local files):
```yaml
font_small:  Roboto-Regular.ttf, 14pt
font_medium: Roboto-Bold.ttf, 18pt
font_large:  Roboto-Bold.ttf, 24pt
```

**Material Design Icons** (remote, cached):
```yaml
font_icons: materialdesignicons-webfont.ttf, 28pt
  glyphs:
    \U000F0335  # devices
    \U000F0425  # power
    \U000F0E0C  # palette
    \U000F0766  # star-box
    \U000F00DF  # brightness
    \U000F1254  # groups
    \U000F1C2B  # bookmarks
    \U000F0FE2  # swap
    \U000F05E1  # checkbox-marked
    \U000F0130  # checkbox-blank-outline
```

### Rendering Pipeline

1. **Clear screen**: `it.fill(COLOR_BLACK)`
2. **Check device sync**: If `num_devices == 0`, show "Syncing WLED devices" message
3. **Render current menu**: Switch on `current_menu` ID
4. **Update display**: Automatic at 250ms intervals

---

## Touch System

### Touch Hardware

**Controller**: FT6336U capacitive touch
- I2C Address: 0x38
- Interrupt Pin: GPIO14 (not used)
- Reset Pin: GPIO13 (not used)

### Touch Data Format

**Register Read**:
```cpp
uint8_t data[16];
i2c_bus.read(0x38, data, 16);

// Extract coordinates
int touch_x = ((data[3] & 0x0F) << 8) | data[4];
int touch_y = ((data[5] & 0x0F) << 8) | data[6];
```

**Touch Count Byte** (unreliable):
- data[2] supposedly contains touch count
- IGNORED - unreliable, produces phantom values

### Touch Processing (When Enabled)

**Polling Rate**: 200ms interval (slower to reduce I2C bus load)

**Validation**:
1. Coordinates must be in range 5-235 (filter 0,0 phantoms and edges)
2. Coordinates must differ from last processed touch
3. Static tracking: `last_raw_x`, `last_raw_y`

**Phantom Touch Filtering**:
Common phantom coordinates:
- (0, 0) - filtered by range check
- (1025, 6) - filtered by range check
- (4095, 4095) - filtered by range check
- Repeating touches - filtered by coordinate change check

**Debouncing Strategy**:
```cpp
static int last_raw_x = -1;
static int last_raw_y = -1;

if (touch_x != last_raw_x || touch_y != last_raw_y) {
  // Process new touch
  last_raw_x = touch_x;
  last_raw_y = touch_y;
}
// Else: ignore repeated coordinate
```

### Touch Disabled by Default

**Reason**: I2C bus contention causes:
- NACK errors from touch controller
- Button press delays
- System instability

**Current State**: Touch interval component is commented out in YAML

**To Re-enable**: Uncomment the interval section, but expect:
- Periodic I2C errors in logs
- Possible button responsiveness issues
- Need to monitor system stability

---

## Scripts and Functions

### toggle_power_action Script

**Purpose**: Toggle power for current device or group

**Logic**:
```yaml
if control_mode == 0 (single device):
  service: light.toggle
  entity_id: device_entity_ids[selected_device_index]

if control_mode == 1 (group):
  service: light.toggle
  entity_id: "light.wled1,light.wled2,light.wled3"  # Comma-separated
```

### convert_hsv_to_rgb Script

**Purpose**: Convert HSV color to RGB for display

**Parameters**:
- h (int): Hue 0-360
- s (int): Saturation 0-100
- v (int): Value 0-100
- r, g, b (uint8_t&): Output RGB 0-255

**Usage**: All color rendering, especially color wheel

---

## Critical Design Rules

### ✅ DO

1. **Always clear screen** with `it.fill(COLOR_BLACK)` at start of lambda
2. **Preserve menu_index** when returning to main menu
3. **Execute service calls BEFORE menu navigation** in button handler
4. **Use inverted colors** - display has `invert_colors: true`
5. **Filter touch coordinates** - only accept 5-235 range
6. **Track coordinate changes** - ignore repeated touches
7. **Position selected item at 9 o'clock** (270°, left middle)
8. **Use contour-following X calculation** for circular edge alignment
9. **Validate all array access** with bounds checking
10. **Wrap hue values** at 0/360 boundary

### ❌ DON'T

1. **Don't use touch count byte** - unreliable on FT6336U
2. **Don't reset coordinates on invalid** - breaks duplicate detection
3. **Don't assume service calls are instant** - they're async
4. **Don't change menu state before service calls** - race condition
5. **Don't enable touch without testing** - causes I2C issues
6. **Don't use time-based debouncing for touch** - coordinate-based works better
7. **Don't forget +180 offset on color wheel** - needed for inverted display
8. **Don't use localStorage in artifacts** - not supported in claude.ai
9. **Don't mix single/group service call patterns** - use correct template
10. **Don't exceed array bounds** - max 10 groups, 20 devices, 10 presets

---

## Future Enhancements

### Planned Features

1. **Preset System**:
   - Save current color, effect, brightness as preset
   - 10 preset slots
   - Quick recall from preset menu
   - Preset naming

2. **Touch Re-enablement**:
   - Investigate I2C bus optimization
   - Possibly slower polling (500ms?)
   - Better error handling
   - Optional enable/disable via config

3. **Group Management**:
   - Rename groups
   - Delete groups
   - Edit group membership
   - Save/load from flash

4. **Advanced Color Control**:
   - Saturation adjustment
   - Brightness in color wheel
   - Color temperature (white balance)
   - RGB vs HSV mode

5. **Effect Parameters**:
   - Speed control per effect
   - Intensity adjustment
   - Effect-specific parameters

### Known Issues

1. **Touch Disabled**: I2C reliability problems, needs investigation
2. **Button Responsiveness**: Occasional delays, may be I2C related
3. **Effect List**: Uses first device's effects for all (should be per-device)
4. **Group Edit**: UI exists but editing not fully implemented

---

## Debugging and Diagnostics

### Log Levels
```yaml
logger:
  level: DEBUG
  logs:
    component: ERROR  # Reduce component spam
    display: DEBUG    # Show display updates
```

### Useful Log Messages

**Device Sync**:
```
[wled_sync] Received device list: Name1|entity1,Name2|entity2
[parse] Added device: Name1 (entity1)
[parse] Total devices: 5
```

**Button Presses**:
```
[button] Handler starting at 12345
[button] Menu: 0, Index: 2, Devices: 5
```

**Touch Events** (when enabled):
```
[touch] NEW TOUCH at x=120, y=180
[touch] Scroll down to 3
```

**Group Operations**:
```
[group] Toggle entities: light.wled1,light.wled2
```

### Common Issues

**No devices showing**:
- Check Home Assistant template sensor exists
- Verify WLED devices are in HA
- Check ESPHome API connection
- Look for parsing errors in logs

**Colors wrong**:
- Verify `invert_colors: true` in display config
- Check color wheel +180 offset
- Verify HSV conversion script

**Button not responsive**:
- Check for I2C NACK errors
- Verify GPIO42 wiring
- Check debounce settings
- Look for lambda blocking calls

**Display artifacts**:
- Reduce update_interval (try 500ms)
- Check SPI wiring
- Verify power supply (needs 500mA+)

---

## Version History

**v1.0** (Current):
- Circular main menu with 8 options
- Device selection with multi-select
- Color wheel with HSV picker
- Effect browser
- Brightness control
- Group creation and management
- Rotary encoder + button input
- Touch disabled (I2C issues)
- Home Assistant integration
- State preservation

---

*This document should be updated whenever significant features or behaviors are added or changed.*
