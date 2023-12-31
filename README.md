# Water Level  - Solar Powered - ESP32 base

This project is about a well's water level solar powered.

The main point here is to have an autonomous off grid system due to the outdoor location.

The battery capacity will be monitored and a deep sleep state is used to prevent the full discharge.

I'm using an HC-SR04+ ultrasound system to get the distance between the water and the sensor.


# Macro view

This is a macro view of the project.

Solar panel will be placed on the roof. 

The esp32, some electronics and battery will be placed just under the roof in a box.

The HC-SCR04+ will be placed in the well iteself

<img src="./images/macrodiagram.png" width="600">


# List of materials

**Be informed that all links are affiliated, feel free to find it by your side if you don't want to use the following links**

Main assembly :

- 1 x Solar panel 6V : https://s.click.aliexpress.com/e/_DezvD27
- 1 x TP4056 USB-C : https://s.click.aliexpress.com/e/_Dko0C5N
- 1 x Li-On battery 18650 : https://s.click.aliexpress.com/e/_DlPwYRp
- 1 x 18650 battery holder : https://s.click.aliexpress.com/e/_DBchngR
- 1 x MCP1700-3302E : https://s.click.aliexpress.com/e/_DCsN7Wx
- 1 x 100nF capacitor : https://s.click.aliexpress.com/e/_DBgmV2j
- 1 x 100µF capacitor : https://s.click.aliexpress.com/e/_Dlit8gF
- 1 x 27k ohm resistor : https://s.click.aliexpress.com/e/_DEmGfjN
- 1 x 100k ohm resistor : https://s.click.aliexpress.com/e/_DEmGfjN
- 1 x ESP32 Wroom : https://s.click.aliexpress.com/e/_DlSmJxN
- 1 x HC-SR04+ ("plus" is important to have 3.3V version) : https://s.click.aliexpress.com/e/_Dcf8nF9
- *you can also use RCWL-1601 instead* : https://s.click.aliexpress.com/e/_DdKQkZV 

Optional : 

- 1 x electrician box : https://s.click.aliexpress.com/e/_DFK50aP
- 1 x long distance cable to connect the HC-SR04+ : https://s.click.aliexpress.com/e/_DCSRG5d
- 1 x soldering board : https://s.click.aliexpress.com/e/_DDgjZ0B
- 1 x external coaxial antenna : https://s.click.aliexpress.com/e/_DkJyf2F

Add more if you want to have solder plate and ready to use box


# Electrical diagram

Here I'm using the solar panel connected to the TP4056 in the input voltage.

Battery will be connected between battery+ and battery- on the TP4056.

To power the ESP32 I will use a voltage regulator system composed by  MCP1700-3302E + a 100nF capacitor + a 100µF capacitor to have exactly 3.3V output.

Also using a voltage divider to get the voltage of the battery and repot it to GPIO35

The HC-SR04+ will be connected to GPIO25 (for trigger) and GPIO26 (for echo).
 
<img src="./images/elecschema.png" width="800">


# Assembly

Here is a view of the first prototype (all in a big electrician box) 

## Inside the box

Not really clean, fixing all with some tape.

The TP4056 is placed in a side of the box and protected with USB-C plastic protection. This let me the possibiltiy to charge the battery from outside the box without having to open it.

<img src="./images/assembly1.png" width="600">

## Antenna 

You may need to extend the wifi range of the ESP32. 

By default the range is already not so good, but inside a box and from a long distance of the Wifi access point... It can be complicated.

To prevent this I have added an external antenna 

You can solder it directy by scratching a bit the integrated antenna like this : 

<img src="./images/antennaextension.jpg" width="600">

And solder the antenna like this + using some glue to keep it in place : 

<img src="./images/antennatipphoto.png" width="600">

## Outside the box

This first version is assembled like this.

We have a long cable to join the HC-SR04+.

<img src="./images/assembly2.png" width="600">


# Home Assistant

Now we will see how to integrate this in Home Assistant

## Prevent deepsleep button

Before going to the ESPHome code, we will first create a "helper" in Home Assistant, this helper will be a switch we can activate to prevent the ESP32 deepsleep mode.

Go to Settings > Devices & Services > Helpers

And add a new helper in input.boolean type and name it like "disable_deep_sleep" (it is the name I will use in the code further, adapt it with the name you choose).

<img src="./images/hadeepsleepfakebutton.png" width="600">

## ESPHome code

Find below the code used in Home Assistant for ESPHome code used on the ESP32, see the code comments to have more explanation : 

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

# Configure the sleep duration, the run duration will be configured a bit further, I set it to 30 min, you can change with what you want
deep_sleep:
  id: deep_sleep_enabled
  sleep_duration: 30min

# Configure sensor for status, will be used to determine if the deepsleep is really working
binary_sensor:
  - platform: status
    name: "Water-Level Status"

# Configure the input boolean used to prevent ESP32 to go in deepsleep 
  - platform : homeassistant
    id: disable_deep_sleep
    entity_id: input_boolean.disable_deep_sleep

# Scrit used to go in running mode for 10 min if input boolean is not pressed, if pressed, it goes to full run mode
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

#Configure the ultrasonic sensor HC-SR04+ to pin 25 and 26 + accuracy and interval
  - platform: ultrasonic
    trigger_pin: 25
    echo_pin: 26
    name: "puit hauteur"
    id: measured_distance
    accuracy_decimals: 4
    update_interval: 30s

# Will indicate the uptime of the ESP32, usefull to know if the deepsleep is doing his job or not
  - platform: uptime
    name: "Water-Level Uptime"  
    update_interval: 60s

# Got the wifi signal status in dB
  - platform: wifi_signal
    name: "Water-Level WiFi Signal"
    update_interval: 120s

# Configure the GPIO used for adc (to get the voltage of the battery)
  - platform: adc
    pin: GPIO35
    id : "ADCV"
    name: "Water-Level Battery V"
    update_interval: 60s
    attenuation: 11db
    filters:
      - multiply: 1.292

# Estimate a % from the Voltage of the battery
  - platform: template
    name: "Water-Level Battery %"
    unit_of_measurement: '%'
    update_interval: 60s
    lambda: |-
      return ((id(ADCV).state -3.30) /0.82 * 100.00);

# A switch to force the ESP32 to restart
switch:
  - platform: restart
    name: "Water-Level Restart"


```


# Next steps

## Add a Home Assistant view or capture

## Add a lovelace card to represent the water level in the well

## PCB creation with EasyEDA + add to the release page

## Case creation 

## update release page with 3D file, PCB + gerberfile + BOM and YAML
