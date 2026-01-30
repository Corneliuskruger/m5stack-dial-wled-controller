# M5Stack Dial WLED Controller

A beautiful, modern touchscreen controller for WLED devices using the M5Stack Dial (ESP32-S3). Features a circular UI with rotating menus, color wheel, device groups, and multi-device control.

![M5Stack Dial](https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/docs/products/core/M5Dial/img-6daf09b5-a89f-4ab6-8de2-fb0d76cf67e0.webp)

## Features

- 🎨 **Circular Color Wheel** - Intuitive HSV color selection with live preview
- 📱 **Modern Circular UI** - Rotating menu with icons at 9 o'clock position
- 👆 **Touch + Dial Control** - Full touchscreen support plus rotary encoder
- 🔘 **Multi-Device Selection** - Control multiple WLED devices simultaneously
- 👥 **Device Groups** - Create and save groups for easy control
- ⚡ **Effects Browser** - Navigate through all WLED effects
- 💡 **Brightness Control** - Visual arc slider for brightness adjustment
- 🔖 **Presets** - Save and recall favorite settings (coming soon)
- 🔄 **Auto-Discovery** - Automatically syncs WLED devices from Home Assistant

## Hardware Requirements

- **M5Stack Dial** (ESP32-S3)
  - 1.28" round touchscreen (240x240 GC9A01A display)
  - Rotary encoder with button
  - FT6336U touch controller
- **Home Assistant** with WLED integration
- **USB-C cable** for initial programming

## Software Prerequisites

### 1. Install Python and pip (Ubuntu/WSL)
```bash
# Update package list
sudo apt update

# Install Python and pip
sudo apt install python3 python3-pip python3-venv -y

# Verify installation
python3 --version
pip3 --version
```

### 2. Install ESPHome
```bash
# Install ESPHome
pip3 install esphome

# Add to PATH (add to ~/.bashrc for permanent)
export PATH="$HOME/.local/bin:$PATH"

# Verify ESPHome is installed
esphome version
```

## Project Setup

### 1. Clone the Repository
```bash
git clone https://github.com/Corneliuskruger/m5stack-dial-wled-controller.git
cd m5stack-dial-wled-controller
```

### 2. Create secrets.yaml
```bash
# Copy the example file
cp secrets.yaml.example secrets.yaml

# Edit with your credentials
nano secrets.yaml
```

Fill in your actual values:
```yaml
wifi_ssid: "YourWiFiName"
wifi_password: "YourWiFiPassword"
encryptionKey: "WILL_BE_GENERATED"
ota_password: "create-a-secure-password"
```

### 3. Generate Encryption Key
```bash
# This will validate config and generate the encryption key
esphome config m5dial.yaml
```

Copy the generated `encryptionKey` and update your `secrets.yaml`.

### 4. Compile the Firmware
```bash
# Compile (first time takes longer - downloads dependencies)
esphome compile m5dial.yaml
```

### 5. Initial Upload via USB
```bash
# Connect M5Stack Dial via USB-C
# Upload firmware
esphome upload m5dial.yaml

# Monitor logs
esphome logs m5dial.yaml
```

### 6. Future Updates (OTA via WiFi)

Once connected to WiFi, you can update wirelessly:
```bash
esphome upload m5dial.yaml
```

## Home Assistant Configuration

Add this to your Home Assistant `configuration.yaml`:
```yaml
template:
  - sensor:
      - name: "WLED Device List"
        unique_id: wled_devices_v2
        state: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
        attributes:
          devices: >
            {% set ns = namespace(devices=[]) %}
            {% for state in states.light %}
              {% if 'wled' in state.entity_id and state.attributes.effect_list is defined %}
                {% set friendly_name = state.attributes.friendly_name | default(state.entity_id.split('.')[1] | replace('_', ' ') | title) %}
                {% set ns.devices = ns.devices + [friendly_name ~ '|' ~ state.entity_id] %}
              {% endif %}
            {% endfor %}
            {{ ns.devices | join(',') if ns.devices | length > 0 else 'none' }}
          effects: >
            {% for state in states.light %}
              {% if 'wled' in state.entity_id and state.attributes.effect_list is defined %}
                {{ state.attributes.effect_list | join(',') }}
                {% break %}
              {% endif %}
            {% endfor %}
```

Restart Home Assistant to activate the template sensor.

## Usage Guide

### Main Menu (Circular Layout)

- **Rotate dial** - Navigate through menu options
- **Press dial** - Select current option
- **Touch screen** - Tap anywhere to interact

Menu Options:
1. 🔘 **Select Device** - Choose which WLED device(s) to control
2. ⚡ **Toggle Power** - Turn selected device(s) on/off
3. 🎨 **Change Color** - Open color wheel
4. ✨ **Change Effect** - Browse WLED effects
5. 💡 **Brightness** - Adjust brightness
6. 👥 **Groups** - Manage device groups
7. 🔖 **Presets** - Save/load presets
8. 🔄 **Control Mode** - Toggle single/group control

### Device Selection

- **Scroll** - Rotate dial or touch top/bottom of screen
- **Select/Deselect** - Press dial or tap device
- **Done** - Select multiple devices, press "Done" to create temporary group or save

### Color Wheel

- **Choose color** - Rotate dial or touch color ring
- **Confirm** - Press dial or tap center circle
- Shows live preview, hue value (0-360°), and color name

### Device Groups

Groups allow controlling multiple devices simultaneously:

1. Select multiple devices
2. Choose "Save as Group" when done
3. Groups appear in the Groups menu
4. Switch to group control mode to use

## Hardware Pinout Reference

| Component | GPIO Pin |
|-----------|----------|
| Display CS | GPIO7 |
| Display DC | GPIO4 |
| Display RST | GPIO8 |
| Display CLK | GPIO6 |
| Display MOSI | GPIO5 |
| Backlight PWM | GPIO9 |
| Touch SDA | GPIO11 |
| Touch SCL | GPIO12 |
| Encoder A | GPIO40 |
| Encoder B | GPIO41 |
| Encoder Button | GPIO42 |

## Troubleshooting

### Device won't connect to WiFi
- Check `secrets.yaml` credentials
- Verify WiFi is 2.4GHz (ESP32 doesn't support 5GHz)
- Check router firewall settings

### No devices showing
- Ensure Home Assistant template sensor is configured
- Verify WLED devices are integrated in Home Assistant
- Check ESPHome logs: `esphome logs m5dial.yaml`

### Touch not responsive
- Touch uses I2C polling, may have slight delay
- Firmware includes debouncing - rapid touches filtered
- Scroll by touching top/bottom thirds of screen

### Button presses missed
- Check logs for "Button pressed" messages
- Debounce set to 10ms - very quick presses may be filtered
- Use firm, deliberate presses

### Compilation errors
- Ensure all fonts downloaded: `ls fonts/`
- Check Python version: `python3 --version` (need 3.7+)
- Try: `pip3 install --upgrade esphome`

## Development

### Project Structure
```
m5stack-dial-wled-controller/
├── m5dial.yaml           # Main ESPHome configuration
├── secrets.yaml          # Your credentials (not in git)
├── secrets.yaml.example  # Template for secrets
├── fonts/                # Roboto font files
│   ├── Roboto-Regular.ttf
│   └── Roboto-Bold.ttf
├── .gitignore           # Git ignore rules
└── README.md            # This file
```

### Making Changes
```bash
# Edit configuration
nano m5dial.yaml

# Validate changes
esphome config m5dial.yaml

# Compile and upload
esphome upload m5dial.yaml

# Monitor logs
esphome logs m5dial.yaml
```

### Commit Changes
```bash
git add .
git commit -m "Description of changes"
git push
```

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

MIT License - feel free to use and modify for your projects!

## Credits

- Built with [ESPHome](https://esphome.io/)
- Hardware by [M5Stack](https://m5stack.com/)
- Icons from [Material Design Icons](https://materialdesignicons.com/)
- Font: [Roboto](https://fonts.google.com/specimen/Roboto)

## Acknowledgments

Created with assistance from Claude (Anthropic) - an AI assistant that helped design the circular UI, implement touch controls, and debug hardware integration issues.

---

**Enjoy your M5Stack Dial WLED Controller!** 🎨✨
