# WORX can logger

Project to have canbus logging of accessories 

## Components

Provided you've the necessary hardware, you need to create the `secrets.h` file based on the `secrets_example.h` file with the proper environment variables.

Hardware:

* `ESP32` -> [link](https://amzn.to/3pe0XVP)
* `Can Transceiver` -> [Waveshare SN65HVD230](https://www.banggood.com/Waveshare-SN65HVD230-CAN-Bus-Module-Communication-CAN-Bus-Transceiver-Development-Board-p-1693712.html?rmmds=myorder&cur_warehouse=CN)

## Connections

  * Transceiver -> Mower
  
    * CAN_H => Yellow
    * CAN_L => Green
  
  * Transceiver-ESP32

    * GND -> GND
    * 3.3 -> 3.3
    * TX -> 16
    * RX -> 17
  
  * ESP32 -> Mower

    * GND -> BLACK
    * VCC(5V) -> RED

## Logging

### Method 1 (via MQTT and SavvyCan)

SavvyCan is an utility mostly used to debug and analyze can data. You can push directly to mosquitto server using savvycan data format to live inspect whats coming from the esp32.

To do so you need to configure SavvyCan setting the MQTT broker info within the *preferences* and adding a MQTT bus in new connection panel with topic "can2"

After doing so you should be able to see data coming from the canbus. Once you are finished capturing you can hit the "Suspend Capturing" button and save the file as csv. 

*Best Practices*:

  * Take short logs and name log file with whatever happened. Ex: `mower-into-trapped-state.csv` or `offlimits-accessory-detect-magnetic-boundary.csv`
  * You can submit log file in an issue here in github or create pull request including the log in the `canbus` folder


## Utils

There is a utils/ script called sendcan.py you need python and all the needed deps installed via `pip`

this utility let you send a message via cli to the canbus through the ESP32. 

For example if you want to send 0x610 02 0f (which is the message sent by the off limits accessory) you just need to do so

```
python sendcan.py -H MQTT_HOST -u MQTT_USER -p MQTT_PASS -P MQTT_PORT -t can2 -i 0x610 -d "02 0f"
```
## CanBus info

  * 500kbit/s standard format.
  * Mower knows how to recognize the attached accessory type
        * Mower sends 0x611 status byte periodically
        * Accessories send their version number on their CAN id periodically 
  * Mower status identification (maybe also version request to accessories?)
    * 0x611: (1 byte ) mower response of the message above
        * `0x01` mowing
        * `0x00` idle/home/not mowing in general
  * Off Limits module:
    * 0x610: (2 bytes) off limits accessory sends this message on the bus every 0.5s
        * `02 0f` is the version number of the off limits firmware
    * 0x418: (8 bytes): offlimits accessory sends such message when in mowing state . This can frame contains the information for the robot to take action. Last byte is `0x00` when in no magnetic tape is away. When magnetic tape is detected and mower is moving relatively to the tape than last byte changes.
    * 0x419 is also used for off limits
  * Radio link module
    * 0x640: (2 bytes) radio link accessory sends this message on the bus every 0.5s
        * `02 16` is the version number of the radio link firmware
    * 0x540: ISO-TP Channel Radio Link -> Mower,  Response for AT commands
    * 0x541: ISO-TP Channel Mower -> Radio Link, AT-Commands for MQTT
    * 0x542: ISO-TP Channel Radio Link -> Mower: Status info 16 bytes, hex `dc0ba197020101bbc100760015000100` with some bytes with minor changes
    * 0x543: ISO-TP Channel Mower -> Radio Link. I was only able to see ISO-TP FC (flow control) frames, consisting of 3 bytes, mostly `30 08 00` (3=ISO-TP FC message, 08 = length of single message accepted, 00 = rate limit none)

## ISO-TP Protocol on CAN-ID 0x540/0x541

### Reconnect to base station
* Command: `AT+ISBOOT\n`
* Response: `+ok=1,V.0.21`
* Command (repeated): `AT+S\n`
* Responses: `+ok=2520` (probably before reset was executed), then `+ok=0000`, then `+ok=2000`
* Command: `AT+MQINIT=cmd-eu-ats-prod.worxlandroid.com,34865D79C360,60,1\n`
* Response: `+ok=`
* Command: `AT+MQSUB=0,PRM100/34865D79C360/commandIn`
* Response: `+ok=`
* Command: `AT+MQCON=2\n`
* Response: `+ok=`
* Command: `AT+S\n`
* Response: `+ok=2100`, then `+ok=2200`, then `+ok=2400`, then `+ok=2500` for a long time
* (Response changes after first PUBINFO-Command to `+ok=2520`)

### Query for data
* Command: `AT+S\n`
* Response: `+ok=2521` (New message!)
* Command: `AT+MSGINFO\n`
* Response: `+ok=PRM100/<MAC>/commandIn,<LENGTH>,C902` (MSGINFO Response. MQTT-Channel followed by the message length some checksum? NOT the CRC described below!)
* Command: `AT+MSGRD=0\n` (Read message, begin at byte 0)
* Response: `+ok=eyJpZCI6MzM1NDgsImNtZCI6MCwibGciOiJkZSIsInNuIjoiMjAyMjMwMjY3MzA0MDAxNjM3RDciLCJ0bSI6IjE5OjM3OjM3IiwiZHQiOiIxMy8wNi8yMDI0In0,E6F1` (Message, BASE64 encoded + CRC-16). Message reads `{"id":33548,"cmd":0,"lg":"de","sn":"...........","tm":"19:37:37","dt":"13/06/2024"}`. In this case: Please update mqtt status.
* Command: `AT+MSGRD=5C\n` (Read message, begin at byte 0x5C)
* Response: `+ok=done` (No bytes left)
* Command: `AT+S` (Status query)
* Response: `+ok=2520` (Normal status reply)

### Send status update        
* Command: AT+PUBWR=0000,<base64-data>,<CRC>\n` (Write data, offset 0000, data base64 encoded, CRC see below)
* Response: +ok=
* Command: AT+PUBWR=0096,<base64-data>,<CRC>\n`
* Response: +ok=
* ....
* Command: `AT+PUBINFO=PRM100/<MAC>/commandOut,<LENGTH>,<CRC>` (Write data to MQTT topic, MAC address of mower, length of base64 decoded message, CRC see below)
* Response: +ok=
* After first send message, the status info changes from +ok=2500 to +ok=2520

## CRC algorithm
The CRC used for the base64 encoded data is calculated using CRC-16/ARC on the base64 decoded (plain text) data. Found using https://crccalc.com/.

### C
#include <stddef.h>
#include <stdint.h>

uint16_t crc16arc_bit(uint16_t crc, void const *mem, size_t len) {
    unsigned char const *data = mem;
    if (data == NULL)
        return 0;
    for (size_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (unsigned k = 0; k < 8; k++) {
            crc = crc & 1 ? (crc >> 1) ^ 0xa001 : crc >> 1;
        }
    }
    return crc;
}



## Why?
  
There are several reasons why I started working on this. First of all worx accessories are able to both control mower and read its full state (radiolink accessory). 

It's my belief (not confirmed) that by fully reverse engineering the canbus messages we will be able to do wonderful stuff such as:

  * have GPS RTK based navigation (or lidar or vision only?)
  * have a better "untrap routine"
  * create your own range extender (EX: radiolink clone) based on rf433mhz 
  * isolate the mower from the internet and have the status data sent locally. This may also keep the landroid running and controllable even when no internet connection is available

## Want to help?

You can help in many ways:

  1. If you've  aworx with an attached accessory you can build this with platformio and deploy it to an esp32. Once you collect the logs you can send it over as described above.
  2. If you want to help reverse engineer the bus messages you can do it. For now all investigation on such worx mowers is done in the OpenMower [discord](https://discord.gg/jE7QNaSxW7) on this  [channel/thread](https://discord.com/channels/958476543846412329/966633787133947914).


