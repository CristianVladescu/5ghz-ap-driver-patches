# ath-user-regd
Atheros driver patch to override the country set in the Wi-Fi card's EEPROM. This is an adaption for Debian based distros of these Arch Linux patches https://github.com/twisteroidambassador/arch-linux-ath-user-regd and https://github.com/CodePhase/patch-atheros-regdom .

## Issue
Some Atheros Wi-Fi PCIe cards (like QCA6174 that I have) come with a [global wireless regulatory domain](https://wireless.wiki.kernel.org/en/users/drivers/ath#eeprom_world_regulatory_domain) burned in EEPROM.
```
root@debian11:~# dmesg | grep ath
[23629.454420] ath: EEPROM regdomain: 0x6c <<<< WORC_WORLD
[23629.454421] ath: EEPROM indicates we should expect a direct regpair map
[23629.454422] ath: Country alpha2 being used: 00 <<<< country unspecified so broadcasting on 5GHz channels cannot be granted
[23629.454422] ath: Regpair used: 0x6c
root@debian:~# iw reg get
global
country 00: DFS-UNSET
        (755 - 928 @ 2), (N/A, 20), (N/A), PASSIVE-SCAN
        (2402 - 2472 @ 40), (N/A, 20), (N/A)
        (2457 - 2482 @ 20), (N/A, 20), (N/A), AUTO-BW, PASSIVE-SCAN
        (2474 - 2494 @ 20), (N/A, 20), (N/A), NO-OFDM, PASSIVE-SCAN
        (5170 - 5250 @ 80), (N/A, 20), (N/A), AUTO-BW, PASSIVE-SCAN
        (5250 - 5330 @ 80), (N/A, 20), (0 ms), DFS, AUTO-BW, PASSIVE-SCAN
        (5490 - 5730 @ 160), (N/A, 20), (0 ms), DFS, PASSIVE-SCAN
        (5735 - 5835 @ 80), (N/A, 20), (N/A), PASSIVE-SCAN
        (57240 - 63720 @ 2160), (N/A, 0), (N/A)
```
Because of this all 5GHz channels will be flagged with `no IR` (radiation initialization not allowed) in kernel, so they cannot be used in AP mode, only in managed/client mode.  
```
root@debian11:~# iw list | grep MHz
                        * 2412 MHz [1] (20.0 dBm)
                        * 2417 MHz [2] (20.0 dBm)
                        * 2422 MHz [3] (20.0 dBm)
                        * 2427 MHz [4] (20.0 dBm)
                        * 2432 MHz [5] (20.0 dBm)
                        * 2437 MHz [6] (20.0 dBm)
                        * 2442 MHz [7] (20.0 dBm)
                        * 2447 MHz [8] (20.0 dBm)
                        * 2452 MHz [9] (20.0 dBm)
                        * 2457 MHz [10] (20.0 dBm)
                        * 2462 MHz [11] (20.0 dBm)
                        * 2467 MHz [12] (20.0 dBm) (no IR)
                        * 2472 MHz [13] (20.0 dBm) (no IR)
                        * 2484 MHz [14] (disabled)
                        short GI (80 MHz)
                        * 5180 MHz [36] (23.0 dBm) (no IR)
                        * 5200 MHz [40] (23.0 dBm) (no IR)
                        * 5220 MHz [44] (23.0 dBm) (no IR)
                        * 5240 MHz [48] (23.0 dBm) (no IR)
                        * 5260 MHz [52] (20.0 dBm) (no IR, radar detection)
                        * 5280 MHz [56] (20.0 dBm) (no IR, radar detection)
                        * 5300 MHz [60] (20.0 dBm) (no IR, radar detection)
                        * 5320 MHz [64] (20.0 dBm) (no IR, radar detection)
                        * 5500 MHz [100] (26.0 dBm) (no IR, radar detection)
                        * 5520 MHz [104] (26.0 dBm) (no IR, radar detection)
                        * 5540 MHz [108] (26.0 dBm) (no IR, radar detection)
                        * 5560 MHz [112] (26.0 dBm) (no IR, radar detection)
                        * 5580 MHz [116] (26.0 dBm) (no IR, radar detection)
                        * 5600 MHz [120] (26.0 dBm) (no IR, radar detection)
                        * 5620 MHz [124] (26.0 dBm) (no IR, radar detection)
                        * 5640 MHz [128] (26.0 dBm) (no IR, radar detection)
                        * 5660 MHz [132] (26.0 dBm) (no IR, radar detection)
                        * 5680 MHz [136] (26.0 dBm) (no IR, radar detection)
                        * 5700 MHz [140] (26.0 dBm) (no IR, radar detection)
                        * 5720 MHz [144] (13.0 dBm) (no IR, radar detection)
                        * 5745 MHz [149] (13.0 dBm) (no IR)
                        * 5765 MHz [153] (13.0 dBm) (no IR)
                        * 5785 MHz [157] (13.0 dBm) (no IR)
                        * 5805 MHz [161] (13.0 dBm) (no IR)
                        * 5825 MHz [165] (13.0 dBm) (no IR)
                        * 5845 MHz [169] (13.0 dBm) (no IR)
                        * 5865 MHz [173] (disabled)
                 * short GI for 40 MHz
```
## Solution
There are 2 ways to solve this with driver patches:
 - for kernel 5.10 `ath_country.patch` will ignore in driver the value for wireless regulatory domain from EEPROM, and use the country specified in this patch
 - for kernel 5.13 (and 5.10) `ath_etsi_regd.patch` will define wireless regulatory domain for ETSI region

## How to apply the patch to Debian 11
First get the kernel source code and the required packages in order to compile the driver:
```
apt-get install build-essential git
apt-get build-dep linux
# this will download kernel source code in current working directory
apt-get source linux
# or if you're not using the latest kernel
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.13.19.tar.xz
unxz -v linux-5.13.19.tar.xz
tar xvf linux-5.13.19.tar
```
Clone this repo `git clone https://github.com/CristianVladescu/deb-ath-user-regd`.
If you're going to use `ath_country.patch`, edit the patch (instead of `CTRY_ROMANIA` use yours; you can find the full list [here](https://github.com/torvalds/linux/blob/c69cf88cda5faca0e411babb67ac0d8bfd8b4646/drivers/net/wireless/ath/regd.h#L61)):
```
sed -i 's/<CountryCode to use>/CTRY_ROMANIA/' deb-ath-user-regd/ath_country.patch
```
Compile ath module:
```
cd linux-5.10.140
cp -v /boot/config-$(uname -r) .config
make oldconfig
patch -p1 < ../deb-ath-user-regd/ath_country.patch
# or
patch -p1 < ../deb-ath-user-regd/ath_etsi_regd.patch
# either compile the whole kernel (instead of -j8 use the number of cores your CPU has)
time make -j8 modules
# after first compile, if you need to recompile, you can do it faster by compiling only the ath module
time make -j8 M=$(pwd)/drivers/net/wireless/ath modules
# or, you can compile only the driver using the current kernel source headers (if available)
apt isntall linux-headers-$(uname -r)
pushd drivers/net/wireless/ath
make -C /lib/modules/`uname -r`/build M=$PWD
popd
# then copy the newly compiled driver
cp drivers/net/wireless/ath/ath.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/ath/
# if you also want to install the kernel you just compiled
make modules_install
make install
update-initramfs -c -k 5.13.19
update-grub
```
Reboot or soft "replug" the PCI device:
```
# find the PCI device id
lspci
# mine is
# 00:10.0 Network controller: Qualcomm Atheros QCA6174 802.11ac Wireless Network Adapter (rev 32)
echo 0000\:00\:10.0 > /sys/bus/pci/drivers/ath10k_pci/unbind
echo 1 > /sys/bus/pci/devices/0000\:00\:10.0/remove
rmmod ath10k_pci ath10k_core ath
insmod /lib/modules/$(uname -r)/kernel/drivers/net/wireless/ath/ath.ko
echo "3" > /sys/bus/pci/rescan
```
Now 5GHz channels available in your country can be used for starting an AP.  
(radar channels need DFS enabled to work)
```
root@debian11:~/linux-5.10.140# iw list | grep MHz
                        * 2412 MHz [1] (20.0 dBm)
                        * 2417 MHz [2] (20.0 dBm)
                        * 2422 MHz [3] (20.0 dBm)
                        * 2427 MHz [4] (20.0 dBm)
                        * 2432 MHz [5] (20.0 dBm)
                        * 2437 MHz [6] (20.0 dBm)
                        * 2442 MHz [7] (20.0 dBm)
                        * 2447 MHz [8] (20.0 dBm)
                        * 2452 MHz [9] (20.0 dBm)
                        * 2457 MHz [10] (20.0 dBm)
                        * 2462 MHz [11] (20.0 dBm)
                        * 2467 MHz [12] (20.0 dBm)
                        * 2472 MHz [13] (20.0 dBm)
                        * 2484 MHz [14] (disabled)
                        short GI (80 MHz)
                        * 5180 MHz [36] (23.0 dBm)
                        * 5200 MHz [40] (23.0 dBm)
                        * 5220 MHz [44] (23.0 dBm)
                        * 5240 MHz [48] (23.0 dBm)
                        * 5260 MHz [52] (20.0 dBm) (radar detection)
                        * 5280 MHz [56] (20.0 dBm) (radar detection)
                        * 5300 MHz [60] (20.0 dBm) (radar detection)
                        * 5320 MHz [64] (20.0 dBm) (radar detection)
                        * 5500 MHz [100] (26.0 dBm) (radar detection)
                        * 5520 MHz [104] (26.0 dBm) (radar detection)
                        * 5540 MHz [108] (26.0 dBm) (radar detection)
                        * 5560 MHz [112] (26.0 dBm) (radar detection)
                        * 5580 MHz [116] (26.0 dBm) (radar detection)
                        * 5600 MHz [120] (26.0 dBm) (radar detection)
                        * 5620 MHz [124] (26.0 dBm) (radar detection)
                        * 5640 MHz [128] (26.0 dBm) (radar detection)
                        * 5660 MHz [132] (26.0 dBm) (radar detection)
                        * 5680 MHz [136] (26.0 dBm) (radar detection)
                        * 5700 MHz [140] (26.0 dBm) (radar detection)
                        * 5720 MHz [144] (13.0 dBm) (radar detection)
                        * 5745 MHz [149] (13.0 dBm)
                        * 5765 MHz [153] (13.0 dBm)
                        * 5785 MHz [157] (13.0 dBm)
                        * 5805 MHz [161] (13.0 dBm)
                        * 5825 MHz [165] (13.0 dBm)
                        * 5845 MHz [169] (13.0 dBm)
                        * 5865 MHz [173] (13.0 dBm)
                 * short GI for 40 MHz
root@debian11:~/linux-5.10.140# hostapd ../deb-ath-user-regd/hostapd.conf 
Configuration file: ../hostapd.conf
wls16: interface state UNINITIALIZED->COUNTRY_UPDATE
ACS: Automatic channel selection started, this may take a bit
wls16: interface state COUNTRY_UPDATE->ACS
wls16: ACS-STARTED 
wls16: ACS-COMPLETED freq=5805 channel=161
Using interface wls16 with hwaddr 00:0e:8e:72:fb:ac and ssid "WIFI_5G"
wls16: interface state ACS->ENABLED
wls16: AP-ENABLED 
```

## How to apply the patch to Proxmox VE 7
```
git clone https://git.proxmox.com/git/pve-kernel.git
cd pve-kernel
git checkout -b pve-kernel-5.13 origin/pve-kernel-5.13

git submodule update --init --recursive
or
make submodule
or
git submodule foreach git fetch --tags
git submodule update --init

apt install devscripts
mk-build-deps --install debian/control.in
make deb # no need to specify -jx, it will utilize all cores
dpkg -i *.deb
```
