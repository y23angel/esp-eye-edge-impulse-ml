# 05. FOMO Post-processing

**File**: `edge-impulse-sdk/classifier/postprocessing/ei_postprocessing_common.h`

## Model Output

The TFLite model outputs a **12×12 grid**, not bounding boxes directly.  
Each cell holds confidence scores per class:

```
12 × 12 × (1 background + 1 class) = 288 INT8 values
```

Config (from `model-parameters/model_variables.h`):
```cpp
.threshold  = 0.5
.out_width  = 12
.out_height = 12
.zero_point = -128
.scale      = 0.00390625
```

## process_fomo_i8() Steps

```
1. Dequantize
   int8_t v  →  float vf = (v - zero_point) * scale
   range: INT8 -128~127  →  float 0.0~1.0

2. Threshold filter
   for each cell (x, y) in 12×12:
     if vf >= 0.5  →  ei_handle_cube(x, y, vf, "glasses")

3. Merge adjacent detections (NMS-like)
   ei_handle_cube():
     overlapping cube?  → expand existing bounding box
     no overlap?        → create new cube

4. Scale grid → pixel coordinates
   out_width_factor = 96 / 12 = 8
   bounding_box.x   = cube.x * 8

5. Write to result
   result->bounding_boxes       = [{ label, x, y, w, h, confidence }, ...]
   result->bounding_boxes_count = N
```

## Output Examples

**Glasses detected:**
```
#Object detection bounding boxes:
  glasses (0.87): [ x: 24, y: 16, width: 8, height: 8 ]
```

**Nothing detected** (our serial log — no glasses in frame):
```
#Object detection bounding boxes:
(empty — no cell exceeded 0.5 threshold)
```

## Serial Log (actual run, Apr 11 2026)

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
```
