# Linux Bluetooth Gamepad Connection Fix

Struggling to reconnect your Bluetooth gamepad on Linux? You're not alone.

After days of tinkering with Bluetooth profiles, I finally solved it. Here's my solution.

**My setup:** Linux Mint 22.3, GuliKit King Kong 2 Pro

**The problem:** Gamepad pairs and works fine, but after disconnecting it won't reconnect. I had to re-pair every single time.

---

## The Fix

I needed both steps below to get reliable reconnection working.

### Step 1: Pair Using Legacy Mode

The GUI uses "Just Works" pairing, which many gamepads handle poorly—they pair but don't save the link key. Force legacy PIN-based pairing instead:

```bash
bluetoothctl
remove [GAMEPAD_MAC]
power off
power on
agent KeyboardOnly
default-agent
scan on
# Put your controller in pairing mode now
pair [GAMEPAD_MAC]
trust [GAMEPAD_MAC]
connect [GAMEPAD_MAC]
```

If prompted for a PIN, enter `0000`.

### Step 2: Modify Bluetooth Input Config

Edit `/etc/bluetooth/input.conf` (create it if it doesn't exist):

```bash
sudo nano /etc/bluetooth/input.conf
```

Add the following:

```ini
[General]
UserspaceHID=true
ClassicBondedOnly=false
LEAutoSecurity=false
```

Restart Bluetooth:

```bash
sudo systemctl restart bluetooth
```

---

## Credits

Step 2 sourced from [this Reddit comment](https://www.reddit.com/r/linux_gaming/comments/tdh3gq/comment/kjxg9bk/).
