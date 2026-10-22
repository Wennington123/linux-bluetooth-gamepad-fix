# Case study: generic multi-mode gamepad (USB `045e:028e` / BT `1949:0402`)

A cheap PS3-shaped "generic" controller that works over USB but refuses to work
over Bluetooth on Linux. Everything below was diagnosed from real HCI traces, not
guesswork — the trace output is included so you can compare against your own
device.

## Device

| Transport | ID | Reported name |
|---|---|---|
| USB | `045e:028e` | `Microsoft Corp. Xbox360 Controller` |
| USB (transient, during enumeration) | `054c:0268` | `Nintendo Co., Ltd. USB Gamepad` (bound by `hid-sony`) |
| Bluetooth | `1949:0402` | `Gamepad` (class `0x00002504`) |

This is a ShanWan-style multi-mode board. The USB descriptor reports
*iManufacturer = "Nintendo Co., Ltd."* and *iProduct = "XBOX 360 For Windows"*
with serial `000000000001` — a strong hint that the "Xbox 360" identity is only a
USB compatibility mode, not the real chipset.

Over Bluetooth it is **not** a DualShock 3 and does **not** need `sixpair`. It
enumerates as a generic HID gamepad (`hid-generic`, `HID v1.1b Gamepad`) and
gets a normal `/dev/input/jsX`.

## Symptoms

- The controller appears in a scan as `Gamepad` and `pair` succeeds
  (`Paired: yes`, `Bonded: yes`).
- Immediately after pairing it may show `Connected: yes` for a second, then drop.
- Every later attempt fails with any of:

```
Failed to connect: org.bluez.Error.Failed br-connection-create-socket
bluetoothd: profiles/input/device.c:control_connect_cb() connect to <MAC>: Host is down (112)
```

- No `/dev/input/js*` is ever created.

## Diagnosis

Capture the HCI traffic while attempting to connect:

```bash
sudo btmon
# in another terminal
bluetoothctl connect <MAC>
```

Look for the `Connect Complete` event and its `Status`.

### Trace 1 — `Page Timeout (0x04)`

```
< HCI Command: Create Connection (0x01|0x0005) plen 13
> HCI Event: Command Status (0x0f) plen 4
      Status: Success (0x00)
> HCI Event: Connect Complete (0x03) plen 11
      Status: Page Timeout (0x04)
```

The controller is not answering pages. Two different causes produce this, and
they must be handled in order:

1. The controller is not in pairing mode (it is off, asleep, or already trying to
   page the host itself).
2. The controller *is* trying to connect to the host and the host is **not
   connectable**.

Check the adapter's current settings:

```bash
sudo btmgmt info | grep "current settings"
```

A healthy adapter lists `connectable`:

```
current settings: powered connectable bondable ssp br/edr le secure-conn ...
```

If `connectable` is missing, the host will refuse the controller's incoming
connection and the controller gives up. Fix it with:

```bash
sudo btmgmt connectable on
```

### Trace 2 — `Authentication Failure (0x05)`

Once the page succeeds, the link comes up and then dies a few milliseconds later:

```
> HCI Event: Connect Complete (0x03) plen 11
      Status: Success (0x00)
< HCI Command: Authentication Requested (0x01|0x0011) plen 2
      Status: Authentication Failure (0x05)
< HCI Command: Disconnect (0x01|0x0006) plen 3
      Reason: Remote User Terminated Connection (0x13)
```

This is a link-key mismatch: the controller still holds an old key while the host
created a new one. It happens after a `remove` + re-pair cycle, because removing
the device from BlueZ deletes the host's key but **the controller does not forget
its own**.

Fix: delete the host-side pairing and re-pair cleanly while the controller is
in fast-blink pairing mode, and make sure the adapter is connectable first.

## Root causes and fixes

| # | Cause | Fix |
|---|---|---|
| 1 | Adapter not `connectable` | `sudo btmgmt connectable on` |
| 2 | Stale link key on the controller (`Authentication Failure 0x05`) | `bluetoothctl remove <MAC>` then re-pair while blinking |
| 3 | BlueZ defaults to userspace HID, clones need kernel HIDP | `UserspaceHID=false` |
| 4 | Just-Works pairing does not persist the key on clones | `ClassicBondedOnly=false` (and/or legacy `agent KeyboardOnly` + PIN `0000`) |
| 5 | `bluetoothd` segfault on HID connect | update BlueZ; restart the service and retry |
| 6 | ERTM corrupting the HID connection | disable ERTM |
| 7 | USB autosuspend of the Bluetooth adapter | disable `btusb` autosuspend |

### `/etc/bluetooth/input.conf`

```ini
[General]
UserspaceHID=false
ClassicBondedOnly=false
LEAutoSecurity=false
```

> Note: the README recommends `UserspaceHID=true`. For this class of clone the
> kernel HIDP path (`false`) is what actually works. Try both — if `true` gives
> you `br-connection-create-socket`, switch to `false`.

### ERTM and autosuspend

```bash
# runtime
echo Y | sudo tee /sys/module/bluetooth/parameters/disable_ertm
echo 0 | sudo tee /sys/module/btusb/parameters/enable_autosuspend

# persistent
echo "options bluetooth disable_ertm=1" | sudo tee /etc/modprobe.d/bluetooth.conf
echo "options btusb enable_autosuspend=n" | sudo tee /etc/modprobe.d/btusb-autosuspend.conf
```

Optionally keep the adapter powered:

```bash
# /etc/udev/rules.d/50-bt-no-autosuspend.rules
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="8087", ATTR{idProduct}=="0026", ATTR{power/control}="on"
```

Replacing the IDs with your own adapter (`lsusb`).

## Working sequence

```bash
MAC=<GAMEPAD_MAC>

# 1. Make sure the adapter accepts incoming connections
sudo btmgmt connectable on

# 2. Clean state
bluetoothctl remove "$MAC"
bluetoothctl power on
bluetoothctl pairable on

# 3. Put the controller in pairing mode (LED blinking fast), then:
bluetoothctl --timeout 60 scan on      # confirm "<name>" shows up
bluetoothctl pair "$MAC"
bluetoothctl trust "$MAC"

# 4. Wait a moment: the controller may connect by itself
bluetoothctl info "$MAC" | grep Connected
```

If it still fails, repeat with the legacy agent:

```bash
bluetoothctl
agent KeyboardOnly
default-agent
pair "$MAC"
# PIN: 0000
trust "$MAC"
```

Expected result:

```
$ bluetoothctl info "$MAC" | grep -E "Paired|Trusted|Connected"
	Paired: yes
	Trusted: yes
	Connected: yes

$ dmesg | grep -i gamepad
hid-generic 0005:1949:0402.0003: input,hidraw1: BLUETOOTH HID v1.1b Gamepad [Gamepad] on <host-bt-mac>

$ ls -l /dev/input/js*
crw-rw-r--+ 1 root input 13, 0 ... /dev/input/js0
```

## Notes

- **It will not be recognised as an Xbox 360 controller over Bluetooth.** The
  `045e:028e` identity only exists in USB mode. Over Bluetooth the kernel exposes
  a generic gamepad (`hid-generic`). If a game insists on XInput, use Steam Input
  (it emulates an Xbox 360 pad) or `xboxdrv --mimic-xpad`.
- **Reconnecting:** once the device is `Paired` + `Trusted` and the adapter is
  `connectable`, pressing the Home button should reconnect without any command.
  If it stops working after a kernel/BlueZ update, redo the clean re-pair — the
  usual culprit is a stale link key on the controller.
- **Sniffing tip:** `btmon` needs root and only shows new traffic; start it
  *before* issuing `connect`.
- The `Gamepad` name is extremely common on cheap controllers, and these boards
  are often in pairing mode near each other. If you are not sure the device you
  found is yours, turn yours off and rescan: the entry should disappear.
