# TODO

## Completed

- [x] Guard against zero effects before clamping `menu_index` in rotary handlers
- [x] Use correct device effect count in effect-click guard
- [x] Prevent negative preset index when no presets exist
- [x] Populate "All Devices" group (group 0) after device sync
- [x] Enforce 20-device limit (clamp and resize vectors)
- [x] Fix group device index validation in all 4 entity_id builders
- [x] Initialize group_names array slots 2-9 to empty strings in on_boot
- [x] Clean stale group device indices after device sync
- [x] Add bounds checks on effect_counts/effect_lists array access in display lambda
- [x] Use device_names.size() instead of num_devices for display bounds checks
- [x] Fix effect display using wrong device index in group mode
- [x] Replace broken HSV-to-RGB script (reference params don't work in ESPHome scripts) with inline C++ lambda
- [x] Validate restored globals on boot (selected_group_index, control_mode, num_presets)
- [x] Add sqrt() guard against negative argument in contour-following layout
- [x] Fix menu 6 dead-end trap (group edit had no display or exit path)
- [x] Add long-press (>1s) back navigation from any submenu
- [x] Fix long-press restoring incorrect menu_index when submenus reused it
- [x] Remove device name header that overlapped submenu titles
- [x] Validate device_select_index when device list syncs from HA
- [x] Clear selected_devices on entry to device selection menu
- [x] Fix selected_devices not cleared when "Save as Group" hits max capacity (10 groups)
- [x] Fix group Done button forcing control_mode=1 without actual selection change
- [x] Regenerate preset names on boot (preset_names can't be restored from flash)
- [x] Remove "or touch" text from color wheel (touch is disabled)
- [x] Main menu wraps around on circular layout
- [x] Redesign group selection to match device selection (contour-following layout, checkboxes, Done button, scroll indicators, device counts)
- [x] Remove Presets and Mode from main menu (8 items reduced to 6)
- [x] Increase color wheel scroll speed (3 deg to 10 deg per click)
- [x] Remove unused multi_select_mode global
- [x] Remove unreachable preset menu code and handlers

## Open

- [ ] Group mode HA service calls use comma-separated entity_ids which may not work via the ESPHome native API; needs testing, may require individual calls per device
- [ ] "Use Once" on save group prompt silently overwrites "Custom 1" group without warning
- [ ] Groups created at runtime are lost on reboot (group_names and group_devices cannot use restore_value)
- [ ] No way to delete groups once created
- [ ] Menu 8 "Save as Group" gives no feedback when group limit (10) is reached

## Test Checklist

- [ ] Main menu: rotate past last item (Groups), confirm it wraps to first item (Devices) and vice versa
- [ ] Color wheel: rotate dial, confirm hue changes in ~10-degree steps and feels responsive
- [ ] Group selection: enter Groups menu, confirm contour-following list with checkboxes, scroll through groups, tap to select, press Done to confirm
- [ ] Group selection cancel: enter Groups menu, long-press back, confirm control_mode unchanged
- [ ] Group selection no-change: enter Groups menu, press Done without selecting, confirm control_mode unchanged
- [ ] Device selection: enter Devices menu, confirm checkboxes start cleared, select 2+ devices, press Done, confirm save group prompt
- [ ] Long-press back: enter each submenu (Color, Effect, Brightness, Groups, Devices), long-press, confirm return to correct main menu item
- [ ] Effects menu in group mode: switch to group mode, enter Effects, confirm correct effect list displayed
- [ ] 20+ device limit: configure HA to return 21+ devices, confirm only first 20 are used
- [ ] Reboot persistence: create a group, reboot, confirm selected_group_index and control_mode are restored but group data is reset
