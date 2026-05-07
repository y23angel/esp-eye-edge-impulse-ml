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

---

## Hardware & Software

### Hardware
- ESP-EYE (ESP32)  
- Micro-USB cable  
- Windows PC  

### Software / Tools
- Edge Impulse Studio  
- ESP-IDF (5.x)  
- ESP-IDF VSCode Extension

---

## Study Notes

| # | Topic | Status |
|---|-------|--------|
| 01 | [Boot & Initialization](notes/01-boot-and-init.md) | Done |
| 02 | [UART & AT Commands](notes/02-uart-at-commands.md) | Done |
| 03 | [Inference Loop](notes/03-inference-loop.md) | Done |
| 04 | [process_impulse — Branch Selection](notes/04-process-impulse.md) | Done |
| 05 | [FOMO Post-processing](notes/05-fomo-postprocessing.md) | Done |
| 06 | I2C & Accelerometer | Upcoming |
