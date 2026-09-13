# STM32 IoT Custom Dongle

Custom 4-layer PCB designed for GNSS tracking and fall detection.

## Hardware Overview
![PCB Render](images/pcb_render.png)
![Real Board](images/pcb_photo.jpg)

### Key Specs
* **Microcontroller:** STM32 (ARM Cortex-M)
* **Stackup:** 4 Layers (Signal - GND - Power - Signal)
* **Connectivity:** GSM/GNSS module via AT Commands (UART)
* **Sensors:** 6-axis IMU interfaced via I2C/SPI

## Manufacturing & Files
* [Download Complete Schematics (PDF)](hardware/schematic.pdf)
* Gerber files available in the `/hardware/gerber` directory.
