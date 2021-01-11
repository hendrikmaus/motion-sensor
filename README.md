# Motion Sensor

A DIY radar motion sensor based on ESP8266.

This is for you, if you are unhappy with PIR sensors.

## Table of Contents

- [Components](#components)
- [Wiring](#wiring)
- [Example Integration](#example-integration)
- [3D printable case](#3d-printable-case)

## Components

tl;dr

- [Microcontroller - Wemos D1 mini (ESP8266)](https://www.wemos.cc/en/latest/d1/d1_mini.html)
- [Ambient Light Sensor (BH1750)](https://www.mouser.com/datasheet/2/348/bh1750fvi-e-186247.pdf)
- [Radar Motion Sensor - RCWL-0516](https://www.epitran.it/ebayDrive/datasheet/19.pdf)
- Micro USB breakout board
- 5V 1A Micro USB power supply

The **Wemos D1 mini** is a convenient component for two main reasons:

1. It can be powered directly using 5V
2. It has a small form factor

However, any other ESP8266 can be used. Be aware that you might need additional components though and the provided 3D printed case might not fit your setup then.

The **ambient light sensor** is optional, since the RCWL-0516 can also be hidden in drawers etc. where the on-board lux reading wouldn't be of much use anyway.

The **RCWL-0516** is used as a 360 degree motion sensor which proved to have a range of about 5 meters, at least in my indoor use cases. There is a [GitHub Repo](https://github.com/jdesbonnet/RCWL-0516) with lots of details and dicussions, but here is a tl;dr of what I have learned about this chip to use it properly:

- Power it from a stable source, i.e. *not* from a microcontroller which might experience voltage fluctuations when working on its wifi modem and whatnot
- Provide at least 5V, although others have reported stable usage with a stable 3.3V supply as well
- Use `VIN` to power the board
- Beware of *glass*; it does not work well, or not at all, when located in a glass cabinet or similar

The micro **USB breakout board** is handy, because:

1. The RCWL-0516 can be hooked up to 5V

    It cannot be powered off of the Wemos board; it needs to be powered seperately to function without false positives.

2. It is easier to mount inside of a case to expose the micro USB port to the exterior 

## Wiring

```txt
USB Breakout Board
  GND  -> Wemos GND
  5V   -> Wemos 5V

Wemos
  D1  -> BH1750 SCL
  D2  -> BH1750 SDA
  GND -> BH1750 GND
  3V3 -> BH1750 3V3

RCWL-0516
  OUT -> Wemos D6
  5V  -> USB Breakout Board 5V
  GND -> Wemos GND
```

*Want to contribute? Turn this into a proper schematic diagram using [KiCad](https://kicad.org), for example.*

## Example Integration

I used [esphome.io](https://esphome.io) to create a custom firmware for the microcontroller. It publishes data via [MQTT](https://mqtt.org) and I consume and use the sensor data with [FHEM](https://fhem.de).

Esphome configuration example:

> Read carefully and refer to the esphome documentation.

```yaml
esphome:
  name: device_name
  platform: ESP8266
  board: d1_mini

wifi:
  ssid: your_wifi_ssid
  password: your_wifi_password
  use_address: resolvable_hostname_or_fixed_ip

logger:

# Disable Home Assistant API
# MUST be disabled for use with MQTT
#api:

mqtt:
  broker: your_broker_ip_or_host
  username: mqtt_username
  password: mqtt_password
  client_id: device_name
  discovery: False
  topic_prefix: esphome/device_name

ota:

web_server:
  port: 80

status_led:
  pin:
    number: GPIO02
    inverted: True

i2c:
  sda: GPIO4
  scl: GPIO5
  scan: True
  id: bus_a

binary_sensor:
- platform: gpio
  name: "motion"
  pin:
    number: D6
  device_class: motion
- platform: status
  name: "status"
  
sensor:
- platform: wifi_signal
  name: wifi_signal
  update_interval: 10s
- platform: uptime
  name: uptime
- platform: bh1750
  name: "lux"
  address: 0x23
  update_interval: 60s  

text_sensor:
- platform: version
  name: version
- platform: wifi_info
  ip_address:
    name: ip_address
  ssid:
    name: connected_ssid
  bssid:
    name: connected_bssid
  mac_address:
    name: mac_address
```

FHEM integration:

*Of course you can use any automation platform that can consume MQTT messages.*

> `device_name` should be replaced, as well as `mqtt_broker`.

The values for the attribute `readingList` match the `topic_prefix` in the esphome configuration; make sure to have them match up.

```perl
define device_name MQTT2_DEVICE
attr device_name IODev mqtt_broker
attr device_name devStateIcon ON:people_sensor@orange OFF:people_sensor
attr device_name readingList esphome/device_name/status:.* LWT\
esphome/device_name/sensor/version/state:.* version\
esphome/device_name/sensor/uptime/state:.* uptime\
esphome/device_name/sensor/lux/state:.* lux\
esphome/device_name/sensor/wifi_signal/state:.* rssi\
esphome/device_name/sensor/mac_address/state:.* mac_address\
esphome/device_name/sensor/ip_address/state:.* ip_address\
esphome/device_name/sensor/connected_ssid/state:.* connected_ssid\
esphome/device_name/sensor/connected_bssid/state:.* connected_bssid\
esphome/device_name/binary_sensor/motion/state:.* state
```

FYI: I then use a `notify` which looks at the `lux` reading and decides whether or not to turn some lights on. A `watchdog` is used to turn any lights off once no motion has been detected for a given time.

## 3D Printable Case

Please find an example for a printable case over on [Thingiverse](https://www.thingiverse.com/thing:4718903).

I used white PETG which allows for the blue onboard status LED on the Wemos to shine through.

I created the models using [FreeCAD](https://www.freecadweb.org/); please find the source file in this repository.


