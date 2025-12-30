# ESP32 Wireless Development Starter

**Go wireless from day one.** One USB flash, then never touch a cable again.

This is a comprehensive template and instructions for Claude Code to set up a complete wireless ESP32 development environment. After the initial USB flash, all development happens over WiFi - no more cables needed.

## Features

- **OTA Updates**: Push firmware wirelessly - device auto-updates within 60 seconds
- **Remote Logging**: All `Serial.print` output sent to your server via syslog
- **Captive Portal**: Easy WiFi configuration via phone/laptop
- **Double-Reset Recovery**: Press RESET twice to reconfigure WiFi (never bricked)
- **Automatic Rollback**: Bad firmware? ESP32 reverts after 3 failed boots
- **Multi-Board Support**: Wemos D1 Mini, XIAO ESP32-C6, generic ESP32 DevKit

## Quick Start

1. Start a new ESP32 project in VS Code
2. Open Claude Code
3. Say: **"Set up ESP32 wireless dev environment using this guide"** and share this file
4. Answer Claude's questions about your setup
5. Run `./deploy.sh --usb` once (USB flash)
6. Configure WiFi via captive portal
7. Never touch a USB cable again - just run `./deploy.sh`

## What Gets Created

```
your-project/
├── src/
│   ├── main.cpp                 # Main application with OTA + logging
│   ├── config.h                 # All configuration in one place
│   ├── network/
│   │   ├── wifi_manager.cpp     # WiFi + captive portal
│   │   └── ota_update.cpp       # OTA polling + update
│   ├── utils/
│   │   └── debug_log.cpp        # Serial + remote syslog
│   └── storage/
│       └── nvs_manager.cpp      # Persistent settings
├── platformio.ini               # Multi-board build config
├── deploy.sh                    # One command deployment
├── CLAUDE.md                    # Instructions for future Claude sessions
└── README.md                    # User documentation
```

## Requirements

- **ESP32 Board**: Wemos D1 Mini ESP32, XIAO ESP32-C6, or generic ESP32 DevKit
- **Development Server**: Raspberry Pi (recommended), Mac, or Linux box on your network
- **PlatformIO**: Installed in VS Code or CLI
- **SSH Key**: Passwordless SSH to your server

## How It Works

```
┌─────────────┐         ┌──────────────────┐
│   ESP32     │  HTTP   │  Your Server     │
│  (polling)  │ ──────► │  (Pi/Mac/Linux)  │
└─────────────┘         └──────────────────┘
     │                         │
     │ GET /version.txt        │
     │ ◄────────────────────── │ "project:1.0.5-dev"
     │                         │
     │ (if different)          │
     │ GET /firmware.bin       │
     │ ◄────────────────────── │ (binary)
     │                         │
     ▼                         │
  OTA Flash & Reboot           │
     │                         │
     │ UDP Syslog (logs)       │
     │ ────────────────────►   │ → esp32.log
```

1. **ESP32 boots** and connects to WiFi
2. **Checks server** for new version (`version.txt`)
3. **Downloads & flashes** if version differs
4. **Sends logs** to server via UDP syslog
5. **Repeats** every 60 seconds

## Supported Boards

| Board | Status | Notes |
|-------|--------|-------|
| **Wemos D1 Mini ESP32** | Recommended | Stable WiFi, easy to source |
| **XIAO ESP32-C6** | Supported | Requires pioarduino for SSL |
| **ESP32 DevKit** | Supported | Generic ESP32 boards |

### XIAO ESP32-C6 Notes

The C6 requires special handling:
- Uses pioarduino platform fork for SSL support
- Deep sleep wake only from GPIO 0-7
- Uses `NetworkClientSecure` instead of `WiFiClientSecure`

All of this is handled automatically by the template.

## Development Workflow

```bash
# Edit code, bump version in config.h
./deploy.sh

# Watch it happen
ssh pi@192.168.x.x 'tail -f ~/projects/your-project/logs/esp32.log'

# See: [BOOT] your-project v1.0.6-dev
# See: [OTA] Firmware is up to date
```

## WiFi Configuration

**First time**: Device creates AP named `your-project-XXXX`. Connect and configure.

**Later**: Double-press RESET within 3 seconds to re-enter config mode.

## Server Requirements

Just needs:
- SSH access from your dev machine
- Python 3 (for HTTP server and syslog listener)
- Port 8080 (HTTP) and 514 (syslog UDP)

The `deploy.sh` script automatically:
- Starts HTTP server if not running
- Starts syslog listener if not running
- Creates directory structure

## Troubleshooting

### OTA not updating
```bash
# Check server is accessible
curl http://your-server:8080/version.txt

# Should return: your-project:X.X.X-dev
```

### No logs appearing
```bash
# Check syslog listener
ssh user@server 'ss -uln | grep :514'

# If not running, deploy.sh starts it automatically
./deploy.sh --skip-build
```

### Device keeps rebooting
- Wait for automatic rollback (after 3 failed boots)
- Or flash via USB: `./deploy.sh --usb`

## Production Deployment

When ready for production, disable development features in `config.h`:

```cpp
#define LOCAL_DEPLOY_ENABLED  0   // Disable OTA polling
#define SYSLOG_ENABLED        0   // Disable remote logging
```

## License

MIT License - Use freely in your projects.

## Contributing

Issues and PRs welcome at [GitHub](https://github.com/ksorbo/esp32-wireless-dev-starter).

---

**Created from real-world ESP32 development experience.** This template was extracted from an IoT project after realizing how much time was wasted plugging/unplugging USB cables during development.
