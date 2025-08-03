# Mercusys-MB115-4G
Mercusys MB115-4G

This document provides technical details on the reverse engineering process of the Mercusys MB115-4G router.

## Components

The router consists of the following components:
| Name         | Component           | Description                                               |
| ------------ | -------------       | --------------------------------------------------------- | 
| SoC          | MEDIATEK MT7628NN   | CPU: 580MHz /w 64kb                                       | 
| RAM          | ESMT M14D5121632A   | 65536kb / 64 MiB @ 200 MHz                                |
| Serial Flash | EEC SH16T01N0       |                                                           |
| Modem module | QUECTEL EC-200-EL   |                                                           | 

## Tools used
| Name                       | Description         |
| -------------------------- | ------------------- |
| Digital Multimeter         | Checking UART pins for continuity & voltage to identify TX, RX, and GND |
| FT232 USB-C to UART Module | Converts USB to UART for serial comms                                   |
| Soldering iron kit         | Soldering pins to UART                                                  |
| Anti-static mat            | N/A                                                                     |
| RJ45 (additional)          | Service enumeration on the device                                       |

# Mercusys MB115-4G
## Brand new product
Body of router:<br/>
<img width="666" height="636" alt="Image" src="https://github.com/user-attachments/assets/947c8402-2443-4f2b-97de-15f3d9165906" /><br/><br/>
<img width="699" height="534" alt="Image" src="https://github.com/user-attachments/assets/df91d64f-73dd-4f33-8152-61b5bd4c7fb0" /><br/><br/>
<img width="694" height="881" alt="Image" src="https://github.com/user-attachments/assets/df011122-16f3-461e-b2ba-dcc6ae234405" /><br/><br/>
Router powered on.<br/><br/>
<img width="701" height="872" alt="Image" src="https://github.com/user-attachments/assets/9479689f-05f8-4e1f-9d22-98d7235e9f57" /><br/>

<br/>Attempt at connecting to the router using iPhone 12 Pro Max:<br/><br/>
<img width="426" height="655" alt="Image" src="https://github.com/user-attachments/assets/f56ab3ee-ae1f-41ad-bd5d-3b9c3c1426e8" /><br/>

## Service enumeration

``` nmap -A -p- 192.168.1.1 -vv -oX <filename.txt> ```
<Insert image>

Port 80 - Web GUI:
<img width="1919" height="1042" alt="Image" src="https://github.com/user-attachments/assets/583c29ee-c7fb-4f22-8a1e-927aab893d5c" />

## Identifying firmware from vendor's website (easiest route)

<img width="1296" height="975" alt="Image" src="https://github.com/user-attachments/assets/a965a93f-b336-4eb4-963f-de0c9aa8040d" />


## Extracting firmware
Once the firmware was downloaded, it just need to be unpacked.
``` unzip <filename>.zip ```
<img width="1148" height="259" alt="Image" src="https://github.com/user-attachments/assets/99460d6d-ce3d-4023-9d8f-2c5b74dda682" />

I experienced a couple of issues when trying to extract the filesystem from the binary file. In my case, I appended ```-run-as=root```.
<img width="1898" height="722" alt="Image" src="https://github.com/user-attachments/assets/90b15b7c-b6ae-4768-b8f2-3ceec28116d5" />

Accessing the file system, squash-fs (file system) was notable. Upon accessing the fs, there were a couple of files of interest: passwd and passwd.bak.
<img width="1832" height="636" alt="Image" src="https://github.com/user-attachments/assets/6305adfb-8018-447d-8fee-7313ca70e282" />
Unsure why the admin hash is in a backup file - maybe something that the devs had forgotten about? We'll never know.

## Password cracking
After reading the /etc/passwd.bak file, we can see that there is an admin account. The hash can be copied for offline brute-force techniques.
```admin:$1$$iC.dUsGpxNNJGeOm1dFio/:0:0:root:/:/bin/sh```

Employed the use of hash-id (https://github.com/psypanda/hashID) to identify the most probable hashing algorithm.

<img width="432" height="229" alt="Image" src="https://github.com/user-attachments/assets/f5ef6e2b-e67b-465d-8182-1effebc84198" />

Once identified, hashcat was utilised to crack the passwod, using the common wordlist, rockyou.txt.

<img width="1055" height="660" alt="Image" src="https://github.com/user-attachments/assets/1bfbf5bd-ec19-4024-b38b-0fdf11684a3e" />

The hash was cracked and the credentials are
``` admin ```<br/>
``` 1234 ```

This seems to be a common default credential set from TP-Link (references to TP-Link on product box and internal directories.

## Disassembling the router
The two screws on the back hold the top part of the enclosure (black piece) on.
<img width="698" height="869" alt="Image" src="https://github.com/user-attachments/assets/4aae7b85-8b19-4192-a80e-ef56bff0dce0" /><br/>

A card was used to open the body of the router, revealing the internals of the router.
<img width="1230" height="917" alt="Image" src="https://github.com/user-attachments/assets/84fb7ff4-f1bc-4b20-8f5d-f0715e2b10d9" /><br/>

## Identifying the UART connectors
<img width="682" height="908" alt="Image" src="https://github.com/user-attachments/assets/a8504a93-88c4-4f75-8462-87024e893927" /><br/><br/>
<img width="687" height="903" alt="Image" src="https://github.com/user-attachments/assets/91b6cb5d-6fe3-4938-93e8-d16b4b27e61a" /><br/><br/>

Measure continuity to find Ground<br/>
GND - Will hear a beeping noise.<br/>
RX - Should have a voltage reading of near 0.<br/>
TX - Voltage should be fluctuating.<br/>
PWR - Expect around 3.3V when device is plugged in.<br/>
Full duplex UART needs a minimum of 3 contacts, TX, RX, and GND.<br/>

If anyone ever follows this as a guide, don't make the rookie mistake that I did and connect RX to RX and TX to TX as you won't receive an output. The roles are inversed when connecting to UART reader.

<img width="1219" height="911" alt="Image" src="https://github.com/user-attachments/assets/41362e61-3f7b-4052-82d9-74df6f239480" /><br/><br/>

## Soldering to UART
As you can see, definitely not an expert in soldering. Managed to solder the pins and they were _relatively_ robust <br/>
<img width="689" height="909" alt="Image" src="https://github.com/user-attachments/assets/852054c4-0734-406e-8fd4-c44268d0bd83" /><br/><br/>

## Reading the data
In comes Minicom (https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom) which is a serial communication tool. You could also use screen.<br/>
Doing some research online, and watching Andrew Bellini's DefCon talk (https://www.youtube.com/watch?v=YPcOwKtRuDQ&t - very informative by the way), we run into something called UART baud rates.<br/>
Just think of baud rates as the speed of communication in UART, essentially how many bits are sent per second between devices. Without specifying the correct baud rate, we're not going to see any useful data.<br/><br/>

A great analogy of this is having a conversation between two people. Baud rate is how fast they talk. If they talk too fast, words get jumbled up, so the pace has to be agreed.<br/>

```minicom -s``` to enter the setup of minicom<br/>
Select 'Serial port setup' and configure it to be the right device and correct baud rate.<br/>
The most common baud rate for IoT devices is 115200. Others could be 9600, 57600, 38400, and 74880. Without a logic analyser, it's a guessing game. For this project, a logic analyser wasn't utilised.<br/>
Set the serial device to your UART port (mine was /dev/ttyUSB0).<br/>
Or just do ``` minicom -b 115200 -D /dev/ttyUSB0 ```.<br/>

<img width="346" height="194" alt="Image" src="https://github.com/user-attachments/assets/e12255fe-6771-445c-b760-83fdbfed4d55" /><br/>

At this point, the program is waiting to receive transmission from UART. Powering on the router will show the following:<br/>
<img width="1006" height="956" alt="Image" src="https://github.com/user-attachments/assets/43e7e013-01b6-4085-8e14-0ce06b24b923" /><br/>

Enter in the credentials that were discovered previously. This grants root access to the router.<br/>
<img width="692" height="533" alt="Image" src="https://github.com/user-attachments/assets/059afcf9-4710-4058-b479-4d092f682f76" /><br/>
Note: 'SIM Response Error!' is present as there is no SIM card inserted into the router.<br/>

## Troubleshooting issues
As this was my first time doing a reverse engineering project, some rookie mistakes were made:</br>
1. Not inverting the UART port connections<br/>
2. The voltage on the UART controller was set to 3V. After no display of data, the jumper on the UART controller was removed, which had then created another issue. The output in minicom was somewhat readble, but not great. The input was not being parsed correctly either. After hours of troubleshooting, it turns out that the voltage was not stable on the UART controller, interrupting / disrupting transmission. To fix this, a jumper was put on the 5V pins on the UART controller, and fixed the issue.<br/>
# Conclusion

From the service enumeration, it was obvious that security implementation was lacking.
Several open ports - do they need to be open? Will look into this.
No credential set for Telnet. You can submit a '(new password)' to be used as the default credential set for Telnet. If the device is not properly configured, anyone connected to the router can utilise the Telnet service, set a password, and have full control of the device.
