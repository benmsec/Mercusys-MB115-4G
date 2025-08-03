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
``` admin ```
``` 1234 ```

This seems to be a common default credential set from TP-Link (references to TP-Link on product box and internal directories.

## Disassembling the router

## Identifying the UART connectors

## Soldering to UART

## 

## Troubleshooting issues

# Conclusion

From the service enumeration, it was obvious that security implementation was lacking.
Several open ports - do they need to be open? Will look into this.
No credential set for Telnet. You can submit a '(new password)' to be used as the default credential set for Telnet. If the device is not properly configured, anyone connected to the router can utilise the Telnet service, set a password, and have full control of the device.
