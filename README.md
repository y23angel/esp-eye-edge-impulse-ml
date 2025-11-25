
# ESP-EYE + Edge Impulse: Embedded Object & Audio Recognition (Ongoing)

This project implements **on-device machine learning** using the **ESP-EYE (ESP32)** development board and **Edge Impulse**.  
The goal is to capture image/audio data, train ML models, and deploy real-time inference directly on the embedded device.

---

## 📌 Project Overview

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

## 🛠 Hardware & Software

### **Hardware**
- ESP-EYE (ESP32)  
- Micro-USB cable  
- Windows PC  
- Optional: USB-UART adapter

### **Software / Tools**
- Edge Impulse Studio  
- ESP-IDF (5.x recommended)  
- Python 3  
- esptool / idf.py  
- Git  
- Serial console (PuTTY, screen, minicom, etc.)

---

## ⚙️ Setup Steps

### **1. Clone ESP-EYE Edge Impulse Firmware**
```bash
git clone https://github.com/edgeimpulse/firmware-espressif-esp32
cd firmware-espressif-esp32
idf.py set-target esp32
idf.py build
