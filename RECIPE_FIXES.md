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

## Version History

| Version | Changes |
|---------|---------|
| 1.0.0-dev | Initial from recipe (broken) |
| 1.0.1-dev | Fixed NVS namespace, WDT init, NetworkClient |
| 1.0.2-dev | Added WDT feeding in WiFi connect loop |
| 1.0.3-dev | Fixed double-reset detection with WDT feeding |
| 1.0.4-dev | Disable WDT during OTA download |
| 1.0.5-dev | Fixed version in platformio.ini (was overriding config.h) |
| N/A (server) | Fixed syslog: deploy.sh skipped script creation due to port conflict |
