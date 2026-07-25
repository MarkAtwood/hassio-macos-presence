# ha-macos-presence

A simple macOS daemon that reports presence state to Home Assistant.

Works on **any Mac** — desktop or laptop, ethernet or WiFi. No location services required.

## Why?

The official Home Assistant Companion app uses WiFi network names for presence detection, which doesn't work on:
- Desktop Macs with ethernet
- Macs without WiFi
- Networks where SSID-based detection is unreliable

This daemon monitors actual system state: screen lock, idle time, and active application.

## What it reports

| Entity | Type | Description |
|--------|------|-------------|
| `binary_sensor.macos_presence_<host>_active` | occupancy | On when: unlocked + screen on + idle < threshold |
| `binary_sensor.macos_presence_<host>_screen_locked` | lock | Screen lock / screensaver state |
| `binary_sensor.macos_presence_<host>_screen_on` | binary | Display power state |
| `sensor.macos_presence_<host>_idle_time` | sensor | Seconds since last keyboard/mouse input |
| `sensor.macos_presence_<host>_frontmost_app` | sensor | Currently active application |
| `sensor.macos_presence_<host>_apple_id` | sensor | Logged-in Apple ID (iCloud account) |
| `sensor.macos_presence_<host>_console_user` | sensor | macOS username at console |

## Requirements

- macOS 10.15 Catalina or later
- Home Assistant instance with REST API access
- Long-lived access token (create in HA → Profile → Long-Lived Access Tokens)

No additional software required — uses only built-in macOS tools (`curl`, `python3`, `osascript`, `ioreg`).

## Installation

### 1. Download

```bash
# Option A: Clone the repo
git clone https://github.com/MarkAtwood/hassio-macos-presence.git
cd hassio-macos-presence

# Option B: Download just the script
curl -O https://raw.githubusercontent.com/MarkAtwood/hassio-macos-presence/main/ha-presence
chmod +x ha-presence
```

### 2. Install to a permanent location

macOS security blocks launchd from running scripts in Downloads or external volumes. Copy to a standard location:

```bash
mkdir -p ~/.local/bin
cp ha-presence ~/.local/bin/
chmod +x ~/.local/bin/ha-presence
```

### 3. Configure

Run the interactive setup:

```bash
~/.local/bin/ha-presence --setup
```

You'll be prompted for:

| Setting | Description | Example |
|---------|-------------|---------|
| HA Server URL | Your Home Assistant instance | `http://192.168.1.10:8123` |
| Access Token | Long-lived token from HA | `eyJhbGciOiJI...` |
| Device Name | How this Mac appears in HA | `Office iMac` |
| Poll Interval | Seconds between updates | `30` |
| Idle Threshold | Seconds before "active" turns off | `300` |

Configuration is saved to `~/.config/ha-presence/config` (mode 600).

### 4. Test

Run once to verify it works:

```bash
~/.local/bin/ha-presence --once
```

Check Home Assistant — you should see new entities like `binary_sensor.macos_presence_office_imac_active`.

### 5. Install as auto-start daemon

```bash
~/.local/bin/ha-presence --install
```

This creates a launchd user agent that:
- Starts automatically on login
- Restarts if it crashes
- Logs to `/tmp/ha-presence.log`

### 6. Grant permissions

On first run, macOS will prompt:

> "Terminal wants to control System Events"

Click **OK** — this allows the script to detect which application is in front. No other permissions are required.

If you missed the prompt, grant it manually:
- System Settings → Privacy & Security → Automation
- Find Terminal (or your terminal app) → enable "System Events"

## Usage

```bash
ha-presence --setup      # Interactive configuration
ha-presence --once       # Update states once and exit
ha-presence --daemon     # Run continuously (used by launchd)
ha-presence --install    # Install as launchd agent
ha-presence --uninstall  # Remove launchd agent
ha-presence --help       # Show help
```

## Logs

View daemon logs:

```bash
tail -f /tmp/ha-presence.log
```

## Example automations

### Turn off office lights when Mac is inactive

```yaml
automation:
  - alias: "Office lights off when Mac inactive"
    trigger:
      - platform: state
        entity_id: binary_sensor.macos_presence_office_imac_active
        to: "off"
        for: "00:05:00"
    action:
      - service: light.turn_off
        target:
          area_id: office
```

### Notify when workstation is locked

```yaml
automation:
  - alias: "Workstation locked notification"
    trigger:
      - platform: state
        entity_id: binary_sensor.macos_presence_office_imac_screen_locked
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          message: "Workstation locked"
```

### Presence-based heating

```yaml
automation:
  - alias: "Office heating when at desk"
    trigger:
      - platform: state
        entity_id: binary_sensor.macos_presence_office_imac_active
    action:
      - service: climate.set_temperature
        target:
          entity_id: climate.office
        data:
          temperature: "{{ 72 if trigger.to_state.state == 'on' else 65 }}"
```

## How it works

The daemon polls macOS every N seconds:

1. **Idle time** — `ioreg -c IOHIDSystem` reads HIDIdleTime (seconds since last input)
2. **Screen lock** — checks if ScreenSaverEngine is running or loginwindow is frontmost
3. **Display power** — `CoreGraphics.CGDisplayIsOnline()` via Python ctypes
4. **Frontmost app** — AppleScript query to System Events

States are pushed to Home Assistant via the REST API (`POST /api/states/<entity_id>`).

The launchd agent runs as your user, starts on login, and auto-restarts on failure.

## Uninstall

```bash
# Stop and remove the daemon
~/.local/bin/ha-presence --uninstall

# Remove the script
rm ~/.local/bin/ha-presence

# Remove configuration
rm -rf ~/.config/ha-presence
```

Entities in Home Assistant will become "unavailable" and can be removed via the UI.

## Troubleshooting

### "States updated" but nothing in HA

- Verify your server URL is correct (include port)
- Check the token is valid (test with `curl -H "Authorization: Bearer $TOKEN" $SERVER/api/`)
- Check firewall isn't blocking the connection

### Frontmost app shows "unknown"

- Grant Automation permission: System Settings → Privacy & Security → Automation → Terminal → System Events

### Daemon not starting

- Check logs: `tail /tmp/ha-presence.log`
- Verify script location: `launchctl list | grep ha.presence`
- Exit code 126 = permission denied, copy script to `~/.local/bin/`

### Screen lock not detected

Detection works via ScreenSaverEngine process or loginwindow being frontmost. If you lock without screensaver (e.g., hot corners), it may not detect until the screensaver activates.

## License

MIT License — see [LICENSE](LICENSE)
