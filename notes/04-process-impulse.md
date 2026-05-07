# 04. process_impulse — Branch Selection

**File**: `edge-impulse-sdk/classifier/ei_run_classifier.h`

## Call Chain

```
run_classifier()
    └── process_impulse()
```

## Compile-Time Branch Selection (#if)

`process_impulse()` selects a completely different code path at **compile time** based on macros:

```
process_impulse()
 │
 ├─ #if VLM_CONNECTOR           → run_vlm_inference()
 │                                (vision-language models like GPT-4V)
 │
 ├─ #if QUANTIZATION_ENABLED    → can_run_classifier_image_quantized()?
 │     + TFLite engine               YES → run_classifier_image_quantized()  ← THIS PROJECT
 │     + image DSP block                    stays in INT8, no float buffer
 │
 └─ general path                → DSP block loop → run_inference() → run_postprocessing()
                                   (audio: MFCC/MFE/spectrogram, non-quantized models)
```

## Why This Project Takes the Quantized Image Path

`can_run_classifier_image_quantized()` checks 4 conditions:

| Condition | This project |
|-----------|-------------|
| TFLite engine | ✅ EON-compiled TFLite |
| INT8 quantization enabled | ✅ |
| DSP block is image (`extract_image_features`) | ✅ |
| No anomaly detection | ✅ |

→ All pass → takes the fast quantized image shortcut.

## Quantized Path vs General Path

| | Quantized image path | General path |
|--|--|--|
| DSP | Inlined before inference | Separate DSP block loop |
| Memory | No float feature matrix | Allocates float matrix |
| Speed | Faster | Slower |
| Use case | Camera + INT8 model | Audio, sensors, non-quantized models |

## General Path DSP: extract_fn varies by model type

```cpp
// audio keyword spotting
block.extract_fn == extract_mfcc_features

// audio spectrogram
block.extract_fn == extract_spectrogram_features

// image
block.extract_fn == extract_image_features
```
