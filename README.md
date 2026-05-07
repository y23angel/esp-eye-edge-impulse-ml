# ESP-EYE + Edge Impulse: Embedded Object & Audio Recognition (Ongoing)

This project implements **on-device machine learning** using the **ESP-EYE (ESP32)** development board and **Edge Impulse**.  
The goal is to capture image/audio data, train ML models, and deploy real-time inference directly on the embedded device.

---

## Project Overview

Using the ESP-EYE camera and microphone, this project builds a complete workflow:

1. Data acquisition from the device  
2. Feature generation (audio/image)  
3. Model training and evaluation  
4. Building firmware with the deployed ML model  
5. Running inference on the ESP-EYE  
6. Monitoring predictions over UART serial output

Supported use cases include:

- Object/face recognition  
- Simple audio classification (keyword spotting, environmental sounds)  
- Experimenting with Edge Impulse TFLite models on microcontrollers

---

## Hardware & Software

### Hardware
- ESP-EYE (ESP32)  
- Micro-USB cable  
- Windows PC  
- Optional: USB-UART adapter

### Software / Tools
- Edge Impulse Studio  
- ESP-IDF (5.x recommended)  
- Python 3  
- esptool / idf.py  
- Git  
- Serial console (PuTTY, screen, minicom, etc.)

---

## Setup Steps

### 1. Clone ESP-EYE Edge Impulse Firmware
```bash
git clone https://github.com/edgeimpulse/firmware-espressif-esp32
cd firmware-espressif-esp32
idf.py set-target esp32
idf.py build
```

---

## Code Walkthrough (Study Notes)

A deep-dive into the firmware codebase — tracing how a single `AT+RUNIMPULSE` command triggers the full ML inference pipeline.

### Model deployed in this study

| Item | Detail |
|------|--------|
| Project | `esp-eye-cup-detector` |
| Model | FOMO (Faster Objects, More Objects) |
| Input | 96×96 RGB image |
| Output | Bounding boxes on a 12×12 grid |
| Class | `glasses` |
| Threshold | 0.5 |
| Quantization | INT8 |

---

### 1. Boot & Initialization (`main/main.cpp`)

```
app_main()
 ├── setup_led()               GPIO 21 (RED), GPIO 22 (WHITE)
 ├── ei_inertial_init()        LIS3DHTR accelerometer via I2C
 ├── ei_analog_sensor_init()   ADC sensor on GPIO34
 ├── ei_at_init()              UART AT command server (115200 baud)
 └── while(1)
       data = ei_get_serial_byte()   // read UART
       at->handle(data)              // parse & dispatch
```

> **Pin conflict**: GPIO 21/22 are shared between the LEDs and the I2C bus (accelerometer SDA/SCL).  
> Running `ei_inertial_init()` reconfigures these pins for I2C, which disables LED control.

---

### 2. UART → AT Command Flow

```
User types: "AT+RUNIMPULSE\r"
                │
                ▼
ei_get_serial_byte()           // wraps getchar(); maps \n → \r
                │              // returns 0xFF (EOF) when no data available
                ▼
ATServer::handle(char c)
  ├── normal chars  → buffer.add(c)         // accumulate the command string
  ├── 0x1b (ESC)   → control_sequence[]    // arrow keys, HOME, END, etc.
  └── '\r'          → execute("AT+RUNIMPULSE")
                │
                ▼
ATServer::execute(input)
  ├── parser.parse()            // extract command name
  └── registered_commands[]    // find & call matching handler
```

#### AT Commands for inference

| Command | `continuous` | `debug` | Behavior |
|---------|-------------|---------|----------|
| `AT+RUNIMPULSE` | false | false | Single run, 2 s delay between frames |
| `AT+RUNIMPULSECONT` | true | false | Continuous, no delay |
| `AT+RUNIMPULSEDEBUG=n` | false | true | Single run + base64 JPEG over UART |
| `AT+RUNIMPULSEDEBUG=y` | false | true | Same, but UART bumped to 1 Mbps during transfer |

All three call `ei_start_impulse(continuous, debug, use_max_uart_speed)`.

---

### 3. Inference Loop (`edge-impulse/inference/ei_run_camera_impulse.cpp`)

#### `ei_start_impulse()` — setup & loop

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

#### `ei_run_impulse()` — one frame

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
  // callback packs RGB888 → int32: (R<<16)|(G<<8)|B
        │
        ▼
run_classifier(&signal, &result)       // → process_impulse()
        │
        ▼
ei_print_results()                     // print bounding boxes over UART
```

**Observed timings (from serial log):**

| Stage | Time |
|-------|------|
| DSP (image pre-processing) | 8 ms |
| TFLite inference | 647 ms |
| Post-processing | ~0.2 ms |
| **Total (~1.5 FPS)** | **~655 ms** |

---

### 4. `process_impulse()` — compile-time branch selection

`run_classifier()` delegates to `process_impulse()`, which selects a code path at **compile time** via `#if`:

```
process_impulse()
 │
 ├─ #if VLM_CONNECTOR           → run_vlm_inference()          (vision-language models)
 │
 ├─ #if QUANTIZATION_ENABLED    → can_run_classifier_image_quantized()?
 │     + TFLite engine               YES → run_classifier_image_quantized()  ← THIS PROJECT
 │     + image DSP block                    stays in INT8, no float buffer needed
 │
 └─ general path                → DSP loop → run_inference() → run_postprocessing()
                                   (audio: MFCC/MFE/spectrogram; non-quantized models)
```

This project takes the **quantized image shortcut** because:
- Engine: TFLite (EON-compiled)
- Quantization: INT8 enabled
- DSP block: `extract_image_features`
- No anomaly detection

---

### 5. Post-processing — `process_fomo_i8()`

The TFLite model outputs a **12×12 grid** where each cell holds confidence scores per class.

```
Raw output: 12×12×(1 class + 1 background) = 288 INT8 values
```

#### Steps:

```
1. Dequantize
   int8_t v  →  float vf = (v - zero_point) * scale
   zero_point = -128,  scale = 0.00390625

2. Threshold filter
   for each cell (x, y) in 12×12 grid:
     if vf >= 0.5  →  ei_handle_cube(x, y, vf, "glasses")

3. Merge adjacent detections (NMS-like)
   overlapping cube found?  → expand existing bounding box
   no overlap?              → create new cube

4. Scale grid coords → pixel coords
   out_width_factor = 96 / 12 = 8
   bounding_box.x   = cube.x * 8

5. Write to result
   result->bounding_boxes       = [{ label, x, y, w, h, confidence }, ...]
   result->bounding_boxes_count = N
```

#### Output when glasses detected:
```
#Object detection bounding boxes:
  glasses (0.87): [ x: 24, y: 16, width: 8, height: 8 ]
```

#### Output when nothing detected (our log — no glasses in frame):
```
#Object detection bounding boxes:
(empty)
```

---

### Key Takeaways

1. **GPIO pin sharing**: On ESP-EYE, GPIO 21/22 serve as both LED pins and I2C SDA/SCL. The LED and accelerometer cannot be used simultaneously.

2. **AT command buffering**: The UART loop reads one byte at a time. The AT server accumulates characters in `buffer` and only dispatches on `\r`.

3. **Compile-time branching**: Inferencing engine, quantization, and DSP block type are all resolved at compile time via `#if`. Changing the model type on Edge Impulse generates different macro values → different C++ code paths.

4. **FOMO output is a grid, not a list**: The model does not directly output bounding boxes. Post-processing converts a 12×12 confidence grid into merged bounding boxes via a simple NMS-like algorithm.

5. **INT8 quantization saves RAM**: The quantized image shortcut avoids allocating a separate float feature matrix — important on an MCU with ~300 KB of usable RAM.

---

### Serial Log Excerpt (actual run, Apr 11 2026)

```
Hello from Edge Impulse Device SDK.
Compiled on Apr 11 2026 19:17:04
WARNING: Failed to connect to Inertial sensor!   ← GPIO 21/22 conflict
ABC sensor initialization success
> AT+RUNIMPULSE
Inferencing settings:
    Image resolution: 96x96
    Frame size: 9216
    No. of classes: 1
Starting inferencing in 2 seconds...
Taking photo...
Timing: DSP 8 ms, inference 647 ms, anomaly 0 ms, postprocessing 270 us
#Object detection bounding boxes:
                                                  ← no glasses in frame
```
