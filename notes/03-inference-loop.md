# 03. Inference Loop

**File**: `edge-impulse/inference/ei_run_camera_impulse.cpp`

## ei_start_impulse() — Setup & Loop

```
ei_start_impulse(continuous, debug, use_max_uart_speed)
 ├── Set target resolution: 96×96
 ├── camera->search_resolution()    // find closest resolution the sensor supports
 ├── camera->init()
 ├── state = continuous ? INFERENCE_DATA_READY   // start immediately
 │                      : INFERENCE_WAITING      // wait 2 s first
 └── while (!ei_user_invoke_stop())
       ei_run_impulse()
       ei_sleep(1)
```

## ei_run_impulse() — One Frame

```
State machine:
  INFERENCE_WAITING    → timer elapsed? → INFERENCE_DATA_READY
  INFERENCE_DATA_READY → proceed
        │
        ▼
camera->ei_camera_capture_jpeg()       // capture JPEG from sensor
        │
        ▼
camera->ei_camera_jpeg_to_rgb888()     // decompress → [R,G,B, R,G,B, ...]
                                       // 96×96×3 = 27,648 bytes
        │  (if resize needed)
crop_and_interpolate_rgb888()          // scale down to 96×96
        │
        ▼
signal_t signal { get_data = ei_camera_get_data }
  // callback: packs RGB888 → int32: (R<<16)|(G<<8)|B
        │
        ▼
run_classifier(&signal, &result)
        │
        ▼
ei_print_results()                     // print bounding boxes over UART
```

## Observed Timings (from serial log)

| Stage | Time |
|-------|------|
| DSP (image pre-processing) | 8 ms |
| TFLite inference | 647 ms |
| Post-processing | ~0.2 ms |
| **Total (~1.5 FPS)** | **~655 ms** |

## debug_mode Output

When `AT+RUNIMPULSEDEBUG` is used, each frame also outputs the captured image as a base64-encoded JPEG over UART:

```
Taking photo...
Time resizing: 3
Begin output
Framebuffer: /9j/4AAQSkZJRgABAQAA...  (base64, ~4000-8000 chars)
Timing: DSP 8 ms, inference 647 ms ...
----------------------------------
End output
```
