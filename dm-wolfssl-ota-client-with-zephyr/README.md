# wolfSSL NXP Application Code Hub

<a href="https://www.nxp.com"> <img src="https://mcuxpresso.nxp.com/static/icon/nxp-logo-color.svg" width="125" style="margin-bottom: 40px;" /> </a> <a href="https://www.wolfssl.com"> <img src="../Images/wolfssl_logo_300px.png" width="100" style="margin-bottom: 40px" align=right /> </a>

## wolfSSL OTA client using Zephyr RTOS

This demo demonstrates capabilities of the new FRDM-MCXN947.  

### Demo
Creating a simple OTA client using the Zephyr RTOS and wolfMQTT to realize the OTA functionality under the TrustZone.
This app is intended to work with wolfBoot, which is a secure bootloader running in the secure world.
#### Boards:        FRDM-MCXN947
#### Categories:    RTOS, Zephyr, Networking
#### Peripherals:   UART, ETHERNET
#### Toolchains:    Zephyr

## Table of Contents
1. [Software](#step1)
2. [Hardware](#step2)
3. [Setup](#step3)
4. [Run Demonstrator](#step4)
5. [Sequence Diagram](#step5)
6. [FAQs](#step6)
7. [Support](#step7)
8. [Release Notes](#step8)

## 1. Software<a name="step1"></a>
- [MCUXpresso for VS Code 25.7.59 or newer](https://www.nxp.com/products/processors-and-microcontrollers/arm-microcontrollers/general-purpose-mcus/lpc800-arm-cortex-m0-plus-/mcuxpresso-for-visual-studio-code:MCUXPRESSO-VSC?cid=wechat_iot_303216)

- [Zephyr Setup](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)
    - [wolfSSL as a Module added to Zephyr](https://github.com/wolfSSL/wolfssl/blob/master/zephyr/README.md)
    - [Adding the Zephyr Repository (Part 5)](https://community.nxp.com/t5/MCUXpresso-for-VSCode-Knowledge/Training-Walkthrough-of-MCUXpresso-for-VS-Code/ta-p/1634002)

- MCUXpresso Installer:
   - MCUXpresso SDK Developer
   - Zephyr Developer
   - Linkserver

- Ubuntu or MacOS with the following packages:
    - autoconf
    - automake
    - libtool
    - make
    - gcc
    - git 

 - Zephyr:
    - SDK 1.0.1
    - Version 4.4.0
- wolfSSL
  - Version v5.9.1-stable
- wolfBoot
    - Version 2.8.0
- wolfMQTT
   - Version 2.0.0

- Optional Software:
    - Local mqtt broker like mosquitto

## 2. Hardware<a name="step2"></a>
- [FRDM-MCXN947.](https://www.nxp.com/products/processors-and-microcontrollers/arm-microcontrollers/general-purpose-mcus/mcx-arm-cortex-m/mcx-n94x-and-n54x-mcus-with-dual-core-arm-cortex-m33-eiq-neutron-npu-and-edgelock-secure-enclave-core-profile:MCX-N94X-N54X)   
[<img src="Images/FRDM-MCXN947-TOP.jpg" width="300"/>](Images/FRDM-MCXN947-TOP.jpg)
- USB Type-C cable.
- Ethernet Cable.
- Networking/Router
- Personal Computer.


## 3. Setup<a name="step3"></a>

### Import the Project
Follow section 1: `Setup` in the top level [README](../README.md).
The project should be called `dm-wolfssl-ota-client-with-zephyr`.
You need to several stuff before build.

### Add cacheVariables
This application runs in the non-secure world and is booted by wolfBoot running in the secure world.
Please add the following cacheVariables into your CMakePresets.json to use the ns board setting in your build:
```
"BOARD": "frdm_mcxn947/mcxn947/cpu0/ns",
"DTC_OVERLAY_FILE": "app.overlay",
"WOLFBOOT_ROOT": {
          "value": "/Path/To/Your/wolfBoot",
          "type": "PATH"
        },
```

### Setup wolfBoot
We need to build the secure boot loader using wolfBoot.
Please get the wolfBoot version 2.8.0 from official [Github](https://github.com/wolfSSL/wolfBoot).
Please apply `wolfbootConfig/0001-Update-configs-and-memory-map.patch` to add the specific initialization and change the memory map.
Then, please build it with `wolfbootConfig/.config`.

### Build and Prepare application image
You can trigger the build from the MCUXpresso for VS Code extension, or build manually.
```
cmake . --preset debug
cmake --build debug --parallel
```
You'll see the outputs like `debug/zephyr/zephyr.bin`.
To boot this application by wolfBoot, you need to add the image header and sign it using tools contained in wolfBoot.
First of all, we need to strip first 1024 bytes for image header space.
```
dd if=debug/zephyr/zephyr.bin of=debug/zephyr/zephyr_stripped.bin bs=1 skip=1024
```
Then, do this to sign the image:
```
IMAGE_HEADER_SIZE=1024 /Path/To/wolfBoot/tools/keytools/sign 
--ecc384 --sha384 /Path/To/YourProject/dm-wolfssl-ota-client-with-zephyr/debug/zephyr/zephyr_stripped.bin 
/Path/To/wolfBoot/wolfboot_signing_private_key.der 1
```
The output file like `debug/zephyr/zephyr_stripped_v1_signed.bin` is ready to flash now.
Please repeat the same command with version 2 or greater to produce another image file which is to be downloaded during OTA sequence.

### Build fwserver
fwserver is a simple OTA server app running on your laptop.
Build:
```
cd fwserver
mkdir build && cd build
cmake ..
cmake --build . --parallel
```
Then, please copy the downloadable image like `debug/zephyr/zephyr_stripped_v2_signed.bin` to the same directory with `fwserver/build/fwserver`.

### Prepare MQTT Broker
We recommend preparing a local MQTT broker for a stable evaluation environment.
You can use mqttBroker/dockerfile to build the mosquitto test broker in local.
For example:
```
cd mqttBroker
docker build -t mosquitto .
docker run -d -p 1883:1883 -p 8883:8883 --name mosquitto mosquitto
```
The current setting is to use 192.168.1.10/24 on port 8883 with TLS.
Please assign the IP address to your network interface.
If you want to change the target broker, you can define it using the macro DEFAULT_MQTT_HOST.
Or please edit the following code directly.
In fwserver/mqttexample.h:
```
#ifndef DEFAULT_MQTT_HOST
    /* Default MQTT host broker to use,
     * when none is specified in the examples */
    #define DEFAULT_MQTT_HOST   "localhost"
    /* "iot.eclipse.org" */
    /* "broker.emqx.io" */
    /* "broker.hivemq.com" */
#endif
```
In src/mqttClient/mqttexample.h:
```
#ifndef DEFAULT_MQTT_HOST
    /* Default MQTT host broker to use,
     * when none is specified in the examples */
    #define DEFAULT_MQTT_HOST   "192.168.1.10"
    /* "iot.eclipse.org" */
    /* "broker.emqx.io" */
    /* "broker.hivemq.com" */
#endif
```

### Connect Hardware
1. Connect the FRDM-MCXN947 to your computer with the provided USB-C Cable

2. Connect the FRDM-MCXN947 to your network with an Ethernet cable

### Program the wolfBoot and application
Flash both of wolfBoot and application image file to FRDM-MCXN947.
You can use LinkServer, which is installed with MCUXpresso toolchains.
```
/Path/To/YourLinkServer/LinkServer_25.12.83/LinkServer flash  MCXN947 load debug/zephyr/zephyr_stripped_v1_signed.bin:0x10000
debug/zephyr/wolfboot.bin:0x10000000
```

## 4. Run Demonstrator<a name="step4"></a>
Please connect to the Serial Output of the FRDM-MCXN947 via:
    - Serial monitor - `/dev/tty/YourMCXN-Port`
    - Screen Command - `screen /dev/tty"MCXN-Port 115200"`
    - Some Serial Terminal you are familiar with 

Push reset button on the FRDM-MCXN947 board and view the startup message. Note the IP address and MQTT subscriptions.
```
Boot partition: 0x10000 (sz 300508, ver 0x1, type 0x601)
Update partition: 0xD0000 (sz 300508, ver 0x2, type 0x601)
Boot partition: 0x10000 (sz 300508, ver 0x1, type 0x601)
Booting version: 0x1
*** Booting Zephyr OS build v4.0.0-5-gfd6b8b33303b ***

Running wolfSSL example from the frdm_mcxn947!
IP Address is: 192.168.1.70
Running wolfMQTT firmware client for OTA
MQTT Firmware Client: QoS 2, Use TLS 1
MQTT Net Init: Success (0)
MQTT Init: Success (0)
NetConnect: Host 192.168.1.10, Port 8883, Timeout 5000 ms, Use TLS 1
MQTT TLS Setup (1)
MQTT TLS Verify Callback for fwclient: PreVerify 0, Error -188 (no support for error strings built in)
  Subject's domain name is MyMosquittoCA
  Allowing cert anyways
MQTT Socket Connect: Success (0)
MQTT Connect: Proto (v3.1.1), Success (0)
MQTT Connect Ack: Return Code 0, Session Present 0
[00MQTT Subscribe: Success (0)
  Topic wolfMQTT/example/command, Qos 2, Return Code 2
MQTT Subscribing to firmware topic...
:0MQTT Subscribe: Success (0)
  Topic wolfMQTT/example/firmware, Qos 2, Return Code 2
MQTT Waiting for message...
```
Open another terminal and run fwserver:
```
cd fwserver/build
fwserver -t
```
The serial output shows the logs during OTA.
After the new image is downloaded, you'll see the messages:
```
Firmware transfer complete: 301532 bytes
MQTT Exiting...
MQTT Disconnect: Success (0)
MQTT Socket Disconnect: Success (0)
Firmware client completed successfully!
Triggering wolfBoot update
```
Push the reset button again.
wolfBoot detects the new image in the update partition and verifies it.
Then, wolfBoot will swap the downloaded image from update partition to boot partition.
```
...
Copy sector 33 (part 0->1)
Copy sector 33 (part 2->0)
Copy sector 34 (part 1->2)
Copy sector 34 (part 0->1)
Copy sector 34 (part 2->0)
Copy sector 35 (part 1->2)
Copy sector 35 (part 0->1)
Copy sector 35 (part 2->0)
Copy sector 36 (part 1->2)
Copy sector 36 (part 0->1)
Copy sector 36 (part 2->0)
Erasing remainder of partition (57 sectors)...
Boot partition: 0x10000 (sz 300508, ver 0x2, type 0x601)
Update partition: 0xD0000 (sz 300508, ver 0x1, type 0x601)
Copy sector 93 (part 0->2)
Copied boot sector to swap
Boot partition: 0x10000 (sz 300508, ver 0x2, type 0x601)
Booting version: 0x2
```

## 5. Sequence Diagram<a name="step5"></a>
### Overview
[<img src="Images/mcxn-OTA.svg">](Images/mcxn-OTA.svg)

## 6. FAQs<a name="step6"></a>
No FAQs have been identified for this project.

## 7. Support<a name="step7"></a>

#### Project Metadata
<!----- Boards ----->
[![Board badge](https://img.shields.io/badge/Board-FRDM&ndash;MCXN947-blue)](https://github.com/search?q=org%3Anxp-appcodehub+FRDM-MCXN947+in%3Areadme&type=Repositories)

<!----- Categories ----->


<!----- Peripherals ----->
[![Peripheral badge](https://img.shields.io/badge/Peripheral-UART-yellow)](https://github.com/search?q=org%3Anxp-appcodehub+uart+in%3Areadme&type=Repositories) [![Peripheral badge](https://img.shields.io/badge/Peripheral-ETHERNET-yellow)](https://github.com/search?q=org%3Anxp-appcodehub+ethernet+in%3Areadme&type=Repositories)

<!----- Toolchains ----->
[![Toolchain badge](https://img.shields.io/badge/Toolchain-VS%20CODE-orange)](https://github.com/search?q=org%3Anxp-appcodehub+vscode+in%3Areadme&type=Repositories)

Questions regarding the content/correctness of this example can be entered as Issues within this GitHub repository.

>**Warning**: For more general technical questions regarding NXP Microcontrollers and the difference in expected functionality, enter your questions on the [NXP Community Forum](https://community.nxp.com/)



## 8. Release Notes<a name="step8"></a>
| Version | Description / Update                           | Date                        |
|:-------:|------------------------------------------------|----------------------------:|
| 1.0     | Initial release on Application Code Hub        | April 23rd 2026|
