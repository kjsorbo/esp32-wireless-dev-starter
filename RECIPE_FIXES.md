# ESP32 Wireless Dev Starter - Recipe Fixes

This document tracks all issues encountered when using the ESP32_WIRELESS_DEV_STARTER.md recipe and the fixes applied. These should be incorporated back into the recipe.

---

## Issue 1: NVS Namespace Too Long

**Error:**
```
[E][Preferences.cpp:47] begin(): nvs_open failed: KEY_TOO_LONG
```

**Cause:** The recipe calculated NVS namespace as first 15 chars of project name, but `mqtt-weight-esp3` was actually 16 characters.

**Fix in config.h:**
```cpp
// WRONG (16 chars)
#define NVS_NAMESPACE "mqtt-weight-esp3"

// CORRECT (13 chars, max is 15)
#define NVS_NAMESPACE "mqtt-wt-esp32"
```

**Recipe Update Needed:** The recipe should truncate to 13-14 chars max and verify the length.

---

## Issue 2: esp_task_wdt_init() API Changed in Arduino-ESP32 3.x

**Error:**
```
error: invalid conversion from 'int' to 'const esp_task_wdt_config_t*'
error: too many arguments to function 'esp_err_t esp_task_wdt_init(const esp_task_wdt_config_t*)'
```

**Cause:** The recipe used the old Arduino-ESP32 2.x API:
```cpp
esp_task_wdt_init(WATCHDOG_TIMEOUT_SEC, true);
```

But Arduino-ESP32 3.x (framework-arduinoespressif32 @ 3.0.7) changed the API.

**Fix:** Don't manually initialize the watchdog - Arduino framework already does it. Just add the current task:
```cpp
// OLD (recipe code - doesn't work with Arduino-ESP32 3.x)
esp_task_wdt_config_t wdt_config = {
    .timeout_ms = WATCHDOG_TIMEOUT_SEC * 1000,
    .idle_core_mask = (1 << portNUM_PROCESSORS) - 1,
    .trigger_panic = true
};
esp_task_wdt_init(&wdt_config);
esp_task_wdt_add(NULL);

// NEW (correct for Arduino-ESP32 3.x)
// Arduino framework already initialized WDT, just add loopTask
esp_task_wdt_add(NULL);
```

**Recipe Update Needed:** Remove manual WDT init, just use `esp_task_wdt_add(NULL)`.

---

## Issue 3: WiFiClient Changed to NetworkClient in Arduino-ESP32 3.x

**Error:**
```
error: 'WiFiClient' was not declared in this scope
```

**Cause:** Arduino-ESP32 3.x renamed `WiFiClient` to `NetworkClient`.

**Fix in ota_update.cpp:**
```cpp
// Add include
#include <NetworkClient.h>

// Change type
// OLD
WiFiClient* stream = http.getStreamPtr();

// NEW
NetworkClient* stream = http.getStreamPtr();
```

**Recipe Update Needed:** Use `NetworkClient` instead of `WiFiClient`, add `#include <NetworkClient.h>`.

---

## Issue 4: esp_ota_mark_app_valid_cancel_rollback() Include Location

**Error:**
```
error: 'esp_ota_mark_app_valid_cancel_rollback' was not declared in this scope
```

**Cause:** The recipe had the include inside an if block which caused parsing issues:
```cpp
if (connected) {
    #include <esp_ota_ops.h>  // BAD - include inside block
    esp_ota_mark_app_valid_cancel_rollback();
}
```

**Fix in main.cpp:**
```cpp
// Move include to top of file with other includes
#include <esp_ota_ops.h>

// Then just call the function where needed
esp_ota_mark_app_valid_cancel_rollback();
```

**Recipe Update Needed:** Move `#include <esp_ota_ops.h>` to the top includes section.

---

## Issue 5: Watchdog Triggers During WiFi Connection

**Error:**
```
E (10787) task_wdt: Task watchdog got triggered. The following tasks/users did not reset the watchdog in time:
E (10787) task_wdt:  - loopTask (CPU 1)
```

**Cause:** The `wifiConnect()` function has a blocking while loop waiting for connection (up to 15 seconds) without feeding the watchdog.

**Fix in wifi_manager.cpp:**
```cpp
while (WiFi.status() != WL_CONNECTED && millis() - startTime < WIFI_TIMEOUT) {
    esp_task_wdt_reset();  // ADD THIS LINE - Feed watchdog during connection
    delay(500);
    DEBUG_PRINT(".");
    attempts++;
}
```

**Recipe Update Needed:** Add `esp_task_wdt_reset()` inside the WiFi connection loop.

---

## Issue 6: Watchdog Triggers in Config Portal Loop

**Error:** Same watchdog error as Issue 5.

**Cause:** The `wifiStartPortal()` blocking loop didn't feed the watchdog.

**Fix in wifi_manager.cpp:**
```cpp
// Add include at top
#include <esp_task_wdt.h>

// Add in portal loop
while (portalRunning) {
    esp_task_wdt_reset();  // ADD THIS LINE - Feed watchdog
    dnsServer.processNextRequest();
    server.handleClient();
    // ...
}
```

**Recipe Update Needed:** Add watchdog feeding in portal loop and include `<esp_task_wdt.h>`.

---

## Issue 7: Double-Reset Detection Causes Boot Loop

**Error:** Device boot loops with garbled serial output, eventually enters config mode incorrectly.

**Cause:**
1. The `checkDoubleReset()` function has a 3-second blocking delay
2. The watchdog was added AFTER this function was called
3. RTC_NOINIT_ATTR variables can have garbage values on first boot

**Fix in main.cpp:**
```cpp
// Check for double-reset (enter config mode)
// Uses RTC memory to detect if reset happened within 3 seconds
bool checkDoubleReset() {
    // Feed watchdog during this check
    esp_task_wdt_reset();

    unsigned long now = millis();

    // If reset happened within 3 seconds of last reset
    // Also validate resetCount is reasonable (not garbage)
    if (resetCount > 0 && resetCount < 10 && (now < 3000)) {
        resetCount = 0;
        return true;
    }

    // Set reset count to 1 (not increment - avoids garbage accumulation)
    resetCount = 1;

    // Wait 3 seconds, feeding watchdog periodically
    for (int i = 0; i < 30; i++) {
        esp_task_wdt_reset();
        delay(100);
    }

    resetCount = 0;
    return false;
}
```

**Recipe Update Needed:**
1. Feed watchdog inside checkDoubleReset()
2. Validate resetCount is reasonable (not garbage)
3. Use loop with periodic WDT feeding instead of single delay(3000)

---

## Issue 8: Watchdog Triggers During OTA Download

**Error:**
```
[OTA] Downloading: http://192.168.86.220:8080/firmware.bin
[OTA] Firmware size: 1121232 bytes
E (23032) task_wdt: Task watchdog got triggered. The following tasks/users did not reset the watchdog in time:
E (23032) task_wdt:  - loopTask (CPU 1)
```

**Cause:** The `Update.writeStream()` call is blocking and takes 10-30+ seconds to download and flash 1MB+ firmware. The default watchdog timeout is ~5 seconds.

**Fix in ota_update.cpp:**
```cpp
// Add include at top
#include <esp_task_wdt.h>

// Before the blocking download
esp_task_wdt_delete(NULL);  // Remove task from WDT
debugLog("[OTA] Watchdog disabled for download...");

NetworkClient* stream = http.getStreamPtr();
size_t written = Update.writeStream(*stream);
http.end();

// Re-enable watchdog after download
esp_task_wdt_add(NULL);
```

**Recipe Update Needed:**
- Add `#include <esp_task_wdt.h>` to ota_update.cpp
- Disable watchdog before `writeStream()`, re-enable after

---

## Issue 9: FIRMWARE_VERSION in platformio.ini Overrides config.h

**Error:**
```
[OTA] Update SUCCESS! Rebooting...
[BOOT] mqtt-weight-esp32-stole v1.0.0-dev   <-- Still shows old version!
```

**Cause:** The recipe puts `FIRMWARE_VERSION` in both `platformio.ini` build_flags AND `config.h`. The build_flags `-D` takes precedence because it's defined at compile time before the header is processed. The `#ifndef` guard in config.h doesn't help because the macro is already defined.

```ini
# platformio.ini
build_flags =
    -DFIRMWARE_VERSION=\"1.0.0-dev\"  # THIS WINS
```

```cpp
// config.h
#ifndef FIRMWARE_VERSION
#define FIRMWARE_VERSION  "1.0.4-dev"  // Never used!
#endif
```

**Fix:** Keep version in sync in BOTH files, or remove from one location.

**Recipe Update Needed:** Either:
1. Only define FIRMWARE_VERSION in platformio.ini (single source of truth), OR
2. Remove from platformio.ini and only use config.h, OR
3. Document clearly that BOTH must be updated together

---

## Issue 10: Syslog Listener Script Never Created + Port 514 Conflict

**Error:** No logs appearing in project's `logs/esp32.log` file despite device running and syslog enabled.

**Cause (deploy.sh logic flaw):** The deploy.sh script has flawed syslog detection logic:

```bash
# deploy.sh lines 268-270
SYSLOG_RUNNING=$(ssh ... "ss -uln | grep ':514 '" ...)
if [ -z "$SYSLOG_RUNNING" ]; then
    # Create and start listener - ONLY runs if port 514 is FREE
else
    echo "Syslog listener already running"  # ASSUMES it's ours!
fi
```

**The problem:**
1. It only checks if **any** process is listening on port 514
2. If port 514 is in use (by ANY project), it assumes "our" listener is running
3. It **never creates the script** for this project if another project's listener exists
4. Result: `scripts/` directory is empty, logs go to the wrong project

```bash
$ ls -la /home/osmc/projects/mqtt-weight-esp32-stole/scripts/
total 8
drwxr-xr-x 2 osmc osmc 4096 Dec 31 16:03 .
drwxr-xr-x 5 osmc osmc 4096 Dec 31 16:03 ..
# EMPTY! Script was never created because roborock-button listener was running!
```

**What happened in this case:**
```bash
$ pgrep -a -f syslog_listener
23505 python3 /home/osmc/projects/roborock-button/scripts/syslog_listener.py  # WRONG PROJECT!
```

The roborock-button listener was receiving ALL syslog messages and writing them to its own log directory, not mqtt-weight-esp32-stole's.

**Fix Options:**

1. **Shared multi-project listener** (recommended): Create a single syslog listener that routes messages to the correct project based on the project name in the syslog message. The ESP32 already includes project name: `<14>mqtt-weight-esp32-stole: message`

2. **Different ports per project**: Each project uses a unique syslog port (514, 515, 516, etc.) - requires changing both server listener and ESP32 config

3. **Central syslog daemon**: Use rsyslog/syslog-ng with filtering rules

**Recipe Update Needed:**
1. The recipe MUST check if port 514 is already in use before starting a listener
2. The recipe SHOULD use a shared multi-project listener approach
3. The deploy.sh script should detect and warn about port conflicts
4. Document this limitation prominently in the recipe

**Shared Listener Solution:**
```python
#!/usr/bin/env python3
# /home/osmc/projects/shared_syslog_listener.py
# Routes syslog messages to project-specific logs based on project name

import socket
import os
import re
from datetime import datetime

BASE_DIR = "/home/osmc/projects"
PROJECT_PATTERN = re.compile(r"<\d+>([^:]+):\s*(.*)")

def main():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind(("0.0.0.0", 514))

    while True:
        data, addr = sock.recvfrom(1024)
        msg = data.decode("utf-8", errors="ignore").strip()

        match = PROJECT_PATTERN.match(msg)
        if match:
            project_name = match.group(1)
            message = match.group(2)
        else:
            project_name = "unknown"
            message = msg

        log_dir = os.path.join(BASE_DIR, project_name, "logs")
        os.makedirs(log_dir, exist_ok=True)
        log_file = os.path.join(log_dir, "esp32.log")

        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        with open(log_file, "a") as f:
            f.write(f"[{timestamp}] [{addr[0]}] {message}\n")

if __name__ == "__main__":
    main()
```

---

## Summary of All Required Recipe Updates

### platformio.ini
- Keep FIRMWARE_VERSION in sync with config.h (or only define in one place)

### config.h
- Ensure NVS_NAMESPACE is max 13-14 chars (not 15)

### main.cpp
- Move `#include <esp_ota_ops.h>` to top includes
- Remove manual `esp_task_wdt_init()` - just use `esp_task_wdt_add(NULL)`
- Fix `checkDoubleReset()` to feed watchdog and validate RTC values

### wifi_manager.cpp
- Add `#include <esp_task_wdt.h>`
- Add `esp_task_wdt_reset()` in WiFi connection loop
- Add `esp_task_wdt_reset()` in config portal loop

### ota_update.cpp
- Add `#include <NetworkClient.h>`
- Add `#include <esp_task_wdt.h>`
- Change `WiFiClient*` to `NetworkClient*`
- Disable watchdog during OTA download with `esp_task_wdt_delete(NULL)` / `esp_task_wdt_add(NULL)`

### Server Infrastructure (syslog)
- **CRITICAL**: Recipe must handle multiple projects sharing port 514
- Use a shared multi-project syslog listener instead of per-project listeners
- deploy.sh must check for port conflicts and either:
  - Kill existing listener and start shared one, OR
  - Warn user about conflict
- The shared listener routes by project name in syslog message

---

## Issue 11: Claude Starts Wrong Syslog Listener

**Error:** Logs from mqtt-weight-esp32-stole appearing in roborock-button's log file with raw syslog format:
```
[2026-01-01 17:35:04] [192.168.86.41] <14>mqtt-weight-esp32-stole: [WEIGHT] Raw: 332233
```

**Cause:** In a previous Claude session, Claude manually started the OLD per-project syslog listener:
```bash
# Claude ran this command (WRONG!)
ssh osmc@192.168.86.220 'cd /home/osmc/projects/roborock-button/scripts && sudo nohup python3 syslog_listener.py'
```

This old listener:
1. Binds to port 514 and captures ALL syslog traffic from ALL ESP32 devices
2. Writes everything to `/home/osmc/projects/roborock-button/logs/esp32.log`
3. Does NOT parse project names - just dumps raw messages

**Evidence from journalctl:**
```
Jan 01 17:35:04 osmc sudo[1454]: osmc : PWD=/home/osmc/projects/roborock-button/scripts ; USER=root ; COMMAND=/usr/bin/nohup python3 syslog_listener.py
```

**Fix Applied:**
1. Updated deploy.sh to **always** kill old per-project listeners before checking shared listener:
```bash
# First, kill any OLD per-project syslog listeners
OLD_LISTENERS=$(ssh ... "pgrep -f 'projects/.*/scripts/syslog_listener'" ...)
if [ -n "$OLD_LISTENERS" ]; then
    ssh ... "sudo pkill -f 'projects/.*/scripts/syslog_listener'" ...
fi
```

2. Added clear documentation to CLAUDE.md about syslog infrastructure rules

**Recipe Update Needed:**
- deploy.sh must proactively kill per-project listeners, not just check port 514
- Document that old per-project `scripts/syslog_listener.py` files should be deleted or ignored

---

## Issue 12: Server Reboot Leaves No Syslog Listener Running

**Error:** After server reboot, no logs are captured until deploy.sh is run.

**Cause:** The syslog listener is started by deploy.sh, not as a system service. After reboot:
1. No syslog listener is running
2. ESP32 sends logs to port 514 but nothing receives them
3. Logs are lost until someone runs deploy.sh

**Design Decision:** This is intentional - the system is "sandboxed" with no permanent Pi configuration. Each deploy.sh run ensures the correct listener is running.

**Mitigation:**
- deploy.sh always ensures the shared listener is running
- Running deploy.sh after reboot restores logging
- The shared listener handles all projects, so any project's deploy.sh fixes it for everyone

---

## CRITICAL: Instructions for Claude AI Sessions

### Syslog Infrastructure - DO NOT BREAK

The server uses a **SHARED** syslog listener at `/home/osmc/projects/shared_syslog_listener.py`.

**NEVER run these commands:**
```bash
# WRONG - starts per-project listener that breaks everything
ssh osmc@192.168.86.220 'cd /home/osmc/projects/*/scripts && python3 syslog_listener.py'
ssh osmc@192.168.86.220 'sudo python3 /home/osmc/projects/roborock-button/scripts/syslog_listener.py'
```

**ALWAYS use deploy.sh to manage syslog:**
```bash
# CORRECT - deploy.sh handles everything
./deploy.sh --skip-build
```

### How the Shared Listener Works

1. Location: `/home/osmc/projects/shared_syslog_listener.py`
2. Parses project name from syslog: `<priority>project-name: message`
3. Routes to: `/home/osmc/projects/{project-name}/logs/esp32.log`
4. Started by deploy.sh, runs as root (needs sudo for port 514)

### Checking Syslog Status

```bash
# Check what's running (should see shared_syslog_listener.py ONLY)
ssh osmc@192.168.86.220 'ps aux | grep syslog | grep -v grep'

# GOOD output:
# root 1888 ... python3 shared_syslog_listener.py

# BAD output (per-project listener - will break things):
# root 1234 ... python3 /home/osmc/projects/roborock-button/scripts/syslog_listener.py
```

### If Logs Are Going to Wrong Place

```bash
# Fix: run deploy.sh - it kills wrong listeners and starts shared one
./deploy.sh --skip-build
```

---

## Issue 13: Legacy Projects Use Per-Project Syslog Listeners

**Problem:** Older projects (e.g., roborock-button) created before the shared syslog listener was implemented still have deploy.sh scripts that create per-project listeners.

**Behavior of Legacy deploy.sh:**
1. Checks if port 514 is in use
2. If port 514 is busy → assumes syslog is running, does nothing (partially safe)
3. If port 514 is free → starts its own per-project listener (breaks other projects)

**Impact:**
- If shared listener is already running: Legacy deploy works fine (sees port busy, skips)
- If shared listener is NOT running: Legacy deploy starts per-project listener that captures ALL syslog traffic and writes only to its own log file

**Why This Matters for the Recipe:**
The ESP32_WIRELESS_DEV_STARTER.md recipe MUST generate deploy.sh scripts that use the shared listener pattern. This ensures:
1. New projects always use shared listener
2. New projects kill any old per-project listeners before starting shared listener
3. Deploying any new project fixes the infrastructure for all projects

**Correct deploy.sh Pattern (for recipe):**
```bash
# 1. Kill any old per-project listeners
ssh user@server "pkill -f 'projects/.*/scripts/syslog_listener'" 2>/dev/null || true

# 2. Check if shared listener is running
SHARED_RUNNING=$(ssh user@server "pgrep -f 'shared_syslog_listener'" 2>/dev/null)

# 3. If not running, start it
if [ -z "$SHARED_RUNNING" ]; then
    # Create/update shared_syslog_listener.py and start it
    ...
fi
```

**Backwards Compatibility:**
- Legacy projects continue to work when deployed AFTER a new project
- Legacy projects may break syslog if deployed when port 514 is free
- Solution: Update legacy projects to use shared listener pattern (optional but recommended)

---

## Version History (Recipe Fixes Only)

These versions represent fixes to get the base wireless dev starter working. Application-specific features (sensors, MQTT, etc.) are not tracked here.

| Version | Recipe Issue Fixed |
|---------|-------------------|
| 1.0.0-dev | Initial from recipe (broken) |
| 1.0.1-dev | Fixed NVS namespace, WDT init, NetworkClient (Issues 1-4) |
| 1.0.2-dev | Added WDT feeding in WiFi connect loop (Issue 5) |
| 1.0.3-dev | Fixed double-reset detection with WDT feeding (Issue 7) |
| 1.0.4-dev | Disable WDT during OTA download (Issue 8) |
| 1.0.5-dev | Fixed version in platformio.ini (Issue 9) |
| N/A (server) | Fixed syslog port conflict (Issue 10) |
| N/A (server) | deploy.sh kills old per-project listeners (Issue 11) |

**After 1.0.5-dev, the base recipe is working.** Any further versions are application-specific.
