 # FS25 on SteamOS: troubleshooting wheels, pedals, joysticks and side panels

This guide is for the Steam version of Farming Simulator 25 running through Proton on SteamOS/Linux. It covers input detection for devices such as HORI farming controllers, Logitech wheels, and other USB simulator hardware. It is a troubleshooting process, not a guarantee that every model or hardware mode is supported.

**Change one thing at a time, restart FS25 after each change, and stop when the device works.** Record your original Steam Input setting, Proton version and launch options so you can undo a test. Back up `inputBinding.xml` before changing existing bindings.

## 1. Check whether Linux detects the hardware

Switch to Desktop Mode, connect and power on the equipment, then open Konsole:

```bash
lsusb
cat /proc/bus/input/devices
```

`lsusb` shows USB IDs in the form `VID:PID`, for example `1234:abcd`. Note the ID and name for each separately connected wheel, pedal set or panel. Pedals plugged into a wheel may appear as axes of the wheel instead of a separate USB device.

In `/proc/bus/input/devices`, look for the device name and its `eventN` handler. A USB listing proves enumeration, not that Linux has decoded every control correctly.

If the device is missing from USB entirely, check power, cables, USB ports and the manufacturer's hardware mode before changing Proton. If it appears in USB but has no normal input entry, collect kernel messages; HIDRAW testing may still be relevant later.

If `evtest` is installed, test the matching event device:

```bash
sudo evtest /dev/input/eventN
```

Replace `eventN` with the actual handler. Move the wheel, press each pedal and test buttons. Device numbers can change after reconnecting or rebooting.

If evtest reports that another process has grabbed the device, identify processes with open handles:

```bash
sudo fuser -v /dev/input/eventN
```

An open handle does not prove that a process holds the exclusive grab. Close remappers or configuration applications normally and retest. Apps that create a virtual controller can also cause duplicate devices. Test with those apps stopped first.

## 2. Test Steam Input and Proton separately

In Steam, open **FS25 → Properties → Controller** and test **Disable Steam Input**. This can help simulator devices reach the game without a gamepad mapping in between.

This is a per-game setting and may affect your ordinary gamepad too. Restore the original setting if the test does not help or removes a controller feature you need.

Under **Properties → Compatibility**, record the selected Proton version. Compare a stable Proton release with Experimental if detection still fails. Do not assume a newer release always handles your hardware better.

Proton 10.0 worked in the setup that prompted this guide; that is a tested example, not a requirement for every wheel or panel.

## 3. Check what FS25 actually sees

Make sure gamepad/joystick input is enabled in FS25. Check its controller binding screen and the current `log.txt` in the FarmingSimulator2025 user-data folder inside the game's Proton prefix.

Look for the **Input System** section and later device-added messages. A device can be added after the initial startup list, so check the whole log.

If a panel with many buttons appears only as `XINPUT_GAMEPAD` with 14 buttons, it may be reaching the game through a gamepad mapping. If a Steam Controller or Xbox controller is also connected, an XInput entry may simply be that controller—identify which device responds before changing anything.

Detection, automatic factory bindings and force feedback are separate checks. A generic device may need manual bindings even when all its inputs reach the game.

## 4. Optional: test native HIDRAW access

Use this when Linux detects the hardware but FS25 has missing controls, incorrect axes or no usable device. Some multi-interface devices have reported detection problems that change when HIDRAW is enabled.

In **FS25 → Properties → General → Launch Options**, add your device's actual USB ID:

```text
PROTON_ENABLE_HIDRAW=0x1234/0xabcd %command%
```

**Replace `1234` and `abcd` with the VID and PID from your own `lsusb` output.** Do not paste an ID from someone else's wheel.

For multiple separately enumerated devices, use a comma-separated list:

```text
PROTON_ENABLE_HIDRAW=0x1234/0xabcd,0x5678/0xef01 %command%
```

These are placeholder IDs. List only hardware you are testing. Preserve other launch options that you already need, and keep environment variables **before** `%command%`. Putting the variable after `%command%` can pass it to the game as an argument instead of setting it for Proton.

Restart FS25 and compare the log and controls. Remove the variable if it makes detection worse. Protontricks launches are a separate test: do not assume they automatically inherit FS25's Steam launch options.

### If HIDRAW permissions are blocking access

First identify the matching HIDRAW node. In Konsole, list each HIDRAW device with its name and hardware ID:

```bash
for device in /sys/class/hidraw/hidraw*; do
    echo "/dev/${device##*/}"
    sed -n '/^HID_ID=/p; /^HID_NAME=/p' "$device/device/uevent"
    echo
done
```

Example output:

```text
/dev/hidraw6
HID_ID=0003:0000046D:0000C262
HID_NAME=Logitech G920 Driving Force Racing Wheel
```

Match the name and VID/PID to your `lsusb` result. In this example, the last two fields of `HID_ID` identify vendor `046d` and product `c262`, matching `046d:c262` in `lsusb`. The matching node is therefore `/dev/hidraw6`.

Replace `/dev/hidrawN` in the commands below with your actual node. A device can have multiple HIDRAW entries; check each matching entry. Repeat identification after reconnecting or rebooting because the numbers can change.

To inspect a candidate node in more detail:

```bash
udevadm info --attribute-walk --name=/dev/hidrawN
```

Match `idVendor` and `idProduct` to your device. Do not assume `/dev/hidraw6` or another number belongs to the same device on another PC.

Check access as your normal user:

```bash
test -r /dev/hidrawN && test -w /dev/hidrawN && echo "Access OK" || echo "Access missing"
```

Only if access is missing, a device-specific udev rule can grant the active local user access. A rule file under `/etc/udev/rules.d/`, such as `70-fs25-controller.rules`, can contain:

```text
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1234", ATTRS{idProduct}=="abcd", TAG+="uaccess"
```

Replace both IDs, use lowercase hex without `0x`, and add one line per necessary VID/PID pair. Creating the file requires administrator privileges. Reload rules and reconnect the hardware:

```bash
sudo udevadm control --reload-rules
```

Then recheck access and relaunch the game. Do not grant every HIDRAW device world-writable access. If SteamOS reports a read-only filesystem while saving the rule, stop and get instructions for your specific OS version rather than disabling protection as a general troubleshooting step.

## 5. Optional: override a simulator device from XInput to DirectInput

Install Protontricks through Discover if needed. Close FS25, then run:

```bash
flatpak run com.github.Matoking.protontricks -c "wine control joy.cpl" 2300320
```

For a non-Flatpak installation:

```bash
protontricks -c "wine control joy.cpl" 2300320
```

`2300320` is FS25's Steam app ID. These commands open the controller panel in its Proton prefix.

If the affected wheel, joystick or side panel appears under **Connected (XInput devices)**, select that specific device and click **Override**. Close and reopen the panel, then restart FS25 and test again. Leave ordinary gamepads under XInput unless you have a separate reason to change them.

Override requests DirectInput handling; it does not install a Linux driver, grant HIDRAW permissions, create factory bindings or reserve joystick slot 0. Skip it if the game already receives the full controls. If it makes things worse, select the overridden device and use **Reset** to remove the override.

## 6. Model-specific checks

| Hardware | What to check |
|---|---|
| HORI wheel/pedals and side panel | Identify every separately connected USB device. Test each device and axis independently; fixing the wheel does not prove the panel is working. Confirm the supported hardware mode in that model's manual. |
| Logitech wheels | Identify the exact model and current USB ID. Models can use different Linux drivers and hardware modes. If steering/pedals fail in Linux, investigate that layer before Proton. If Linux input works but FS25 input fails, compare Proton and HIDRAW handling. |
| Separate pedals, shifters or button boxes | Give each separately enumerated USB device its own test. Do not assume the wheel's VID/PID covers them. |
| Virtual/remapped controllers | Stop the remapper for a baseline test. Determine whether the game sees the physical device, the virtual device or both. |

**Force feedback is a separate problem.** Successful steering and buttons do not guarantee FFB. Investigate the exact wheel's Linux driver and the game's support once basic inputs work.

## 7. What to include when asking for help

Share:

- Exact device models and hardware modes.
- SteamOS version, `uname -r`, and selected Proton version.
- Desktop Mode, Gaming Mode, or both.
- Relevant `lsusb` and `/proc/bus/input/devices` entries.
- Whether Linux input tests respond correctly.
- Steam Input setting and exact launch options.
- Whether Wine lists the device under DirectInput or XInput.
- FS25's Input System and device-added log lines.
- Whether the issue is detection, missing buttons, axes, duplicate devices, bindings or FFB.

Redact serial numbers and personal paths if posting publicly. Report which single change helped so other people can reproduce it.

## Sources and scope

- [Protontricks installation and usage](https://github.com/Matoking/protontricks)
- [Wine controller-panel override implementation](https://github.com/wine-mirror/wine/blob/master/dlls/joy.cpl/main.c)
- [Proton 10 HID DirectInput implementation](https://github.com/ValveSoftware/wine/blob/proton_10.0/dlls/dinput/joystick_hid.c)
- [FS25 Logitech steering-axis report](https://github.com/ValveSoftware/Proton/issues/8671)
- [Proton report concerning multi-interface devices and HIDRAW](https://github.com/ValveSoftware/Proton/issues/9034)

Issue reports describe particular setups and are not proof of a fix for every device. This guide has not been hardware-tested across all HORI and Logitech models.
