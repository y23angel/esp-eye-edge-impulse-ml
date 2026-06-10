# 07. Firebase Realtime Database Integration

**Files**: `main/main.cpp`, `edge-impulse/inference/ei_run_camera_impulse.cpp`

## Overview

After each FOMO inference, the ESP32 sends detection results to Firebase Realtime Database  
via HTTPS POST using `esp_http_client`. The database stores a log of all detections.

## Architecture

```
ESP32 (FOMO inference)
  └── firebase_send_detection()
        └── HTTP POST → https://<project>-default-rtdb.firebaseio.com/detections.json
              └── Firebase Realtime Database
                    └── Mobile App (reads in real-time)
```

## Firebase Setup

1. Create project at console.firebase.google.com
2. Enable Realtime Database (locked mode)
3. Set security rules:
```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```
> **Note**: `.read: true` is for development only. Add Firebase Auth before shipping.

## HTTP POST Function

```cpp
// main/main.cpp
void firebase_send_detection(bool detected, float confidence, const char *label)
{
    char url[200];
    char post_data[200];

    snprintf(url, sizeof(url), "%s/detections.json", FIREBASE_DATABASE_URL);
    snprintf(post_data, sizeof(post_data),
             "{\"detected\":%s,\"label\":\"%s\",\"confidence\":%.3f}",
             detected ? "true" : "false", label ? label : "none", confidence);

    esp_http_client_config_t config = {};
    config.url = url;

    esp_http_client_handle_t client = esp_http_client_init(&config);
    esp_http_client_set_method(client, HTTP_METHOD_POST);
    esp_http_client_set_header(client, "Content-Type", "application/json");
    esp_http_client_set_post_field(client, post_data, strlen(post_data));

    esp_err_t err = esp_http_client_perform(client);
    esp_http_client_cleanup(client);
}
```

## Hook into Inference Results

```cpp
// edge-impulse/inference/ei_run_camera_impulse.cpp
extern void firebase_send_detection(bool detected, float confidence, const char *label);

// After ei_print_results():
#if EI_CLASSIFIER_OBJECT_DETECTION == 1
    bool parcel_found = false;
    float best_conf = 0.0f;
    const char *best_label = "none";
    for (uint32_t i = 0; i < result.bounding_boxes_count; i++) {
        if (result.bounding_boxes[i].value > 0) {
            parcel_found = true;
            if (result.bounding_boxes[i].value > best_conf) {
                best_conf = result.bounding_boxes[i].value;
                best_label = result.bounding_boxes[i].label;
            }
        }
    }
    firebase_send_detection(parcel_found, best_conf, best_label);
#endif
```

## Data Stored in Firebase

Each POST creates a new auto-keyed entry:
```
detections/
  -OueNPK0cTuMEol38fCm:
    confidence: 0
    detected: false
    label: "none"
  -OueNQ_sdw7Zr5Z10m38:
    confidence: 0.875
    detected: true
    label: "box"
```

## SSL Configuration

Firebase requires HTTPS. For development, SSL verification is disabled in `sdkconfig`:
```
CONFIG_ESP_TLS_INSECURE=y
CONFIG_ESP_TLS_SKIP_SERVER_CERT_VERIFY=y
```
> **Note**: Embed proper root CA certificate before production deployment.

## Credentials File (never commit)

```cpp
// main/firebase_config.h  ← listed in .gitignore
#define FIREBASE_DATABASE_URL "https://<project>-default-rtdb.firebaseio.com"
```

## Serial Output

```
Firebase: sent {"detected":false,"label":"none","confidence":0.000}
Firebase: sent {"detected":true,"label":"box","confidence":0.875}
```
