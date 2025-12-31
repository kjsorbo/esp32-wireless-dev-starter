# ESP32 Wireless Development Starter

**Purpose**: Instructions for Claude Code to set up a complete wireless ESP32 development environment. After initial USB flash, all development happens over WiFi - no more cables needed.

**Version**: 1.0.1
**Last Updated**: 2024-12-31

---

## How to Use This File

1. Start a new VS Code project for your ESP32 project
2. Start a new Claude Code conversation
3. Say: "Set up ESP32 wireless dev environment using this guide" and paste/reference this file
4. Answer Claude's questions about your setup
5. Claude creates everything, you do ONE USB flash, then go wireless forever

---

## Phase 1: Information Gathering

**Claude must ask the user these questions before proceeding:**

### Question 1: Project Name
```
What is your project name?
- Will be used for: folder names, NVS namespace, AP SSID
- Example: "greenhouse-sensor", "garage-door", "pool-monitor"
- Keep it short (max 20 chars), lowercase, use hyphens not spaces
```

**Sanitization rules:**
- Convert to lowercase
- Replace spaces with hyphens
- Remove special characters except hyphens
- Truncate to 20 characters
- This becomes `PROJECT_NAME`

### Question 2: ESP32 Board Type
```
Which ESP32 board are you using?

1. Wemos D1 Mini ESP32 (recommended - stable WiFi)
   - LED: GPIO 2, active HIGH
   - Boot button: GPIO 0

2. XIAO ESP32-C6 (requires special platform for SSL)
   - LED: GPIO 15, active LOW
   - Boot button: GPIO 9
   - NOTE: Deep sleep wake only from GPIO 0-7

3. Generic ESP32 DevKit
   - LED: GPIO 2, active HIGH
   - Boot button: GPIO 0

4. Other (specify pins)
```

**Store as:** `BOARD_TYPE` (wemos-esp32 | xiao-esp32c6 | esp32dev | custom)

### Question 3: Development Server
```
Where will the development server run?

1. Raspberry Pi (recommended - always on)
   - IP address: ___
   - Username: ___
   - Example: 192.168.86.220, osmc

2. Mac (your development machine)
   - Will use localhost or local IP
   - Server stops when Mac sleeps

3. Other Linux server
   - IP address: ___
   - Username: ___
```

**Store as:** `SERVER_TYPE` (pi | mac | linux), `SERVER_HOST`, `SERVER_USER`

### Question 4: SSH Key Setup
```
Do you have SSH key authentication set up for the server?

Check with: ssh {user}@{host} 'echo OK'

If it asks for password, we need to set up SSH keys first.
```

**If no SSH key:**
```bash
# Generate key (if you don't have one)
ssh-keygen -t ed25519 -C "esp32-dev"

# Copy to server
ssh-copy-id {user}@{host}

# Test (should not ask for password)
ssh {user}@{host} 'echo OK'
```

---

## Phase 2: Project Structure Creation

**Claude creates these files:**

### Directory Structure
```
{project-name}/
├── src/
│   ├── main.cpp                 # Main application
│   ├── config.h                 # All configuration
│   ├── network/
│   │   ├── wifi_manager.cpp     # WiFi connection + captive portal
│   │   ├── wifi_manager.h
│   │   ├── ota_update.cpp       # OTA polling + update
│   │   └── ota_update.h
│   ├── utils/
│   │   ├── debug_log.cpp        # Serial + syslog logging
│   │   └── debug_log.h
│   └── storage/
│       ├── nvs_manager.cpp      # NVS storage
│       └── nvs_manager.h
├── data/
│   └── config.html              # WiFi configuration page
├── platformio.ini               # PlatformIO configuration
├── deploy.sh                    # Build + deploy script
├── CLAUDE.md                    # Instructions for Claude Code
└── README.md                    # User documentation
```

---

## Phase 3: File Templates

### 3.1 platformio.ini

```ini
; PlatformIO Configuration for {PROJECT_NAME}
; Wireless development environment with OTA updates

[env]
; Common settings for all boards
framework = arduino
monitor_speed = 115200
lib_deps =
    ; Add project-specific libraries here

; Partition scheme for OTA (two app slots + NVS + SPIFFS)
board_build.partitions = min_spiffs.csv

build_flags =
    -DPROJECT_NAME=\"{PROJECT_NAME}\"
    -DFIRMWARE_VERSION=\"1.0.0-dev\"
    ; Stack size for SSL/network operations
    -DARDUINO_LOOP_STACK_SIZE=16384

;; ============================================================================
;; BOARD CONFIGURATIONS
;; ============================================================================

[env:wemos-esp32]
; Wemos D1 Mini ESP32 - Recommended for stable WiFi
platform = espressif32
board = wemos_d1_mini32
build_flags =
    ${env.build_flags}
    -DBOARD_WEMOS_ESP32=1
    -DLED_PIN=2
    -DLED_ON=HIGH
    -DLED_OFF=LOW
    -DBOOT_BTN=0

[env:xiao-esp32c6]
; XIAO ESP32-C6 - Requires pioarduino for SSL support
; CRITICAL: Standard espressif32 has NO SSL on C6!
platform = https://github.com/pioarduino/platform-espressif32/releases/download/51.03.07/platform-espressif32.zip
board = seeed_xiao_esp32c6
build_flags =
    ${env.build_flags}
    -DBOARD_XIAO_ESP32C6=1
    -DLED_PIN=15
    -DLED_ON=LOW
    -DLED_OFF=HIGH
    -DBOOT_BTN=9
    ; C6 uses NetworkClientSecure, not WiFiClientSecure
    -DUSE_NETWORK_CLIENT_SECURE=1

[env:esp32dev]
; Generic ESP32 DevKit
platform = espressif32
board = esp32dev
build_flags =
    ${env.build_flags}
    -DBOARD_ESP32DEV=1
    -DLED_PIN=2
    -DLED_ON=HIGH
    -DLED_OFF=LOW
    -DBOOT_BTN=0
```

### 3.2 src/config.h

```cpp
/**
 * @file config.h
 * @brief Global configuration for {PROJECT_NAME}
 * @generated by ESP32 Wireless Dev Starter
 */

#ifndef CONFIG_H
#define CONFIG_H

// ============================================================================
// PROJECT IDENTIFICATION
// ============================================================================

#ifndef PROJECT_NAME
#define PROJECT_NAME      "{PROJECT_NAME}"
#endif

#ifndef FIRMWARE_VERSION
#define FIRMWARE_VERSION  "1.0.0-dev"
#endif

// ============================================================================
// NETWORK SETTINGS
// ============================================================================

// Access Point (config mode)
#define AP_SSID_PREFIX    PROJECT_NAME "-"
#define AP_PASSWORD       ""              // Open AP for easy setup
#define AP_CHANNEL        6
#define AP_MAX_CONN       4

// WiFi Connection
#define WIFI_TIMEOUT      15000           // 15 seconds to connect
#define WIFI_RETRY_COUNT  3               // Retries before entering config mode

// ============================================================================
// DEVELOPMENT SERVER (OTA + Logging)
// ============================================================================

// Server configuration - set by deploy.sh, matches your setup
#define DEPLOY_SERVER_HOST    "{SERVER_HOST}"
#define DEPLOY_SERVER_PORT    8080
#define DEPLOY_POLL_INTERVAL  60          // Seconds between OTA checks

// Remote syslog
#define SYSLOG_HOST           "{SERVER_HOST}"
#define SYSLOG_PORT           514

// Enable/disable development features
#define LOCAL_DEPLOY_ENABLED  1           // Set to 0 for production
#define SYSLOG_ENABLED        1           // Set to 0 for production

// ============================================================================
// NVS STORAGE
// ============================================================================

// Namespace - must be unique per project (max 15 chars)
#define NVS_NAMESPACE         "{NVS_NAMESPACE}"

// Keys
#define KEY_WIFI_SSID         "wifi_ssid"
#define KEY_WIFI_PASS         "wifi_pass"
#define KEY_CONFIGURED        "configured"
#define KEY_FIRST_BOOT        "first_boot"

// ============================================================================
// HARDWARE - Set by board selection in platformio.ini
// ============================================================================

#ifndef LED_PIN
#define LED_PIN     2
#endif
#ifndef LED_ON
#define LED_ON      HIGH
#endif
#ifndef LED_OFF
#define LED_OFF     LOW
#endif
#ifndef BOOT_BTN
#define BOOT_BTN    0
#endif

// ============================================================================
// WATCHDOG
// ============================================================================

#define WATCHDOG_TIMEOUT_SEC  30          // Auto-reboot if hung

// ============================================================================
// DEBUG
// ============================================================================

#define DEBUG_SERIAL          1
#define SERIAL_BAUD           115200

// Debug macros
#if DEBUG_SERIAL
  #define DEBUG_INIT()        Serial.begin(SERIAL_BAUD)
  #define DEBUG_PRINT(x)      Serial.print(x)
  #define DEBUG_PRINTLN(x)    Serial.println(x)
  #define DEBUG_PRINTF(...)   Serial.printf(__VA_ARGS__)
#else
  #define DEBUG_INIT()
  #define DEBUG_PRINT(x)
  #define DEBUG_PRINTLN(x)
  #define DEBUG_PRINTF(...)
#endif

#endif // CONFIG_H
```

### 3.3 src/utils/debug_log.h

```cpp
/**
 * @file debug_log.h
 * @brief Logging to Serial AND remote syslog
 */

#ifndef DEBUG_LOG_H
#define DEBUG_LOG_H

#include <Arduino.h>

// Initialize logging (call in setup())
void debugLogInit();

// Log message (goes to Serial + syslog if connected)
void debugLog(const char* message);
void debugLogf(const char* format, ...);

// Flush any cached logs (call after WiFi connects)
void debugLogFlush();

#endif // DEBUG_LOG_H
```

### 3.4 src/utils/debug_log.cpp

```cpp
/**
 * @file debug_log.cpp
 * @brief Logging to Serial AND remote syslog with pre-WiFi caching
 */

#include "debug_log.h"
#include "../config.h"
#include <WiFi.h>
#include <WiFiUdp.h>

// Pre-WiFi log cache
#define LOG_CACHE_SIZE 20
#define LOG_LINE_SIZE 128
static char logCache[LOG_CACHE_SIZE][LOG_LINE_SIZE];
static int logCacheIndex = 0;
static bool logCacheFull = false;

// Syslog UDP
static WiFiUDP syslogUdp;
static bool syslogInitialized = false;

void debugLogInit() {
#if DEBUG_SERIAL
    Serial.begin(SERIAL_BAUD);
    delay(100);
#endif
}

static void sendToSyslog(const char* message) {
#if SYSLOG_ENABLED
    if (WiFi.status() != WL_CONNECTED) {
        return;
    }

    if (!syslogInitialized) {
        syslogUdp.begin(0);  // Any local port
        syslogInitialized = true;
    }

    // Simple syslog format: <priority>message
    // Priority 14 = user-level informational
    String syslogMsg = "<14>" + String(PROJECT_NAME) + ": " + String(message);

    syslogUdp.beginPacket(SYSLOG_HOST, SYSLOG_PORT);
    syslogUdp.write((const uint8_t*)syslogMsg.c_str(), syslogMsg.length());
    syslogUdp.endPacket();
#endif
}

static void cacheLog(const char* message) {
    if (logCacheIndex < LOG_CACHE_SIZE) {
        strncpy(logCache[logCacheIndex], message, LOG_LINE_SIZE - 1);
        logCache[logCacheIndex][LOG_LINE_SIZE - 1] = '\0';
        logCacheIndex++;
    } else {
        logCacheFull = true;
    }
}

void debugLog(const char* message) {
#if DEBUG_SERIAL
    Serial.println(message);
#endif

#if SYSLOG_ENABLED
    if (WiFi.status() == WL_CONNECTED) {
        sendToSyslog(message);
    } else {
        cacheLog(message);
    }
#endif
}

void debugLogf(const char* format, ...) {
    char buffer[256];
    va_list args;
    va_start(args, format);
    vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);
    debugLog(buffer);
}

void debugLogFlush() {
#if SYSLOG_ENABLED
    if (WiFi.status() != WL_CONNECTED) {
        return;
    }

    // Send cached logs
    if (logCacheIndex > 0) {
        debugLog("[LOG] Flushing cached logs...");

        if (logCacheFull) {
            debugLog("[LOG] WARNING: Some logs were lost (cache full)");
        }

        for (int i = 0; i < logCacheIndex; i++) {
            sendToSyslog(logCache[i]);
            delay(10);  // Don't flood the network
        }

        debugLogf("[LOG] Flushed %d cached entries", logCacheIndex);
        logCacheIndex = 0;
        logCacheFull = false;
    }
#endif
}
```

### 3.5 src/network/wifi_manager.h

```cpp
/**
 * @file wifi_manager.h
 * @brief WiFi connection + captive portal for configuration
 */

#ifndef WIFI_MANAGER_H
#define WIFI_MANAGER_H

#include <Arduino.h>

// Initialize WiFi manager
void wifiInit();

// Try to connect with stored credentials
// Returns true if connected, false if needs configuration
bool wifiConnect();

// Start captive portal for WiFi configuration
// Blocks until configured or timeout
void wifiStartPortal();

// Check if WiFi is connected
bool wifiIsConnected();

// Get current IP address
String wifiGetIP();

// Save credentials to NVS
void wifiSaveCredentials(const String& ssid, const String& password);

// Clear stored credentials
void wifiClearCredentials();

// Check if credentials are stored
bool wifiHasCredentials();

#endif // WIFI_MANAGER_H
```

### 3.6 src/network/wifi_manager.cpp

```cpp
/**
 * @file wifi_manager.cpp
 * @brief WiFi connection + captive portal
 */

#include "wifi_manager.h"
#include "../config.h"
#include "../utils/debug_log.h"
#include "../storage/nvs_manager.h"
#include <WiFi.h>
#include <WebServer.h>
#include <DNSServer.h>

static WebServer server(80);
static DNSServer dnsServer;
static bool portalRunning = false;
static bool shouldSaveConfig = false;
static String newSSID = "";
static String newPass = "";

// HTML for config portal
static const char CONFIG_HTML[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>WiFi Setup</title>
    <style>
        body { font-family: Arial; margin: 20px; background: #1a1a2e; color: #eee; }
        .container { max-width: 400px; margin: 0 auto; }
        h1 { color: #0f0; }
        input { width: 100%; padding: 12px; margin: 8px 0; box-sizing: border-box; }
        button { width: 100%; padding: 14px; background: #0f0; color: #000; border: none; cursor: pointer; font-size: 16px; }
        button:hover { background: #0d0; }
        .info { background: #333; padding: 10px; margin: 10px 0; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>%PROJECT_NAME%</h1>
        <div class="info">Firmware: %VERSION%</div>
        <form action="/save" method="POST">
            <input type="text" name="ssid" placeholder="WiFi Network Name" required>
            <input type="password" name="pass" placeholder="WiFi Password">
            <button type="submit">Connect</button>
        </form>
    </div>
</body>
</html>
)rawliteral";

static void handleRoot() {
    String html = FPSTR(CONFIG_HTML);
    html.replace("%PROJECT_NAME%", PROJECT_NAME);
    html.replace("%VERSION%", FIRMWARE_VERSION);
    server.send(200, "text/html", html);
}

static void handleSave() {
    newSSID = server.arg("ssid");
    newPass = server.arg("pass");
    shouldSaveConfig = true;

    server.send(200, "text/html",
        "<html><body style='background:#1a1a2e;color:#0f0;font-family:Arial;text-align:center;padding:50px;'>"
        "<h1>Connecting...</h1><p>Device will restart and connect to " + newSSID + "</p>"
        "</body></html>");

    delay(2000);
}

static void handleNotFound() {
    // Redirect all requests to root (captive portal behavior)
    server.sendHeader("Location", "/", true);
    server.send(302, "text/plain", "");
}

void wifiInit() {
    WiFi.mode(WIFI_STA);
    WiFi.setAutoReconnect(true);
}

bool wifiConnect() {
    if (!wifiHasCredentials()) {
        debugLog("[WIFI] No credentials stored");
        return false;
    }

    String ssid = nvsGetString(KEY_WIFI_SSID);
    String pass = nvsGetString(KEY_WIFI_PASS);

    debugLogf("[WIFI] Connecting to: %s", ssid.c_str());

    WiFi.begin(ssid.c_str(), pass.c_str());

    unsigned long startTime = millis();
    int attempts = 0;

    while (WiFi.status() != WL_CONNECTED && millis() - startTime < WIFI_TIMEOUT) {
        delay(500);
        DEBUG_PRINT(".");
        attempts++;
    }
    DEBUG_PRINTLN("");

    if (WiFi.status() == WL_CONNECTED) {
        debugLogf("[WIFI] Connected! IP: %s", WiFi.localIP().toString().c_str());
        debugLogFlush();  // Send cached logs now that we're connected
        return true;
    }

    debugLogf("[WIFI] Failed after %d attempts", attempts);
    return false;
}

void wifiStartPortal() {
    debugLog("[WIFI] Starting config portal...");

    // Create AP
    String apName = String(AP_SSID_PREFIX) + String((uint32_t)ESP.getEfuseMac(), HEX);
    WiFi.mode(WIFI_AP);
    WiFi.softAP(apName.c_str(), AP_PASSWORD, AP_CHANNEL, 0, AP_MAX_CONN);

    debugLogf("[WIFI] AP started: %s", apName.c_str());
    debugLogf("[WIFI] Portal IP: %s", WiFi.softAPIP().toString().c_str());

    // Start DNS server (captive portal redirect)
    dnsServer.start(53, "*", WiFi.softAPIP());

    // Start web server
    server.on("/", handleRoot);
    server.on("/save", HTTP_POST, handleSave);
    server.onNotFound(handleNotFound);
    server.begin();

    portalRunning = true;
    shouldSaveConfig = false;

    // Blink LED to indicate config mode
    unsigned long lastBlink = 0;
    bool ledState = false;

    while (portalRunning) {
        dnsServer.processNextRequest();
        server.handleClient();

        // Blink LED
        if (millis() - lastBlink > 500) {
            ledState = !ledState;
            digitalWrite(LED_PIN, ledState ? LED_ON : LED_OFF);
            lastBlink = millis();
        }

        // Check if user saved config
        if (shouldSaveConfig) {
            wifiSaveCredentials(newSSID, newPass);
            debugLogf("[WIFI] Saved credentials for: %s", newSSID.c_str());
            delay(1000);
            ESP.restart();
        }

        delay(10);
    }
}

bool wifiIsConnected() {
    return WiFi.status() == WL_CONNECTED;
}

String wifiGetIP() {
    return WiFi.localIP().toString();
}

void wifiSaveCredentials(const String& ssid, const String& password) {
    nvsPutString(KEY_WIFI_SSID, ssid);
    nvsPutString(KEY_WIFI_PASS, password);
    nvsPutBool(KEY_CONFIGURED, true);
}

void wifiClearCredentials() {
    nvsRemove(KEY_WIFI_SSID);
    nvsRemove(KEY_WIFI_PASS);
    nvsPutBool(KEY_CONFIGURED, false);
}

bool wifiHasCredentials() {
    return nvsGetBool(KEY_CONFIGURED, false);
}
```

### 3.7 src/network/ota_update.h

```cpp
/**
 * @file ota_update.h
 * @brief OTA update polling from development server
 */

#ifndef OTA_UPDATE_H
#define OTA_UPDATE_H

#include <Arduino.h>

// Get current firmware version
const char* otaGetVersion();

// Check if new firmware is available on server
// Returns true if update available
bool otaCheckForUpdate();

// Download and flash new firmware
// Returns true on success (device will reboot)
bool otaPerformUpdate();

#endif // OTA_UPDATE_H
```

### 3.8 src/network/ota_update.cpp

```cpp
/**
 * @file ota_update.cpp
 * @brief OTA update polling from development server
 */

#include "ota_update.h"
#include "../config.h"
#include "../utils/debug_log.h"
#include <HTTPClient.h>
#include <Update.h>

const char* otaGetVersion() {
    return FIRMWARE_VERSION;
}

bool otaCheckForUpdate() {
#if LOCAL_DEPLOY_ENABLED
    HTTPClient http;
    String url = String("http://") + DEPLOY_SERVER_HOST + ":" + DEPLOY_SERVER_PORT + "/version.txt";

    debugLogf("[OTA] Checking: %s", url.c_str());

    http.begin(url);
    http.setTimeout(5000);
    int code = http.GET();

    if (code != 200) {
        debugLogf("[OTA] Version check failed: %d", code);
        http.end();
        return false;
    }

    String serverVersion = http.getString();
    serverVersion.trim();
    http.end();

    // Parse project:version format
    int colonPos = serverVersion.indexOf(':');
    if (colonPos > 0) {
        String serverProject = serverVersion.substring(0, colonPos);
        String version = serverVersion.substring(colonPos + 1);

        // Verify project name matches (prevent cross-project updates)
        if (serverProject != PROJECT_NAME) {
            debugLogf("[OTA] Project mismatch: server=%s, local=%s",
                      serverProject.c_str(), PROJECT_NAME);
            return false;
        }
        serverVersion = version;
    }

    debugLogf("[OTA] Server: %s, Current: %s", serverVersion.c_str(), FIRMWARE_VERSION);

    return serverVersion != String(FIRMWARE_VERSION);
#else
    return false;
#endif
}

bool otaPerformUpdate() {
#if LOCAL_DEPLOY_ENABLED
    HTTPClient http;
    String url = String("http://") + DEPLOY_SERVER_HOST + ":" + DEPLOY_SERVER_PORT + "/firmware.bin";

    debugLogf("[OTA] Downloading: %s", url.c_str());

    http.begin(url);
    http.setTimeout(30000);
    int code = http.GET();

    if (code != 200) {
        debugLogf("[OTA] Download failed: %d", code);
        http.end();
        return false;
    }

    int size = http.getSize();
    debugLogf("[OTA] Firmware size: %d bytes", size);

    if (size <= 0) {
        debugLog("[OTA] Invalid firmware size");
        http.end();
        return false;
    }

    if (!Update.begin(size)) {
        debugLogf("[OTA] Not enough space: %s", Update.errorString());
        http.end();
        return false;
    }

    WiFiClient* stream = http.getStreamPtr();
    size_t written = Update.writeStream(*stream);
    http.end();

    if (written != size) {
        debugLogf("[OTA] Write error: %d/%d", written, size);
        Update.abort();
        return false;
    }

    if (!Update.end()) {
        debugLogf("[OTA] Update failed: %s", Update.errorString());
        return false;
    }

    debugLog("[OTA] Update SUCCESS! Rebooting...");
    delay(500);
    ESP.restart();
    return true;  // Won't reach here
#else
    return false;
#endif
}
```

### 3.9 src/storage/nvs_manager.h

```cpp
/**
 * @file nvs_manager.h
 * @brief NVS (Non-Volatile Storage) wrapper
 */

#ifndef NVS_MANAGER_H
#define NVS_MANAGER_H

#include <Arduino.h>

// Initialize NVS
void nvsInit();

// Factory reset - clear all stored data
void nvsFactoryReset();

// String operations
void nvsPutString(const char* key, const String& value);
String nvsGetString(const char* key, const String& defaultValue = "");

// Bool operations
void nvsPutBool(const char* key, bool value);
bool nvsGetBool(const char* key, bool defaultValue = false);

// Int operations
void nvsPutInt(const char* key, int value);
int nvsGetInt(const char* key, int defaultValue = 0);

// Remove a key
void nvsRemove(const char* key);

#endif // NVS_MANAGER_H
```

### 3.10 src/storage/nvs_manager.cpp

```cpp
/**
 * @file nvs_manager.cpp
 * @brief NVS (Non-Volatile Storage) wrapper
 */

#include "nvs_manager.h"
#include "../config.h"
#include "../utils/debug_log.h"
#include <Preferences.h>

static Preferences prefs;

void nvsInit() {
    prefs.begin(NVS_NAMESPACE, false);

    // Check for first boot - do factory reset
    if (!prefs.getBool(KEY_FIRST_BOOT, false)) {
        debugLog("[NVS] First boot detected - initializing");
        prefs.clear();  // Clear any remnants from other projects
        prefs.putBool(KEY_FIRST_BOOT, true);
    }
}

void nvsFactoryReset() {
    debugLog("[NVS] Factory reset...");
    prefs.clear();
    prefs.putBool(KEY_FIRST_BOOT, true);  // Mark as initialized
}

void nvsPutString(const char* key, const String& value) {
    prefs.putString(key, value);
}

String nvsGetString(const char* key, const String& defaultValue) {
    return prefs.getString(key, defaultValue);
}

void nvsPutBool(const char* key, bool value) {
    prefs.putBool(key, value);
}

bool nvsGetBool(const char* key, bool defaultValue) {
    return prefs.getBool(key, defaultValue);
}

void nvsPutInt(const char* key, int value) {
    prefs.putInt(key, value);
}

int nvsGetInt(const char* key, int defaultValue) {
    return prefs.getInt(key, defaultValue);
}

void nvsRemove(const char* key) {
    prefs.remove(key);
}
```

### 3.11 src/main.cpp

```cpp
/**
 * @file main.cpp
 * @brief Main application for {PROJECT_NAME}
 * @generated by ESP32 Wireless Dev Starter
 */

#include <Arduino.h>
#include "config.h"
#include "utils/debug_log.h"
#include "network/wifi_manager.h"
#include "network/ota_update.h"
#include "storage/nvs_manager.h"
#include <esp_task_wdt.h>

// Double-reset detection for config mode
#include <esp_system.h>
RTC_NOINIT_ATTR int resetCount;
RTC_NOINIT_ATTR unsigned long lastResetTime;

static unsigned long lastOtaCheck = 0;
static bool configMode = false;

// Check for double-reset (enter config mode)
bool checkDoubleReset() {
    unsigned long now = millis();

    // If reset happened within 3 seconds of last reset
    if (resetCount > 0 && (now < 3000)) {
        resetCount = 0;
        return true;
    }

    // Increment reset count, will be cleared after 3 seconds
    resetCount++;

    // Clear reset count after 3 seconds
    delay(3000);
    resetCount = 0;

    return false;
}

void blinkLED(int times, int onTime = 100, int offTime = 100) {
    for (int i = 0; i < times; i++) {
        digitalWrite(LED_PIN, LED_ON);
        delay(onTime);
        digitalWrite(LED_PIN, LED_OFF);
        delay(offTime);
    }
}

void setup() {
    // Initialize LED
    pinMode(LED_PIN, OUTPUT);
    digitalWrite(LED_PIN, LED_OFF);

    // Initialize debug logging
    debugLogInit();
    delay(100);

    debugLog("================================");
    debugLogf("[BOOT] %s v%s", PROJECT_NAME, FIRMWARE_VERSION);
    debugLog("================================");

    // Initialize NVS
    nvsInit();

    // Initialize watchdog (30 second timeout)
    esp_task_wdt_init(WATCHDOG_TIMEOUT_SEC, true);
    esp_task_wdt_add(NULL);

    // Check for double-reset (manual config mode entry)
    if (checkDoubleReset()) {
        debugLog("[BOOT] Double-reset detected - entering config mode");
        blinkLED(5, 100, 100);  // Fast blink to indicate config mode
        configMode = true;
    }

    // Initialize WiFi
    wifiInit();

    // Try to connect (unless forced config mode)
    if (!configMode && wifiConnect()) {
        debugLog("[BOOT] WiFi connected - normal operation");
        blinkLED(2, 200, 200);  // Two blinks = connected

        // Check for OTA update on boot
        debugLog("[BOOT] Checking for firmware update...");
        if (otaCheckForUpdate()) {
            debugLog("[BOOT] Update available! Downloading...");
            blinkLED(3, 100, 100);
            otaPerformUpdate();
            // If we get here, update failed
            debugLog("[BOOT] Update failed, continuing...");
        } else {
            debugLog("[BOOT] Firmware is up to date");
        }

        // Mark OTA partition as valid (for rollback protection)
        // Only after successful boot + WiFi connect
        #include <esp_ota_ops.h>
        esp_ota_mark_app_valid_cancel_rollback();

    } else {
        // No credentials or connection failed - start config portal
        debugLog("[BOOT] Starting config portal...");
        configMode = true;
        wifiStartPortal();  // Blocks until configured
    }

    debugLog("[BOOT] Setup complete");
}

void loop() {
    // Feed watchdog
    esp_task_wdt_reset();

    // Periodic OTA check (every 60 seconds)
    if (wifiIsConnected() && millis() - lastOtaCheck > (DEPLOY_POLL_INTERVAL * 1000)) {
        lastOtaCheck = millis();

        debugLog("[LOOP] Checking for OTA update...");
        if (otaCheckForUpdate()) {
            debugLog("[LOOP] Update available! Downloading...");
            blinkLED(3, 100, 100);
            otaPerformUpdate();
        }
    }

    // =========================================
    // YOUR APPLICATION CODE GOES HERE
    // =========================================

    // Example: Blink LED every 5 seconds to show we're alive
    static unsigned long lastBlink = 0;
    if (millis() - lastBlink > 5000) {
        lastBlink = millis();
        blinkLED(1, 50, 0);
        debugLogf("[LOOP] Heartbeat - uptime: %lu sec", millis() / 1000);
    }

    delay(10);
}
```

### 3.12 deploy.sh

```bash
#!/bin/bash
#
# deploy.sh - Build and deploy firmware for {PROJECT_NAME}
#
# Usage:
#   ./deploy.sh              - Build and deploy via OTA
#   ./deploy.sh --usb        - First-time USB flash + setup Pi server
#   ./deploy.sh --build-only - Only build, don't deploy
#   ./deploy.sh --skip-build - Deploy without rebuilding
#   ./deploy.sh --help       - Show this help
#

set -e  # Exit on error

# ============================================================================
# CONFIGURATION - Edit these for your setup
# ============================================================================

PROJECT_NAME="{PROJECT_NAME}"
SERVER_HOST="{SERVER_HOST}"
SERVER_PORT="8080"
SERVER_USER="{SERVER_USER}"
SERVER_PROJECT_DIR="/home/{SERVER_USER}/projects/{PROJECT_NAME}"
SERVER_DEPLOY_DIR="$SERVER_PROJECT_DIR/firmware"
SERVER_LOG_DIR="$SERVER_PROJECT_DIR/logs"
SERVER_SCRIPT_DIR="$SERVER_PROJECT_DIR/scripts"
BUILD_ENV="{BUILD_ENV}"
SYSLOG_PORT="514"

# ============================================================================
# Colors
# ============================================================================

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# ============================================================================
# Parse arguments
# ============================================================================

USB_MODE=false
BUILD_ONLY=false
SKIP_BUILD=false

for arg in "$@"; do
    case $arg in
        --usb)
            USB_MODE=true
            ;;
        --build-only)
            BUILD_ONLY=true
            ;;
        --skip-build)
            SKIP_BUILD=true
            ;;
        --help|-h)
            echo "deploy.sh - Build and deploy firmware for $PROJECT_NAME"
            echo ""
            echo "Usage:"
            echo "  ./deploy.sh              - Build and deploy via OTA"
            echo "  ./deploy.sh --usb        - First-time USB flash + setup server"
            echo "  ./deploy.sh --build-only - Only build, don't deploy"
            echo "  ./deploy.sh --skip-build - Deploy without rebuilding"
            echo "  ./deploy.sh --help       - Show this help"
            echo ""
            echo "Configuration:"
            echo "  PROJECT_NAME:  $PROJECT_NAME"
            echo "  SERVER_HOST:   $SERVER_HOST"
            echo "  SERVER_USER:   $SERVER_USER"
            echo "  BUILD_ENV:     $BUILD_ENV"
            exit 0
            ;;
    esac
done

# Get script directory
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
cd "$SCRIPT_DIR"

echo -e "${YELLOW}=== $PROJECT_NAME Deployment ===${NC}"
echo ""

# ============================================================================
# Get version from config.h
# ============================================================================

VERSION=$(grep '#define FIRMWARE_VERSION' src/config.h | sed 's/.*"\(.*\)".*/\1/')
echo -e "Version: ${GREEN}$VERSION${NC}"

# ============================================================================
# Build firmware
# ============================================================================

FIRMWARE_PATH=".pio/build/$BUILD_ENV/firmware.bin"

if [ "$SKIP_BUILD" = true ]; then
    echo -e "${BLUE}Skipping build (--skip-build)${NC}"
    if [ ! -f "$FIRMWARE_PATH" ]; then
        echo -e "${RED}ERROR: No firmware.bin found - run without --skip-build first${NC}"
        exit 1
    fi
else
    echo ""
    echo -e "${YELLOW}Building firmware...${NC}"
    ~/.platformio/penv/bin/pio run -e $BUILD_ENV

    if [ ! -f "$FIRMWARE_PATH" ]; then
        echo -e "${RED}ERROR: Build failed - firmware.bin not found${NC}"
        exit 1
    fi
fi

SIZE=$(ls -lh "$FIRMWARE_PATH" | awk '{print $5}')
echo -e "${GREEN}Build successful${NC} - Size: $SIZE"

if [ "$BUILD_ONLY" = true ]; then
    echo ""
    echo -e "${GREEN}Build complete (--build-only mode)${NC}"
    exit 0
fi

# ============================================================================
# USB Mode - First time setup
# ============================================================================

if [ "$USB_MODE" = true ]; then
    echo ""
    echo -e "${YELLOW}=== USB Flash Mode ===${NC}"
    echo "This will:"
    echo "  1. Flash firmware via USB"
    echo "  2. Set up server infrastructure"
    echo ""

    # Flash via USB
    echo -e "${YELLOW}Flashing via USB...${NC}"
    ~/.platformio/penv/bin/pio run -e $BUILD_ENV --target upload

    echo -e "${GREEN}USB flash complete!${NC}"
    echo ""
    echo "Next steps:"
    echo "  1. Device will boot and create WiFi AP: ${PROJECT_NAME}-XXXX"
    echo "  2. Connect to that AP and configure WiFi"
    echo "  3. After WiFi configured, run ./deploy.sh for OTA updates"
    echo ""

    # Continue to set up server...
fi

# ============================================================================
# Check server connectivity
# ============================================================================

echo ""
echo -e "${YELLOW}Checking server connectivity...${NC}"
if ! ping -c 1 -W 2 $SERVER_HOST > /dev/null 2>&1; then
    echo -e "${RED}ERROR: Cannot reach server at $SERVER_HOST${NC}"
    exit 1
fi
echo -e "${GREEN}Server reachable${NC}"

# ============================================================================
# Check if HTTP server is serving correct project
# ============================================================================

echo ""
echo -e "${YELLOW}Checking HTTP server...${NC}"

# Create directory structure first (needed for directory check)
ssh $SERVER_USER@$SERVER_HOST "mkdir -p $SERVER_DEPLOY_DIR $SERVER_LOG_DIR $SERVER_SCRIPT_DIR"

# Get PID of any HTTP server on our port
# NOTE: Use ps aux instead of pgrep - pgrep can return stale PIDs on some systems
HTTP_PID=$(ssh $SERVER_USER@$SERVER_HOST "ps aux | grep 'python.*http.server.*$SERVER_PORT' | grep -v grep | awk '{print \$2}' | head -1" 2>/dev/null || echo "")

START_NEW_SERVER=false

if [ -z "$HTTP_PID" ]; then
    echo -e "${BLUE}No HTTP server running on port $SERVER_PORT${NC}"
    START_NEW_SERVER=true
else
    # Check what directory it's serving
    echo -e "${BLUE}HTTP server found (PID: $HTTP_PID) - checking directory...${NC}"
    SERVER_CWD=$(ssh $SERVER_USER@$SERVER_HOST "readlink -f /proc/$HTTP_PID/cwd 2>/dev/null" || echo "")

    if [ -z "$SERVER_CWD" ]; then
        echo -e "${YELLOW}Could not determine server directory - restarting${NC}"
        START_NEW_SERVER=true
    elif [ "$SERVER_CWD" != "$SERVER_DEPLOY_DIR" ]; then
        echo -e "${YELLOW}HTTP server serving wrong directory - restarting...${NC}"
        echo "  Current: $SERVER_CWD"
        echo "  Expected: $SERVER_DEPLOY_DIR"
        ssh $SERVER_USER@$SERVER_HOST "pkill -f 'python.*http.server.*$SERVER_PORT'" 2>/dev/null || true
        sleep 1
        START_NEW_SERVER=true
    else
        echo -e "${GREEN}HTTP server running correctly from $SERVER_DEPLOY_DIR (PID: $HTTP_PID)${NC}"
    fi
fi

if [ "$START_NEW_SERVER" = true ]; then
    echo -e "${BLUE}Starting HTTP server from $SERVER_DEPLOY_DIR...${NC}"

    # Start HTTP server
    ssh $SERVER_USER@$SERVER_HOST "cd $SERVER_DEPLOY_DIR && nohup python3 -m http.server $SERVER_PORT > $SERVER_LOG_DIR/http_server.log 2>&1 &"
    sleep 1

    # Verify
    HTTP_PID=$(ssh $SERVER_USER@$SERVER_HOST "ps aux | grep 'python.*http.server.*$SERVER_PORT' | grep -v grep | awk '{print \$2}' | head -1" 2>/dev/null || echo "")
    if [ -z "$HTTP_PID" ]; then
        echo -e "${RED}ERROR: Failed to start HTTP server${NC}"
        exit 1
    fi
    echo -e "${GREEN}HTTP server started (PID: $HTTP_PID)${NC}"
fi

# ============================================================================
# Deploy firmware
# ============================================================================

echo ""
echo -e "${YELLOW}Deploying firmware...${NC}"

# Create directories
ssh $SERVER_USER@$SERVER_HOST "mkdir -p $SERVER_DEPLOY_DIR $SERVER_LOG_DIR $SERVER_SCRIPT_DIR"

# Copy firmware
scp -q "$FIRMWARE_PATH" "$SERVER_USER@$SERVER_HOST:$SERVER_DEPLOY_DIR/firmware.bin"

# Create version.txt with project:version format
ssh $SERVER_USER@$SERVER_HOST "echo '$PROJECT_NAME:$VERSION' > $SERVER_DEPLOY_DIR/version.txt"

echo -e "${GREEN}Deployed: $PROJECT_NAME:$VERSION${NC}"

# ============================================================================
# Verify deployment
# ============================================================================

echo ""
echo -e "${YELLOW}Verifying deployment...${NC}"

REMOTE_VERSION=$(curl -s --connect-timeout 2 "http://$SERVER_HOST:$SERVER_PORT/version.txt" 2>/dev/null || echo "FAILED")
if [ "$REMOTE_VERSION" = "$PROJECT_NAME:$VERSION" ]; then
    echo -e "${GREEN}version.txt OK: $REMOTE_VERSION${NC}"
else
    echo -e "${RED}WARNING: version.txt mismatch${NC}"
    echo "  Expected: $PROJECT_NAME:$VERSION"
    echo "  Got: $REMOTE_VERSION"
fi

FIRMWARE_STATUS=$(curl -sI --connect-timeout 2 "http://$SERVER_HOST:$SERVER_PORT/firmware.bin" 2>/dev/null | head -1 || echo "FAILED")
if echo "$FIRMWARE_STATUS" | grep -q "200"; then
    REMOTE_SIZE=$(curl -sI "http://$SERVER_HOST:$SERVER_PORT/firmware.bin" 2>/dev/null | grep -i content-length | awk '{print $2}' | tr -d '\r')
    echo -e "${GREEN}firmware.bin OK (${REMOTE_SIZE} bytes)${NC}"
else
    echo -e "${RED}WARNING: firmware.bin not accessible${NC}"
fi

# ============================================================================
# Ensure syslog listener is running
# ============================================================================

echo ""
echo -e "${YELLOW}Checking syslog listener...${NC}"

SYSLOG_RUNNING=$(ssh $SERVER_USER@$SERVER_HOST "ss -uln | grep ':$SYSLOG_PORT '" 2>/dev/null || echo "")

if [ -z "$SYSLOG_RUNNING" ]; then
    echo -e "${BLUE}Starting syslog listener...${NC}"

    # Create self-rotating Python syslog listener
    ssh $SERVER_USER@$SERVER_HOST "cat > $SERVER_SCRIPT_DIR/syslog_listener.py << 'EOFPY'
#!/usr/bin/env python3
\"\"\"Self-rotating syslog listener for ESP32 debug logs.\"\"\"
import socket
import os
from datetime import datetime

LOG_DIR = \"$SERVER_LOG_DIR\"
LOG_FILE = os.path.join(LOG_DIR, \"esp32.log\")
MAX_SIZE_MB = 5
MAX_SIZE_BYTES = MAX_SIZE_MB * 1024 * 1024

def rotate_if_needed():
    if not os.path.exists(LOG_FILE):
        return
    if os.path.getsize(LOG_FILE) < MAX_SIZE_BYTES:
        return
    if os.path.exists(LOG_FILE + \".2\"):
        os.remove(LOG_FILE + \".2\")
    if os.path.exists(LOG_FILE + \".1\"):
        os.rename(LOG_FILE + \".1\", LOG_FILE + \".2\")
    os.rename(LOG_FILE, LOG_FILE + \".1\")
    print(f\"[ROTATE] Log rotated at {datetime.now()}\", flush=True)

def main():
    os.makedirs(LOG_DIR, exist_ok=True)
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind((\"0.0.0.0\", 514))
    print(f\"Listening for syslog on UDP port 514...\", flush=True)

    with open(LOG_FILE, \"a\") as f:
        line_count = 0
        while True:
            data, addr = sock.recvfrom(1024)
            msg = data.decode(\"utf-8\", errors=\"ignore\").strip()
            timestamp = datetime.now().strftime(\"%Y-%m-%d %H:%M:%S\")
            line = f\"[{timestamp}] [{addr[0]}] {msg}\"
            print(line, flush=True)
            f.write(line + \"\\n\")
            f.flush()
            line_count += 1
            if line_count >= 100:
                line_count = 0
                rotate_if_needed()

if __name__ == \"__main__\":
    main()
EOFPY"

    # Start listener (requires sudo for port 514)
    ssh $SERVER_USER@$SERVER_HOST "nohup sudo python3 $SERVER_SCRIPT_DIR/syslog_listener.py > $SERVER_LOG_DIR/syslog_listener.out 2>&1 &"
    sleep 1

    SYSLOG_RUNNING=$(ssh $SERVER_USER@$SERVER_HOST "ss -uln | grep ':$SYSLOG_PORT '" 2>/dev/null || echo "")
    if [ -n "$SYSLOG_RUNNING" ]; then
        echo -e "${GREEN}Syslog listener started${NC}"
    else
        echo -e "${YELLOW}WARNING: Could not start syslog listener (may need sudo password)${NC}"
        echo "  Manual start: ssh $SERVER_USER@$SERVER_HOST 'sudo python3 $SERVER_SCRIPT_DIR/syslog_listener.py'"
    fi
else
    echo -e "${GREEN}Syslog listener already running${NC}"
fi

# ============================================================================
# Summary
# ============================================================================

echo ""
echo -e "${GREEN}=== Deployment Complete ===${NC}"
echo ""
echo "Project:  $PROJECT_NAME"
echo "Version:  $VERSION"
echo "Server:   http://$SERVER_HOST:$SERVER_PORT/"
echo ""
echo "ESP32 checks for updates every ${DEPLOY_POLL_INTERVAL:-60} seconds."
echo ""
echo "Monitor logs:"
echo "  ssh $SERVER_USER@$SERVER_HOST 'tail -f $SERVER_LOG_DIR/esp32.log'"
```

### 3.13 CLAUDE.md (Generated for the project)

```markdown
# {PROJECT_NAME} - Claude Code Instructions

**Auto-loaded context for Claude Code sessions**

---

## Project Overview

ESP32 wireless development project with OTA updates and remote logging.

**Current Version**: Check `src/config.h` for `FIRMWARE_VERSION`
**Board**: {BUILD_ENV}
**Server**: {SERVER_USER}@{SERVER_HOST}

---

## CRITICAL: MCU vs Source Code

**The code on disk is NOT the code running on the ESP32!**

1. Editing files only changes source code on your computer
2. Only `./deploy.sh --usb` or `./deploy.sh` (OTA) updates the MCU
3. Check logs to verify MCU version: look for `[BOOT] {PROJECT_NAME} vX.X.X`

### Build Commands

**IMPORTANT**: Always use full path to PlatformIO:

```bash
# CORRECT
~/.platformio/penv/bin/pio run -e {BUILD_ENV}

# WRONG (may fail)
pio run -e {BUILD_ENV}
```

### Deploy Commands

```bash
# First time (USB flash + server setup)
./deploy.sh --usb

# After that (OTA - wireless)
./deploy.sh

# Other options
./deploy.sh --build-only   # Just build
./deploy.sh --skip-build   # Redeploy existing binary
./deploy.sh --help         # Show help
```

---

## Development Infrastructure

**Server**: {SERVER_HOST} ({SERVER_USER}@{SERVER_HOST})

### Directory Structure on Server

```
/home/{SERVER_USER}/projects/{PROJECT_NAME}/
├── logs/
│   ├── esp32.log              # ESP32 debug logs (self-rotating at 5MB)
│   ├── esp32.log.1            # Previous log
│   └── esp32.log.2            # Older log
├── firmware/
│   ├── firmware.bin           # Current firmware for OTA
│   └── version.txt            # {PROJECT_NAME}:X.X.X-dev
└── scripts/
    └── syslog_listener.py     # Log receiver
```

### Commands You'll Use

```bash
# Watch logs in real-time
ssh {SERVER_USER}@{SERVER_HOST} 'tail -f /home/{SERVER_USER}/projects/{PROJECT_NAME}/logs/esp32.log'

# Get last 100 log lines
ssh {SERVER_USER}@{SERVER_HOST} 'tail -100 /home/{SERVER_USER}/projects/{PROJECT_NAME}/logs/esp32.log'

# Check deployed version
curl http://{SERVER_HOST}:8080/version.txt

# Check if services are running
ssh {SERVER_USER}@{SERVER_HOST} 'pgrep -a python.*http.server'
ssh {SERVER_USER}@{SERVER_HOST} 'ss -uln | grep :514'
```

---

## Troubleshooting

### ESP32 not updating via OTA

1. Check WiFi connected: Look for `[WIFI] Connected!` in logs
2. Check server accessible: `curl http://{SERVER_HOST}:8080/version.txt`
3. Check project name matches: version.txt should show `{PROJECT_NAME}:X.X.X`

### Multiple projects on same server

Each project uses its own:
- Port 8080 (shared) but different directories
- Directory: `/home/{SERVER_USER}/projects/{PROJECT_NAME}/`

If ESP32 downloads wrong firmware:
```bash
# Check what HTTP servers are running
ssh {SERVER_USER}@{SERVER_HOST} 'pgrep -a python.*http.server'

# Kill all and restart correct one
ssh {SERVER_USER}@{SERVER_HOST} 'pkill -f "python.*http.server"'
./deploy.sh --skip-build  # Restarts correct server
```

### Enter config mode (reconfigure WiFi)

**Double-reset**: Press RESET button twice within 3 seconds.
Device will blink fast and start config AP.

### Device keeps rebooting (bad firmware)

ESP32 has automatic rollback. After 3 failed boots, it reverts to previous firmware.

---

## Logging Best Practices

**Initial setup**: All `Serial.print` goes to remote syslog automatically.

**As project grows**: Replace generic prints with specific `debugLog()` calls:

```cpp
// Instead of flooding with Serial.print
Serial.println("Temperature: 25.5");

// Use structured logging
debugLogf("[SENSOR] Temperature: %.1f C", temp);
```

This keeps logs meaningful and reduces network traffic.

---

## Adding Project-Specific Functionality

Your application code goes in `src/main.cpp` in the `loop()` function:

```cpp
void loop() {
    // Feed watchdog
    esp_task_wdt_reset();

    // OTA check happens automatically

    // =========================================
    // YOUR CODE HERE
    // =========================================

    delay(10);
}
```

---

## Version Bumping

Before deploying changes, bump the version in `src/config.h`:

```cpp
#define FIRMWARE_VERSION  "1.0.1-dev"  // Was 1.0.0-dev
```

Then run `./deploy.sh`. ESP32 will auto-update within 60 seconds.

---
{XIAO_C6_SSL_NOTE}
```

### 3.14 README.md (User documentation)

```markdown
# {PROJECT_NAME}

ESP32-based wireless project.

## First Time Setup

1. **Flash firmware via USB**:
   ```bash
   ./deploy.sh --usb
   ```

2. **Configure WiFi**:
   - Device creates WiFi network: `{PROJECT_NAME}-XXXX`
   - Connect to it with your phone/laptop
   - Open http://192.168.4.1 in browser
   - Enter your WiFi credentials
   - Device restarts and connects

3. **Verify connection**:
   ```bash
   # Watch device logs
   ssh {SERVER_USER}@{SERVER_HOST} 'tail -f /home/{SERVER_USER}/projects/{PROJECT_NAME}/logs/esp32.log'
   ```

## Updating Firmware

After initial setup, all updates are wireless:

```bash
./deploy.sh
```

Device checks for updates every 60 seconds and auto-updates.

## Reconfiguring WiFi

If you need to change WiFi credentials:
- **Double-reset**: Press RESET button twice within 3 seconds
- Device will blink fast and create config WiFi network
- Connect and reconfigure

## Troubleshooting

### Device not connecting to WiFi
- Check WiFi credentials are correct
- Make sure 2.4GHz network (not 5GHz)
- Double-reset to reconfigure

### OTA not working
- Check server is running: `curl http://{SERVER_HOST}:8080/version.txt`
- Check device can reach server (same network)
- Check logs for errors

### Device keeps rebooting
- Automatic rollback will restore previous firmware after 3 failed boots
- Use USB flash to recover if needed

## Technical Details

- **Board**: {BUILD_ENV}
- **OTA Server**: http://{SERVER_HOST}:8080
- **Log Server**: {SERVER_HOST}:514 (UDP syslog)
- **Config Portal**: http://192.168.4.1 (when in AP mode)
```

---

## Phase 4: XIAO ESP32-C6 Special Handling

**If BOARD_TYPE == xiao-esp32c6, add this note to CLAUDE.md:**

```markdown
## XIAO ESP32-C6 SSL Notes

This board requires special handling for HTTPS/SSL:

### Platform
Standard espressif32 has NO SSL support on C6. Using pioarduino fork:
```ini
platform = https://github.com/pioarduino/platform-espressif32/releases/download/51.03.07/platform-espressif32.zip
```

### WiFiClientSecure vs NetworkClientSecure
In Arduino 3.x for ESP32-C6, use `NetworkClientSecure` instead of `WiFiClientSecure`:

```cpp
#if USE_NETWORK_CLIENT_SECURE
  #include <NetworkClientSecure.h>
  NetworkClientSecure client;
#else
  #include <WiFiClientSecure.h>
  WiFiClientSecure client;
#endif
```

### Deep Sleep Wake Limitation
ESP32-C6 can ONLY wake from deep sleep using GPIO 0-7 (LP GPIOs).
GPIO 8+ (including boot button GPIO 9) CANNOT wake from deep sleep.
```

---

## Phase 5: Execution Checklist

**Claude must complete these steps:**

- [ ] Ask all questions from Phase 1
- [ ] Sanitize project name
- [ ] Verify SSH key access to server
- [ ] Create all files from Phase 3 (with variables replaced)
- [ ] If XIAO C6, add SSL notes to CLAUDE.md
- [ ] Make deploy.sh executable: `chmod +x deploy.sh`
- [ ] Instruct user to run `./deploy.sh --usb` for first flash
- [ ] Verify first flash succeeded
- [ ] Instruct user to configure WiFi via captive portal
- [ ] Verify OTA update works: bump version, run `./deploy.sh`, check logs

---

## Variable Replacement Reference

When creating files, replace these placeholders:

| Placeholder | Source |
|-------------|--------|
| `{PROJECT_NAME}` | User input, sanitized |
| `{SERVER_HOST}` | User input (IP address) |
| `{SERVER_USER}` | User input (username) |
| `{BUILD_ENV}` | Based on board selection |
| `{NVS_NAMESPACE}` | First 15 chars of PROJECT_NAME |
| `{XIAO_C6_SSL_NOTE}` | SSL notes if XIAO C6, empty otherwise |

---

## Success Criteria

Setup is complete when:

1. `./deploy.sh --usb` flashes successfully
2. Device creates WiFi AP on first boot
3. WiFi configuration via portal works
4. Device connects to WiFi and sends logs to server
5. `./deploy.sh` deploys new version
6. Device auto-updates via OTA within 60 seconds
7. Logs appear on server: `tail -f .../logs/esp32.log`

---

**End of ESP32 Wireless Dev Starter**
