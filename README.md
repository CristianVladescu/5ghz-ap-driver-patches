# 5ghz-ap-driver-patches
Driver patches to fix 5GHz AP mode on some Wi-Fi network adapters.  
This is an adaptation for Debian based distros of these Arch Linux patches https://github.com/twisteroidambassador/arch-linux-ath-user-regd and https://github.com/CodePhase/patch-atheros-regdom .

##Issues
### World rergulatory domain issue
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
#### Solution
There are 2 ways to solve this with driver patches:
 - for kernel 5.10 `ath_country.patch` will ignore, in driver, the value for wireless regulatory domain from EEPROM, and use the country specified in this patch (edit the country before applying it, `sed -i 's/<CountryCode to use>/CTRY_ROMANIA/' 5ghz-ap-driver-patch/ath_country.patch`, and instead of `CTRY_ROMANIA` use yours from [this list](https://github.com/torvalds/linux/blob/c69cf88cda5faca0e411babb67ac0d8bfd8b4646/drivers/net/wireless/ath/regd.h#L61))
 - for kernel 5.13 (and 5.10) `ath_etsi_regd.patch` will define and apply the wireless regulatory domain for ETSI region (I have not included other regions, but you can use this as an example)

### QCA9984 not initializing on kernel 5.13 due to IRAM access issue
For QCA9984, apply `ath10k_iram.patch` to workaround this issue:
```
root@debian:~# dmesg | grep "failed to copy target iram"
[    9.043246] ath10k_pci 0000:00:10.0: failed to copy target iram contents: -12
```
### Intel `no IR` issue
`iwlwifi` driver used for Intel adapters, blatantly disables AP mode. At the moment, I could not find a complete solution, but I was able to remove to `no IR` flag by applying `iwlwifi_no_ir.patch`. However, when AP is started on 5GHz, the firmware/microcode is crashing. Tested Intel® Wireless-AC 7265/8260/9260 with kernel 5.13.19, 5.19.11, and all crash.
```
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Microcode SW error detected. Restarting 0x0.
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Start IWL Error Log Dump:
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Status: 0x00000040, count: 6
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Loaded firmware version: 46.6f9f215c.0 9260-th-b0-jf-b0-46.ucode
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x000014FC | ADVANCED_SYSASSERT          
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x000022F0 | trm_hw_status0
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | trm_hw_status1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00481532 | branchlink2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x004715DE | interruptlink1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | interruptlink2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000024 | data1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000F61 | data2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00090010 | data3
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | beacon time
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0050796D | tsf low
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | tsf hi
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | time gp1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0050797C | time gp2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000001 | uCode revision type
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0000002E | uCode version major
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x6F9F215C | uCode version minor
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000321 | hw version
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x18C89004 | board version
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x8026F402 | hcmd
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00022000 | isr0
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | isr1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x08201802 | isr2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00400080 | isr3
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | isr4
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x002801D1 | last cmd Id
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0001AF02 | wait_event
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | l2p_control
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | l2p_duration
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | l2p_mhvalid
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | l2p_addr_match
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0000008F | lmpm_pmg_sel
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x28010826 | timestamp
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0000284C | flow_handler
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Start IWL Error Log Dump:
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Status: 0x00000040, count: 7
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x20000070 | NMI_INTERRUPT_LMAC_FATAL
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | umac branchlink1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xC0088BEE | umac branchlink2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xC0084484 | umac interruptlink1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xC0084484 | umac interruptlink2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000800 | umac data1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xC0084484 | umac data2
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xDEADBEEF | umac data3
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0000002E | umac major
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x6F9F215C | umac minor
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0050798E | frame pointer
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xC088627C | stack pointer
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0029012B | last host cmd
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | isr status reg
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: IML/ROM dump:
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | IML/ROM error/state
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000003 | IML/ROM data1
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Fseq Registers:
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00540280 | FSEQ_ERROR_CODE
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x00000000 | FSEQ_TOP_INIT_VERSION
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x85B98B3B | FSEQ_CNVIO_INIT_VERSION
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0000A371 | FSEQ_OTP_VERSION
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xFACFE797 | FSEQ_TOP_CONTENT_VERSION
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xE6201AF9 | FSEQ_ALIVE_TOKEN
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0xF6E5B2BB | FSEQ_CNVI_ID
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x12D0E108 | FSEQ_CNVR_ID
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x01000200 | CNVI_AUX_MISC_CHIP
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x01300202 | CNVR_AUX_MISC_CHIP
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0000485B | CNVR_SCU_SD_REGS_SD_REG_DIG_DCDC_VTRIM
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: 0x0BADCAFE | CNVR_SCU_SD_REGS_SD_REG_ACTIVE_VDIG_MIRROR
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: WRT: Collecting data: ini trigger 4 fired (delay=0ms).
Oct 05 12:48:12 debian kernel: ieee80211 phy0: Hardware restart was requested
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: FW error in SYNC CMD BINDING_CONTEXT_CMD
Oct 05 12:48:12 debian kernel: CPU: 14 PID: 3776 Comm: hostapd Tainted: G            E     5.13.19 #1
Oct 05 12:48:12 debian kernel: Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.14.0-0-g155821a1990b-prebuilt.qemu.org 04/01/2014
Oct 05 12:48:12 debian kernel: Call Trace:
Oct 05 12:48:12 debian kernel:  dump_stack+0x76/0x94
Oct 05 12:48:12 debian kernel:  iwl_trans_txq_send_hcmd+0x385/0x390 [iwlwifi]
Oct 05 12:48:12 debian kernel:  ? finish_wait+0x80/0x80
Oct 05 12:48:12 debian kernel:  iwl_trans_send_cmd+0x84/0xe0 [iwlwifi]
Oct 05 12:48:12 debian kernel:  iwl_mvm_send_cmd_status+0x2a/0xa0 [iwlmvm]
Oct 05 12:48:12 debian kernel:  iwl_mvm_send_cmd_pdu_status+0x4b/0x70 [iwlmvm]
Oct 05 12:48:12 debian kernel:  iwl_mvm_binding_update+0x142/0x240 [iwlmvm]
Oct 05 12:48:12 debian kernel:  iwl_mvm_start_ap_ibss+0x99/0x280 [iwlmvm]
Oct 05 12:48:12 debian kernel:  ? __cond_resched+0x16/0x40
Oct 05 12:48:12 debian kernel:  ieee80211_start_ap+0x507/0x740 [mac80211]
Oct 05 12:48:12 debian kernel:  nl80211_start_ap+0x823/0xae0 [cfg80211]
Oct 05 12:48:12 debian kernel:  ? lock_timer_base+0x61/0x80
Oct 05 12:48:12 debian kernel:  genl_family_rcv_msg_doit+0xea/0x150
Oct 05 12:48:12 debian kernel:  genl_rcv_msg+0xde/0x1d0
Oct 05 12:48:12 debian kernel:  ? nl80211_join_ibss+0x480/0x480 [cfg80211]
Oct 05 12:48:12 debian kernel:  ? genl_get_cmd+0xd0/0xd0
Oct 05 12:48:12 debian kernel:  netlink_rcv_skb+0x50/0xf0
Oct 05 12:48:12 debian kernel:  genl_rcv+0x24/0x40
Oct 05 12:48:12 debian kernel:  netlink_unicast+0x201/0x2c0
Oct 05 12:48:12 debian kernel:  netlink_sendmsg+0x243/0x480
Oct 05 12:48:12 debian kernel:  sock_sendmsg+0x5e/0x60
Oct 05 12:48:12 debian kernel:  ____sys_sendmsg+0x22e/0x270
Oct 05 12:48:12 debian kernel:  ? import_iovec+0x17/0x20
Oct 05 12:48:12 debian kernel:  ? sendmsg_copy_msghdr+0x7c/0xa0
Oct 05 12:48:12 debian kernel:  ? __check_object_size+0x46/0x150
Oct 05 12:48:12 debian kernel:  ___sys_sendmsg+0x75/0xb0
Oct 05 12:48:12 debian kernel:  ? ___sys_recvmsg+0x8e/0x100
Oct 05 12:48:12 debian kernel:  ? __wake_up_common_lock+0x8a/0xc0
Oct 05 12:48:12 debian kernel:  ? __check_object_size+0x46/0x150
Oct 05 12:48:12 debian kernel:  ? __cond_resched+0x16/0x40
Oct 05 12:48:12 debian kernel:  ? release_sock+0x19/0x90
Oct 05 12:48:12 debian kernel:  ? _raw_spin_unlock_bh+0xa/0x20
Oct 05 12:48:12 debian kernel:  ? sock_setsockopt+0xff/0xfa0
Oct 05 12:48:12 debian kernel:  __sys_sendmsg+0x59/0xa0
Oct 05 12:48:12 debian kernel:  do_syscall_64+0x40/0x80
Oct 05 12:48:12 debian kernel:  entry_SYSCALL_64_after_hwframe+0x44/0xae
Oct 05 12:48:12 debian kernel: RIP: 0033:0x7ff9e475ffc3
Oct 05 12:48:12 debian kernel: Code: 64 89 02 48 c7 c0 ff ff ff ff eb b7 66 2e 0f 1f 84 00 00 00 00 00 90 64 8b 04 25 18 00 00 00 85 c0 75 14 b8 2e 00 00 00 0f 05 <48> 3d 00 f0 ff ff 77 55 c3 0f 1f 40 00 48 83 ec 28 89 54 24 1c 48
Oct 05 12:48:12 debian kernel: RSP: 002b:00007ffc0174a2e8 EFLAGS: 00000246 ORIG_RAX: 000000000000002e
Oct 05 12:48:12 debian kernel: RAX: ffffffffffffffda RBX: 000055aa1dade650 RCX: 00007ff9e475ffc3
Oct 05 12:48:12 debian kernel: RDX: 0000000000000000 RSI: 00007ffc0174a320 RDI: 0000000000000004
Oct 05 12:48:12 debian kernel: RBP: 000055aa1daeaae0 R08: 0000000000000004 R09: 00007ff9e4831be0
Oct 05 12:48:12 debian kernel: R10: 00007ffc0174a3f4 R11: 0000000000000246 R12: 000055aa1dade560
Oct 05 12:48:12 debian kernel: R13: 00007ffc0174a320 R14: 00007ffc0174a3f4 R15: 000055aa1daeab30
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Failed to send binding (action:1): -5
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: Failed to remove MAC context: -5
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: PHY ctxt cmd error. ret=-5
Oct 05 12:48:12 debian kernel: iwlwifi 0000:00:10.0: iwl_trans_wait_tx_queues_empty bad state = 0
```

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
apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev
```
Clone this repo `git clone https://github.com/CristianVladescu/5ghz-ap-driver-patches`.
Compile ath module:
```
cd linux-5.10.140
cp -v /boot/config-$(uname -r) .config
make oldconfig
patch -p1 < ../5ghz-ap-driver-patches/<patch needed>
# either compile the whole kernel (instead of -j8 use the number of cores your CPU has)
time make -j8 modules
# or, you can compile only the driver using the current kernel source headers (if available)
apt install linux-headers-$(uname -r)
pushd drivers/net/wireless/ath
make -C /lib/modules/`uname -r`/build M=$PWD
popd
# then copy the newly compiled driver
cp drivers/net/wireless/ath/ath.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/ath/
cp drivers/net/wireless/ath/ath10k/ath10k_core.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/ath/ath10k/
cp drivers/net/wireless/ath/ath10k/ath10k_pci.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/ath/ath10k/
cp drivers/net/wireless/intel/iwlwifi/iwlwifi.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/intel/iwlwifi/
# if you also want to install the kernel you just compiled
make modules_install
time make -j8
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
echo 0000\:00\:10.0 > /sys/bus/pci/drivers/iwlwifi/unbind
echo 1 > /sys/bus/pci/devices/0000\:00\:10.0/remove
rmmod ath10k_pci ath10k_core ath
insmod /lib/modules/$(uname -r)/kernel/drivers/net/wireless/ath/ath.ko
insmod /lib/modules/$(uname -r)/kernel/drivers/net/wireless/intel/iwlwifi/iwlwifi.ko
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
root@debian:~/linux-5.10.140# iw reg get
global
country RO: DFS-ETSI
        (2400 - 2483 @ 40), (N/A, 20), (N/A)
        (5150 - 5250 @ 80), (N/A, 23), (N/A), NO-OUTDOOR, AUTO-BW
        (5250 - 5350 @ 80), (N/A, 20), (0 ms), NO-OUTDOOR, DFS, AUTO-BW
        (5470 - 5725 @ 160), (N/A, 26), (0 ms), DFS
        (5725 - 5875 @ 80), (N/A, 13), (N/A)
        (57000 - 66000 @ 2160), (N/A, 40), (N/A)

phy#2
country RO: DFS-ETSI
        (2400 - 2483 @ 40), (N/A, 20), (N/A)
        (5150 - 5250 @ 80), (N/A, 23), (N/A), NO-OUTDOOR, AUTO-BW
        (5250 - 5350 @ 80), (N/A, 20), (0 ms), NO-OUTDOOR, DFS, AUTO-BW
        (5470 - 5725 @ 160), (N/A, 26), (0 ms), DFS
        (5725 - 5875 @ 80), (N/A, 13), (N/A)
        (57000 - 66000 @ 2160), (N/A, 40), (N/A)
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
To check what link speed clients negotiated:
```
root@debian:~# hostapd_cli all_sta
Selected interface 'wls16'
74:e5:f9:d3:2a:27
...
root@debian:~# iw wls16 station get 74:e5:f9:d3:2a:27
Station 74:e5:f9:2a:d3:89 (on wls16)
        inactive time:  908 ms
        rx bytes:       456860507
        rx packets:     298140
        tx bytes:       1114720
        tx packets:     14758
        tx retries:     1753
        tx failed:      0
        rx drop misc:   1
        signal:         -56 [-62, -62] dBm
        signal avg:     -55 [-62, -62] dBm
        tx bitrate:     866.7 MBit/s VHT-MCS 9 160MHz short GI VHT-NSS 1
        tx duration:    988247 us
        rx bitrate:     1170.0 MBit/s VHT-MCS 6 160MHz short GI VHT-NSS 2
...
```

## How to apply the patch to Proxmox VE 7
```
git clone https://git.proxmox.com/git/pve-kernel.git
git clone https://github.com/CristianVladescu/deb-ath-user-regd
cd pve-kernel
git checkout -b pve-kernel-5.13 origin/pve-kernel-5.13

git submodule update --init --recursive
or
make submodule
or
git submodule foreach git fetch --tags
git submodule update --init

cp ../5ghz-ap-driver-patches/<patch needed>.patch patches/kernel/
# or
pushd submodules/ubuntu-impish
patch -p1 < ../../../5ghz-ap-driver-patches/<patch needed>.patch
popd

apt install devscripts
mk-build-deps --install debian/control.in
make deb # no need to specify -jx, it will utilize all cores
dpkg -i *.deb
reboot
```
