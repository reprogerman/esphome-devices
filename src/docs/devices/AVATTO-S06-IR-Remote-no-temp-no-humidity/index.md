---
title: AVATTO S06 WiFi IR Universal Remote Controller (No Temp/No Humidity)
date-published: 2021-04-06
type: misc
standard: global
board: esp8266
---
![Product Image](/AVATTO-S06-WiFi-IR-Universal-Remote-Controller.jpg "Product Image")

## GPIO Pinout

| Pin    | Function           |
| ------ | ------------------ |
| GPIO4  | Blue Status LED    |
| GPIO14 | Remote Transmitter |
| GPIO5  | Remote Receiver    |
| GPIO13 | Reset Button       |

## Sensor temperature and humedad
sensor:
  - platform: aht10
  temperature:
    name: "Temperatura"
    filters:
    - offset: 0
    id: temp22
  humidity:
    name: "Humedad"
  address: 0x38
  update_interval: 5s

## GPIO Pinout humidity and temperature sensor
i2c:
  sda: GPIO1
  scl: GPIO3
  scan: true

## Getting it up and running

This device needs very likely to be flashed manually. Tuya-convert didn't worked for me. I have powered the device with
a mobile charger during the flashing procedure, as the used serial cable sets to logic level based on the input voltage.
Which is 3.3V in this case.

| Serial | ESP         |
| ---    | ----        |
| RX     | TX          |
| TX     | RX          |
| RTS    | RESET       |
| 3.3V   | 3.3V        |
| GND    | GND         |
|        | GPIO0 (GND) |

### Flashing

```bash
# Create a backup of the original firmware
esptool.py -p /dev/ttyUSB0 -b 460800 read_flash 0x00000 0x100000 avotta_s06_original_firmware.bin

# Erase the flash
esptool.py -p /dev/ttyUSB0 erase_flash

# Upload the esphome firmware
esptool.py -p /dev/ttyUSB0 write_flash -fs 1MB 0x0 ir_remote.bin
```

### Hardware Hack

If you want to wake the device via the reset button on the bottom from deep sleep. You need to solder a peace of wire
between the GPIO13 and the RESET pin of the ESP8266 MCU.

## Configuration

```yaml
# Basic Config
---

# https://esphome.io/devices/esp8266.html
esphome:
  name: ir_remote
  platform: ESP8266
  board: esp01_1m

wifi:
  ssid: !secret ssid
  password: !secret wlan_password
  fast_connect: true
  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: IR Remote Fallback Hotspot
    password: !secret ap_password

captive_portal:

# Enable logging
logger:
  level: DEBUG # Default

# Enable Home Assistant API
api:
  password: !secret ha_password

ota:
  password: !secret ota_password

# https://esphome.io/components/status_led.html
status_led:
  pin: 4
i2c:
  sda: GPIO1
  scl: GPIO3
  scan: true
# LED  With this configuration the led appears and can be turned off and on   

output:
  - platform: esp8266_pwm
    id: esphome_ir1_led
    pin:
      number: GPIO4
      inverted: false
      
light:
  - platform: monochromatic
    name: "Blue LED"
    output: esphome_ir1_led
    id: light_led

# https://esphome.io/components/remote_receiver.html
remote_receiver:
  id: rcvr
  pin:
    number: 5
    mode: INPUT_PULLUP
    inverted: True

# https://www.esphome.io/components/remote_transmitter.html
remote_transmitter:
  id: tamtr
  pin: 14
  carrier_duty_percent: 50%

# https://esphome.io/components/climate/ir_climate.html
climate:
  - platform: toshiba
    id: ac
    name: AC'
    receiver_id: rcvr
    transmitter_id: tamtr
    # Sensor temperature
    sensor: temp22

# Built-in aht10 temperature and humidity sensor
sensor:
  - platform: aht10
  temperature:
    name: "Temperatura"
    filters:
    - offset: 0
    id: temp22
  humidity:
    name: "Humedad"
  address: 0x38
  update_interval: 5s
switch:
  - platform: template
    name: 'AC Preset'
    id: ac_preset
    icon: mdi:cached
    turn_on_action:
      - climate.control:
          id: ac
          mode: COOL
          target_temperature: 24°C
          fan_mode: AUTO
          swing_mode: VERTICAL

    # https://esphome.io/components/switch/shutdown.html
  - platform: shutdown
    name: "IR Remote Shutdown"
    id: ir_remote_shutdown
```
