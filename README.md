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
The router itself:<br/>
<img width="666" height="636" alt="Image" src="https://github.com/user-attachments/assets/947c8402-2443-4f2b-97de-15f3d9165906" /><br/><br/>
<img width="699" height="534" alt="Image" src="https://github.com/user-attachments/assets/df91d64f-73dd-4f33-8152-61b5bd4c7fb0" /><br/><br/>
<img width="694" height="881" alt="Image" src="https://github.com/user-attachments/assets/df011122-16f3-461e-b2ba-dcc6ae234405" /><br/><br/>
Router is powered on in the image below:<br/><br/>
<img width="701" height="872" alt="Image" src="https://github.com/user-attachments/assets/9479689f-05f8-4e1f-9d22-98d7235e9f57" /><br/>

Attempt at connecting to the router using iPhone 12 Pro Max:<br/><br/>
<img width="426" height="655" alt="Image" src="https://github.com/user-attachments/assets/f56ab3ee-ae1f-41ad-bd5d-3b9c3c1426e8" /><br/>

## Service enumeration
To connect to the router, an RJ45 (ethernet) cable was employed. Running a simple ```ifconfig``` will show the IP address that the hsot device has received from the DHCP client running on the router. Easy enough to figure out what the router IP address is ```192.168.1.1```. NetDiscover / Nmap could also be used to identify any other machines on the same network.<br/><br/>
<img width="644" height="329" alt="Image" src="https://github.com/user-attachments/assets/7788cd39-c6fd-4d14-bcda-d15dedefe35e" /><br/><br/>

An Nmap scan was conudcted to identify what services are running on the router.<br/>
``` nmap -sV -p- 192.168.1.1 -vv -oN <filename.txt> ```<br/><br/>
<img width="1893" height="439" alt="Image" src="https://github.com/user-attachments/assets/932e17dc-f0da-4c4e-a326-199197d8517f" /><br/><br/>

As identified in the scan, several ports are open on the router, including:<br/>
1. Port 23 - Telnet
2. Port 80 - Router running a web server
3. Port 1900 - Portable SDK for uPnP devices
4. Port 7547 - TP-Link remote access?
5. Port 20001 - Dropbear sshd<br/>

Port 23 - Telnet<br/>
From the Nmap scan, Busybox was detected as being used on the Telnet service port, specifically a custom built 'TP-LINK router telnetd'.<br/><br/>
<img width="345" height="165" alt="Image" src="https://github.com/user-attachments/assets/488543e4-91bb-49b4-9937-b7d7cbaad7d0" /><br/><br/>
When trying to authenticate using 'Password123!', we receive an extremely limited Telnet connection to the router. This can be investigated further.<br/><br/>
<img width="518" height="389" alt="Image" src="https://github.com/user-attachments/assets/7555f55c-8454-4f5b-b0f0-45dd867bba5c" /><br/><br/>


Port 80 - Web GUI:<br/>
A web server is running on the router, asking to create a password for the web console. For investigatory purposes, a super-secure password was used - Password123!<br/><br/>
<img width="1919" height="1042" alt="Image" src="https://github.com/user-attachments/assets/583c29ee-c7fb-4f22-8a1e-927aab893d5c" /><br/><br/>

Port 20001 - Dropbear SSH
When trying to SSH to the router, the following error is presented:<br/><br/>
<img width="902" height="68" alt="Image" src="https://github.com/user-attachments/assets/4ff016ee-63b6-4f8e-8530-3638b3fe36bf" /><br/><br/>
This conveys that the SSH that the router is offering is using older algorithms (ssh-rsa and ssh-dss). For security purposes, the later SSH clients refusre older algorithms by default. This is also a pretty common adversary technique called downgrade attack.<br/>
To overcome this, specify the following:<br/>
```ssh -p 20001 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedKeyTypes=+ssh-rsa admin@192.168.1.1``` or ssh-dss.<br/>
Regardless, this didn't work, with the right password (the password is the one that was created when accessing the admin portal on Port 80):<br/><br/>
<img width="794" height="126" alt="Image" src="https://github.com/user-attachments/assets/8a2f3154-4965-40f6-a991-6472331182a3" /><br/>

Although there are likely security concerns that can be taken advantage of on this router, the main focus of this project is to try to get a shell from UART, or potentially dumping the firmware from the router itself.

## Identifying firmware from vendor's website (easiest route)
The latest firmware for the router can be downloaded from the Official Mercusys website.<br/><br/>
<img width="1296" height="975" alt="Image" src="https://github.com/user-attachments/assets/a965a93f-b336-4eb4-963f-de0c9aa8040d" />


## Extracting firmware
Once the firmware was downloaded, it just need to be unpacked.
``` unzip <filename>.zip ```
<img width="1148" height="259" alt="Image" src="https://github.com/user-attachments/assets/99460d6d-ce3d-4023-9d8f-2c5b74dda682" />

An issue arose when trying to extract the file system from the binary file. To resolve the issue, append:```-run-as=root```.<br/>
<img width="1898" height="722" alt="Image" src="https://github.com/user-attachments/assets/90b15b7c-b6ae-4768-b8f2-3ceec28116d5" /><br/>
Fortunately, the firmware was not encrypted. To test for this, use ```binwalk -E <firmware file>```.<br/>

Accessing the file system, squash-fs (file system) was notable. Upon accessing the fs, there were a couple of files of interest: passwd and passwd.bak.<br/>
<img width="1832" height="636" alt="Image" src="https://github.com/user-attachments/assets/6305adfb-8018-447d-8fee-7313ca70e282" /><br/>
Unsure why the admin hash is in a backup file - maybe something that the devs had forgotten about? We'll never know.

## Password cracking
After reading the /etc/passwd.bak file, we can see that there is an admin account. The hash can be copied for offline brute-force techniques.
```admin:$1$$iC.dUsGpxNNJGeOm1dFio/:0:0:root:/:/bin/sh```

Employed the use of hash-id (https://github.com/psypanda/hashID) to identify the most probable hashing algorithm.

<img width="432" height="229" alt="Image" src="https://github.com/user-attachments/assets/f5ef6e2b-e67b-465d-8182-1effebc84198" />

Once identified, hashcat was utilised to crack the passwod, using the common wordlist, rockyou.txt.

<img width="1055" height="660" alt="Image" src="https://github.com/user-attachments/assets/1bfbf5bd-ec19-4024-b38b-0fdf11684a3e" />

The hash was cracked and the credentials are
``` admin ```:``` 1234 ```

This seems to be a common default credential set from TP-Link (references to TP-Link on product box and internal directories.

## Disassembling the router
The two screws on the back hold the top part of the enclosure (black piece) on.
<img width="698" height="869" alt="Image" src="https://github.com/user-attachments/assets/4aae7b85-8b19-4192-a80e-ef56bff0dce0" /><br/>

A card was used to open the body of the router, revealing the internals of the router.
<img width="1230" height="917" alt="Image" src="https://github.com/user-attachments/assets/84fb7ff4-f1bc-4b20-8f5d-f0715e2b10d9" /><br/>

## Identifying the UART connectors
To identify each UART connector, a multimeter is employed. We can set it to the following to conduct a continuity test to identify the ground. The SIM card casing was used for the multimeter ground.<br/><br/>
<img width="682" height="908" alt="Image" src="https://github.com/user-attachments/assets/a8504a93-88c4-4f75-8462-87024e893927" /><br/>

<img width="687" height="903" alt="Image" src="https://github.com/user-attachments/assets/91b6cb5d-6fe3-4938-93e8-d16b4b27e61a" /><br/>

Measure continuity to find Ground<br/>
GND - Will hear a beeping noise.<br/>
RX - Should have a voltage reading of near 0.<br/>
TX - Voltage should be fluctuating.<br/>
PWR - Expect around 3.3V when device is plugged in.<br/>
Full duplex UART needs a minimum of 3 contacts, TX, RX, and GND.<br/>

If anyone follows this as a guide, don't make the rookie mistake that I did and connect RX to RX and TX to TX as you won't receive an output. The connections are inverted when connecting to the UART controller.

<img width="1219" height="911" alt="Image" src="https://github.com/user-attachments/assets/41362e61-3f7b-4052-82d9-74df6f239480" /><br/>

## Soldering to UART
As you can see, definitely not an expert in soldering. Managed to solder the pins and they were _relatively_ robust.<br/>
<img width="689" height="909" alt="Image" src="https://github.com/user-attachments/assets/852054c4-0734-406e-8fd4-c44268d0bd83" /><br/>

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

## Troubleshooting, issues, and learning outcomes
As this was my first time using a soldering iron, some rookie mistakes were made:</br>
1. Not inverting the UART port connections<br/>
2. The voltage on the UART controller was set to 3V. After no display of data, the jumper on the UART controller was removed, which had then created another issue. The output in minicom was somewhat readble, but not great. The input was not being parsed correctly either. After hours of troubleshooting, it turns out that the voltage was not stable on the UART controller, interrupting / disrupting transmission. To fix this, a jumper was put on the 5V pins on the UART controller, and fixed the issue.<br/>
3. Perfectionist tendencies made me re-solder the UART pins many times - still didn't come out too great, but definitely improved.
 
# Conclusion

From the service enumeration, it was obvious that security implementation was lacking.<br/>
Several open ports - do they need to be open? Further investigation needed.<br/>
No credential set for Telnet. You can submit a '(new password)' to be used as the default credential set for Telnet. If the device is not properly configured, anyone connected to the router can utilise the Telnet service, set a password, and have full control of the device.<br/>
