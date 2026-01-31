# TODO

- [ ] Guard against zero effects before clamping `menu_index` in `/mnt/e/code/AI Code/M5Dial-EspHome/m5dial.yaml:261-270`. Validation: select a device/group with 0 effects; rotate dial; confirm index stays at 0 and no crash; effect list remains empty.
- [ ] Use correct device effect count in effect-click guard in `/mnt/e/code/AI Code/M5Dial-EspHome/m5dial.yaml:390-393`. Validation: pick device A with more effects than device 0; rotate to last effect; click; confirm effect applies.
- [ ] Prevent negative preset index when no presets exist in `/mnt/e/code/AI Code/M5Dial-EspHome/m5dial.yaml:551-555`. Validation: ensure 0 presets; rotate in preset menu; confirm index stays at 0 and no preset array access.

## Test Checklist

- [ ] Effects menu with zero effects: select device/group with 0 effects, rotate dial in both directions, confirm menu index clamps to 0 and no crash.
- [ ] Effects menu with different counts: device 0 has fewer effects than selected device, rotate to highest index on selected device, click to apply, confirm effect changes.
- [ ] Presets menu with zero presets: enter presets menu, rotate dial, confirm index stays at 0 and no preset array access.
