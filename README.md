
# ESPHome-SONOFF-Misc
Miscelaneous ESPHome .yaml files for a variety of SONOFF devices I have at home.

Files should be self-explanatory.

However, it could be worth dropping a few remarks and a summary for what the files contain and how I got them to do what I wanted.

## SONOFF TX Light Switches
First, there is a ton (14) of [SONOFF TX WiFi Smart Wall Touch Switches](https://sonoff.tech/product/smart-wall-switches/tx-series/) which were chosen at the design phase of my home, to be used instead of the traditional wall switches for lighting. There are five 1-gang, seven 2-gang and two 3-gang units throughout the house. Of course, all of them flashed with ESPHome, to get rid of the vendor cloud and keep everything close to home and controlled by [Home Assistant](https://www.home-assistant.io/). The way to get these devices flahsed may be found somewhere on the Internet.

Some of the devices are very simple to configure (as they command light fixtures directly), namely, the two 3-gang TX are used in bathrooms were three different lights (tub, ceiling and mirror) may be switched on and off individually, but only from that single place. Same goes for some of the 1-gang and 2-gang SONOFF TXs, as they command one or two light fixtures (or sets thereof) from a single place.

It becomes more interesting when you want to be able to switch some light on / off from two or more places and for all the WiFi switches to be kept in sync. In the end, no matter the switches controlling a set of lights, there will be a single one of them electrically connected to the lights (Live and Neutral wires in my case), the rest of the SONOFF TXs just getting fed with grid power for working. You may look into the following files to see how I did the coordination in a three-switch setup controlling the lights in the hall / corridor (one switch on each end and one in the middle) :

 - sonoff_txt01c_03.yaml : middle of the hall / corridor, a 1-gang SONOFF TX
 - sonoff_txt02c_03.yaml : left end of the hall / corridor, a 2-gang SONOFF TX
 - sonoff_txt02c_05.yaml : right end of the hall / corridor, a 2-gang SONOFF TX

Lights are electrically connected to one of the relays of the 2-gang SONOFF TX at the right. The way they coordinate light status is through MQTT messages and subscribing to specific topics. Using the 1-gang SONOFF TX in the middle to explain further, obviously when we physically touch the switch we want it to toggle the lights, so ESPHome code reads as :
```yaml
    binary_sensor:
      - platform: gpio
        name: "Hall Center Touchpad Ceiling"
        pin:
          number: GPIO0
          mode: INPUT_PULLUP
          inverted: True
        filters:
          - delayed_on: 10ms
        on_press:
          - switch.toggle: relay_1
```
Plain simple : touch the touchpad, then toggle the local relay. Which is defined in the ESPHome file like this :
```yaml
    switch:
      - platform: gpio
        name: "Hall Center Relay Ceiling (Unused)"
        restore_mode: RESTORE_DEFAULT_OFF
        pin: GPIO12
        id: relay_1
        on_turn_on:
          - mqtt.publish:
              topic: home/indoors/hall/switch/ceiling/num_state
              payload: 1
        on_turn_off:
          - mqtt.publish:
              topic: home/indoors/hall/switch/ceiling/num_state
              payload: 0
```
Relay is not physically connected to lights (hence I tag it Unused). However when I touch this touchpad, if the relay goes on, I want this to be published through MQTT (mqtt.publish) to a topic assigned for this lights in particular, and the same goes when I turn the lights off by touching the pad on this 1-gang SONOFF TX. I used "1" for "on" and "0" for "off", but you may use anything else.

If this device (or any of the other two commanding the same lights) want to notice when lights are being toggled from another device, they all have to subscribe to the topics above, and act accordingly on receiving the appropriate messages :
```yaml
    mqtt:
      broker: !secret mqtt_server_ip_address
      port: 1883
      discovery: True
      discovery_retain: True
      discovery_prefix: "homeassistant"
      topic_prefix: home/indoors/hall/center
      on_message:
        - topic: home/indoors/hall/switch/ceiling/num_state
          payload: "1"
          then:
            - switch.turn_on: relay_1
            - delay: 1s
        - topic: home/indoors/hall/switch/ceiling/num_state
          payload: "0"
          then:
            - switch.turn_off: relay_1
            - delay: 1s
```

So all three mentioned SONOFF TXs controlling the lights at the ceiling of the hall / corridor are subscribed to the mentioned MQTT topic so when a "0" (turn off) or a "1" (turn on) are published, they set the local relay to the correct status (so LED dims if lights off and status is consistent across devices).

## Building-integrated temperature and humidity sensors
When you plan your new house from the ground up you risk over engineering some things, and inserting a bunch of home-made sensor (temperature and humidity) bundles through the walls and ceilings may look like it, but it makes for a learning experience and you can also see how a building behaves with changing temperatures and humidity, and how important having the right amount of thermal insulation is.

Files for these two sensors are :

 - esp12f_01.yaml : consists of a BME280 temp, RH and pressure sensor at the outwards ends of the ceiling assembly, a DS18B20 temperature sensor at the middle, and another BME280 at the inwards end of the ceiling. Ceiling consists of a 30-cm. wide sandwich made of OSB-3 boards at both ends
 - esp12f_02.yaml : consists of four DS18B20 temperature sensors stuffed within the wall at different points in between the inner side of the outside facade concrete blocks, and the inner side of the plasterboard

These two sensor assemblies are put together around ESP8266 development boards, the most significant part is the use of a single I2C for the two BME260 sensors for the ceiling sensor assembly. On the other hand DS18B20 1-wire sensors are a breeze to work with and the code says it all anyways.

## Outdoors roller blind / window shade control with  a SONOFF DUAL R3

Configuring this (west_kitchen_blind_control.yaml) was only as complex as timing how long the roller blind motor takes to open fully from closed and vice versa, to put the details in the ESPHome YAML file for a time-based cover. Note this roller blind has hardware limit switches, so it is not exactly critical for the time measured to be correct, to avoid burning the motor, but still has to be pretty spot on or else blind may open less or more than expected when not doing so fully.

The [SONOFF DUAL R3](https://sonoff.tech/product/diy-smart-switches/dualr3/) is a very interesting piece of kit.

## Make two dumb towel heaters smart with a Wi-Fi relay (SONOFF Mini R2)
This required some deep hardware surgery on the towel heaters, as they had their very own electronic control, and an external Wi-Fi switch would not be able to do anything useful. Remove the circuits and put a [SONOFF Mini R2](https://sonoff.tech/product/diy-smart-switches/minir2/) in the housing instead. Files are sonoff-minir2-02.yaml and sonoff-minir2-03.yaml .

This works as simple as when the relay goes ON current flows through the heater electrical resistance and makes your towels warm, when you set the relay to OFF, it stops. Works simply and beautifully except for one thing I noticed experimentally : the SONOFF Mini R2 seems to switch off at 40 ºC for thermal protection, and this happens easily in the tower heater enclosure when active, so eventually after a few minutes the Wi-Fi relay reboots and towel heater goes OFF.

To be fixed ahead of next winter but taking the Wi-Fi relay out of the towel heater and into the in-wall socket.

## Using SONOFF 4-channel Wi-Fi relay products to control several circuits at a time

I have two of them (see files sonoff_4chr2_01.yaml and sonoff_4chpror2_01.yaml), namely :

 - A [SONOFF 4CH R2](https://sonoff.tech/product/diy-smart-switches/4chr3-4chpror3/), at the attic : this controls the lights on the four facades of the house
 - A [SONOFF 4CH Pro R2](https://sonoff.tech/product/diy-smart-switches/4chr3-4chpror3/), at the basement, which I use to ON / OFF a few things (namely, a separate Wi-Fi AP, the entrance lighting, basement lighting and pool pump)

Note in both cases you can control four channels but there is only one single power circuit, so watch our for your local electrical code and the total combined power you may be using, to avoid frying your hardware and potentially causing a fire.

There is nothing particularly fancy or interesting about any of those two one, but have been extremely useful and compact in my setup. Note wiring is different for the 4CH and the 4CH Pro, the second being less intuitive.

## Using a SONOFF DUAL R3 as 2-channel Wi-Fi relay and voltage reference

I needed a voltage measurement at home for power calculations so I went for this [SONOFF DUAL R3](https://sonoff.tech/product/diy-smart-switches/dualr3/) which comes with power measurement (although I only use the grid voltage measurement). YAML code is in file dual_relay_voltage_telecom_closet.yaml .

This is probably the only device for which I found [Tasmota](https://tasmota.github.io/docs/) to be easier and more complete than [ESPHome](https://esphome.io/), but ESPHome was good enough for me and sticked with it. I grabbed the YAML from other places but it is otherwise self-explanatory.

NOTE : the only other place where I did not use ESPHome but Tasmota was on two old [SONOFF Basic R1](https://sonoff.tech/product/diy-smart-switches/basicr2/) 1-channel Wi-Fi relays , which somehow did not work properly with ESPHome but did fine with Tasmota.

