# Water Level  - Solar Powered - ESP32 base

This project is about a well's water level solar powered.
The main point here is to have an autonomous off grid system due to the outdoor location.
The battery capacity will be monitored and a deep sleep state is used to prevent the discharge.


# Macro view

This is a macro view of the project

<img src="./images/macrodiagram.png" width="800">


# List of materials

- 1 x ESP32 Wroom : 
- 1 x Solar panel 6V :
- 1 x Li-On battery 18650 :
- 1 x HC-SR04+ ("plus" is important to have 3.3V version) : 
- X x Resistors XX : 

Total cost arround : 10$


# Electrical schema


<img src="./images/elecschema.png" width="800">

# Assembly


## Antenna 

<img src="./images/antennatipphoto.png" width="800">


# ESPhome code

```yaml

# Define the name of the ESP
esphome:
  name: water-level
  friendly_name: water-level

# Define the boot priority and script to execute at startup
  on_boot: 
    priority: -100
    then:
      - script.execute: deep_sleep_evaluation


# Define the kind of board
esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "yourpassword-generated-esphome"

ota:
  password: "yourpassword-generated-esphome"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "water-level Fallback Hotspot"
    password: "yourpassword-generated-esphome"

captive_portal:


deep_sleep:
  id: deep_sleep_enabled
  # run_duration: 10min
  sleep_duration: 30min


binary_sensor:
  - platform: status
    name: "Water-Level Status"
  
  - platform : homeassistant
    id: disable_deep_sleep
    entity_id: input_boolean.disable_deep_sleep


script: 
  - id: deep_sleep_evaluation
    mode: queued
    then:
      - delay: 10min
      - if:
          condition:
            binary_sensor.is_on: disable_deep_sleep
          then:
            - logger.log: 'Deep Sleep Disabled'
          else:
            - deep_sleep.enter: deep_sleep_enabled
      - script.execute: deep_sleep_evaluation


sensor:
  - platform: ultrasonic
    trigger_pin: 25
    echo_pin: 26
    name: "puit hauteur"
    id: measured_distance
    accuracy_decimals: 4
    update_interval: 30s

  - platform: uptime
    name: "Water-Level Uptime"  
    update_interval: 60s

  - platform: wifi_signal
    name: "Water-Level WiFi Signal"
    update_interval: 120s

  - platform: adc
    pin: GPIO35
    id : "ADCV"
    name: "Water-Level Battery V"
    update_interval: 60s
    attenuation: 11db
    filters:
      - multiply: 1.292
  
  - platform: template
    name: "Water-Level Battery %"
    unit_of_measurement: '%'
    update_interval: 60s
    lambda: |-
      return ((id(ADCV).state -3.30) /0.82 * 100.00);

switch:
  - platform: restart
    name: "Water-Level Restart"


```

# PCB creation

JLCPcb

# Case creation 

** soon **
