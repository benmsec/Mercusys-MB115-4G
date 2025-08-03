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
![Image](https://github.com/user-attachments/assets/06cf097b-bf58-4fa4-b214-a03c4ced21c3)

![Image](https://github.com/user-attachments/assets/45b4a05a-528e-4c2e-994b-5a232f79900c)

![Image](https://github.com/user-attachments/assets/b3fa7a08-16af-4e70-8c0e-01fd35d46fca)

![Image](https://github.com/user-attachments/assets/eb528980-f930-4f35-9088-c4780d70c7d1)

![Image](https://github.com/user-attachments/assets/b611401b-e3eb-4608-ab31-8dfa3f5a1e3f)

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

![Image](https://github.com/user-attachments/assets/75581df3-02f9-4a70-8b1e-9533aa746c7f)

![Image](https://github.com/user-attachments/assets/29744bc5-d481-4a03-809d-a7d354aebc81)


## Identifying the UART connectors
![Image](https://github.com/user-attachments/assets/c8a2f953-64bd-4753-8fb6-c7c9dad342bb)
![Image](https://github.com/user-attachments/assets/030ff52a-d855-4ab9-b98a-a2e75c9e3a90)

Measure continuity to find Ground<br/>
GND - Will hear a beeping noise.<br/>
RX - Should have a voltage reading of near 0.<br/>
TX - Voltage should be fluctuating.<br/>
PWR - Expect around 3.3V when device is plugged in.<br/>
Full duplex UART needs a minimum of 3 contacts, TX, RX, and GND.<br/>

If anyone ever follows this as a guide, don't make the rookie mistake that I did and connect RX to RX and TX to TX as you won't receive an output. The roles are inversed when connecting to UART reader.

![Image](https://github.com/user-attachments/assets/696c2871-b619-46b6-9fca-1a3efab870e2)

## Soldering to UART
As you can see, definitely not an expert in soldering. Managed to solder the pins and they were _relatively_ robust <br/>
![Image](https://github.com/user-attachments/assets/ef26ffc6-f11e-430f-969d-43625578851b)

## Reading the data
In comes Minicom (https://wiki.emacinc.com/wiki/Getting_Started_With_Minicom) which is a serial communication tool. You could also use screen.
Doing some research online, and watching Andrew Bellini's DefCon talk (https://www.youtube.com/watch?v=YPcOwKtRuDQ&t - very informative by the way), we run into something called UART baud rates.<br/>
Just think of baud rates as the speed of communication in UART, essentially how many bits are sent per second between devices. Without specifying the correct baud rate, we're not going to see any useful data.<br/><br/>

A great analogy of this is having a conversation between two people. Baud rate is how fast they talk. If they talk too fast, words get jumbled up.<br/>

```minicom -s```<br/>
Select 'Serial port setup' and configure it to be the right device and correct baud rate.<br/>
The most common baud rate for IoT devices is 115200. Others could be 9600, 57600, 38400, and 74880. Without a logic analyser, it's a guessing game. For this project, a logic analyser wasn't utilised.<br/>
Set the serial device to your UART port (mine was /dev/ttyUSB0).<br/>
Or just do ``` minicom -b 115200 -D /dev/ttyUSB0 ``` and it should work.

## Troubleshooting issues

# Conclusion

From the service enumeration, it was obvious that security implementation was lacking.
Several open ports - do they need to be open? Will look into this.
No credential set for Telnet. You can submit a '(new password)' to be used as the default credential set for Telnet. If the device is not properly configured, anyone connected to the router can utilise the Telnet service, set a password, and have full control of the device.
