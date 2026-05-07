# 01. Boot & Initialization

**File**: `main/main.cpp`

## Execution Flow

```
app_main()
 ├── setup_led()               GPIO 21 (RED), GPIO 22 (WHITE)
 ├── ei_inertial_init()        LIS3DHTR accelerometer via I2C
 ├── ei_analog_sensor_init()   ADC sensor on GPIO34
 ├── ei_at_init()              UART AT command server (115200 baud)
 └── while(1)
       data = ei_get_serial_byte()
       at->handle(data)
```

## FreeRTOS Task (blink_task)

```cpp
void blink_task(void *pvParameters) {
    while(1) {
        gpio_set_level(RED_LED_PIN, 0);
        vTaskDelay(pdMS_TO_TICKS(1000));
        gpio_set_level(RED_LED_PIN, 1);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

// Start the task:
xTaskCreate(blink_task, "blink_task", 4096, nullptr, 5, nullptr);
```

## GPIO Pin Conflict

GPIO 21/22 are shared between LEDs and the I2C bus (accelerometer).  
`ei_inertial_init()` reconfigures these pins for I2C → LED control stops working.

| Pin | LED role | I2C role |
|-----|----------|----------|
| GPIO 21 | RED LED | SDA (data) |
| GPIO 22 | WHITE LED | SCL (clock) |

**Fix for blink test**: comment out `ei_inertial_init()` while testing LEDs.
