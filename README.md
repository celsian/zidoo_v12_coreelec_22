# Fix the Zidoo V12 Remote in CoreELEC 22

## Background

In CoreELEC 21, the Zidoo V12 appeared as a single HID device. In CoreELEC 22, it now exposes itself as **two** devices: a keyboard **and** a mouse. To make matters worse, the keyboard node is also misclassified as a tablet, which causes libinput to silently discard keypresses.

The fix disables both the mouse input and the keyboard's tablet classification.

---

## Steps

### 1. Pair the Remote

Connect the Zidoo V12 to your playback device via Bluetooth.

### 2. SSH into CoreELEC

- Enable SSH: Settings -> CoreELEC -> Services -> SSH -> Enable SSH

Default credentials:

- **User:** `root`
- **Password:** `coreelec`

### 3. Confirm Both Devices Exist

```sh
cat /proc/bus/input/devices | grep -A2 "Remote-V12"
```

You should see both `Remote-V12 Keyboard` and `Remote-V12 Mouse`.

### 4. Install the udev Rule

```sh
mkdir -p /storage/.config/udev.rules.d

cat > /storage/.config/udev.rules.d/99-zidoo-v12.rules <<'EOF'
# Hide the V12 mouse HID interface from Kodi/libinput so air-mouse
# mode does not produce a cursor and does not steal input from Kodi.
SUBSYSTEM=="input", ATTRS{name}=="Remote-V12 Mouse", \
  ENV{LIBINPUT_IGNORE_DEVICE}="1", \
  ENV{ID_INPUT_MOUSE}="0", \
  ENV{ID_INPUT}="0"

# The V12 keyboard HID descriptor also declares REL/ABS axes, which
# makes systemd misclassify it as a tablet. libinput then silently
# discards its keypresses. Force it to be treated as a plain keyboard.
SUBSYSTEM=="input", ATTRS{name}=="Remote-V12 Keyboard", \
  ENV{ID_INPUT_TABLET}="0", \
  ENV{ID_INPUT_TABLET_PAD}="0", \
  ENV{ID_INPUT_JOYSTICK}="0", \
  ENV{ID_INPUT_MOUSE}="0", \
  ENV{ID_INPUT_KEYBOARD}="1", \
  ENV{ID_INPUT_KEY}="1", \
  ENV{ID_INPUT}="1"
EOF

udevadm control --reload-rules
udevadm trigger --subsystem-match=input --action=change
```

### 5. Re-pair the Remote

Disconnect the Zidoo V12 from Bluetooth, then reconnect.

> **Note:** If the remote refuses to reconnect after disconnecting, delete the pairing entirely and pair from scratch.

### 6. Verify Device Properties

Look up the current `event*` numbers:

```sh
cat /proc/bus/input/devices | grep -A2 "Remote-V12"
```

Substitute the correct numbers below and check properties:

```sh
udevadm info -q property -p /sys/class/input/eventN | grep -E 'ID_INPUT|LIBINPUT'
```

**Expected output for the keyboard node:**

```
ID_INPUT=1
ID_INPUT_TABLET=0
ID_INPUT_TABLET_PAD=0
ID_INPUT_KEY=1
ID_INPUT_KEYBOARD=1
ID_INPUT_JOYSTICK=0
ID_INPUT_MOUSE=0
```

**Expected output for the mouse node:**

```
ID_INPUT=0
ID_INPUT_MOUSE=0
ID_INPUT_KEY=1
LIBINPUT_IGNORE_DEVICE=1
```

### 7. Create the HWDB File

```sh
nano /storage/.config/hwdb.d/99-zidoo-v12.hwdb
```

Paste the contents from [99-zidoo-v12.hwdb](./99-zidoo-v12.hwdb).

### 8. Apply the HWDB and Restart Kodi

```sh
systemd-hwdb update
udevadm trigger --subsystem-match=input --action=change
systemctl restart kodi
```
