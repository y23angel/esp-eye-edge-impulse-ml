# 06. WiFi Connectivity

**File**: `main/main.cpp`

## Overview

ESP32 connects to a home WiFi network using `esp_wifi.h` (ESP-IDF station mode).  
Connection is event-driven via FreeRTOS EventGroups — `app_main` blocks until IP is assigned.

## Execution Flow

```
wifi_init()
 ├── xEventGroupCreate()              create WIFI_CONNECTED_BIT flag
 ├── nvs_flash_init()                 required for WiFi config storage
 ├── esp_netif_init()                 initialize network interface
 ├── esp_event_loop_create_default()  create system event loop
 ├── esp_netif_create_default_wifi_sta()
 ├── esp_wifi_init()
 ├── esp_event_handler_register()     register wifi_event_handler
 ├── esp_wifi_set_mode(WIFI_MODE_STA)
 ├── esp_wifi_set_config()            SSID + password from wifi_credentials.h
 ├── esp_wifi_start()                 triggers WIFI_EVENT_STA_START
 └── xEventGroupWaitBits()            blocks until WIFI_CONNECTED_BIT is set
```

## Event Handler

```cpp
static void wifi_event_handler(void *arg, esp_event_base_t event_base,
                               int32_t event_id, void *event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();                          // start connection attempt
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        esp_wifi_connect();                          // auto-reconnect on drop
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);  // unblock main
    }
}
```

## Credentials File (never commit)

```cpp
// main/wifi_credentials.h  ← listed in .gitignore
#define WIFI_SSID     "your_ssid"
#define WIFI_PASSWORD "your_password"
```

## CMakeLists.txt Requirements

```cmake
# esp_wifi, nvs_flash are auto-included by ESP-IDF
# Only explicitly needed components:
target_link_libraries(${COMPONENT_LIB} PRIVATE idf::esp_http_client idf::esp-tls)
```

## Serial Output on Success

```
WiFi connected, IP: 192.168.1.73
```

## Key Concepts

| Concept | Detail |
|---------|--------|
| Station mode | ESP32 connects to existing router (not AP mode) |
| EventGroup | FreeRTOS primitive for synchronization between tasks/events |
| `portMAX_DELAY` | Wait forever until bit is set |
| nvs_flash | Non-Volatile Storage — required to persist WiFi config |
