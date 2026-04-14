# LaView-F1-LV-PWF1P-W
## Notes on making the LaView F1 LV-PWF1P-W camera work without internet connection

Over the years, I have modified various cheap Chinese IP cameras to enable RTSP and/or ONVIF, and either prevent them from phoning home or at least allow them to work without an internet connection (i.e. have the router block their outbound connections).  Most recently, I purchased a bunch of LaView F1 cameras for $5 each, and getting them to work offline proved to be more involved than with previous cameras.  I'm posting the following information publicly in case anyone else finds it useful.

## Overview
The [LaView F1](https://www.laviewusa.com/products/laview-security-cameras-4pc) appears to be a re-badged/OEM version of the [Meari Mini 12Q](https://www.meari.com/en/cooperation/productId/dd9ada55f71243ff8ccc7355002a978e42).  While LaView advertises it as a 1080P camera, the Mini 12Q actually provides a primary stream at 2560x1440 (depending on who you ask, that's also known as 2K or 1440P resolution) using H.265, with a secondary H.264 stream at 640x360.  It includes a TF/μSD card slot, microphone, speaker, and 2.4/5 GHz Wi-Fi (IEEE 802.11b/g/n/ac/ax).  Like many similar cameras, the hardware appears to be produced by Hangzhou PPStrong Technology Co., Ltd., and it includes bindings to the [Tuya](https://tuya.com/) ecosystem.  So, the bundled firmware includes [U-Boot](https://u-boot.org/), Linux, [BusyBox](https://busybox.net/), PPStrong, Meari, and Tuya software.  This is not the best IP camera I've ever owned, but it's reasonably good, and downright exceptional at the $5 price point.

There is a LaView app for configuring the camera, as well as viewing recordings or live video.  The generic Tuya app also works with this camera.  However, my goal is to completely block the camera from the internet and interface to it locally, via Frigate, VLC, Home Assistant, and other similar software.  Some cameras will do this natively, continuing to operate even if you disable internet access.  Other cameras just need a tweak here or there to do the same.  With this camera, they have taken steps to make sure you can't easily "jailbreak" it from the Tuya infrastructure.  After trying various methods of manipulating the configuration file in the JFFS2 partition, I ended up reversing the firmware and flashing a modified version.  I won't document all the missteps, failures, and dead-ends here, nor will I include any binaries that might elicit a takedown notice.  I assume the audience for this document is comfortable with a disassembler, hex editor, flash programmer, and soldering iron.

A big thanks to [Wagner (@guino)](https://github.com/guino), whose various relevant repositories provided a wealth of information.  In particular, the [methodology to dump the flash using only an SD card](https://github.com/guino/BazzDoorbell/issues/11) and a comment from [@jilleb](https://github.com/jilleb) showing a [very focused way to achieve my goal](https://github.com/guino/Merkury1080P/issues/11#issuecomment-934362395).

## Details of camera model LaView F1 LV-PWF1P-W
The following details came from the label on the camera case, an examination of the chips, and the serial runtime logs:

### <ins>Chips:</ins>
- Anyka AK3918EN080  V331L  BESJ07J21
- AltoBeam 60321S  CB45-61 (WiFi chip)
- NOR-MEM NM25Q64EVBSIG  P2S469-2124 (3V SPI 64Mb flash, equivalent to Winbond W25Q64BV)
- LTK8002D  QM220309  (Audio chip)
### <ins>Other info:</ins>
- Manufacturer: PPS IPC based on ONVIF
- Model: Mini 12Q
- Hardware: Ver 2.1
- Firmware: ppstrong-a4-tuya2_lts-5.3.2.20220826
- SDK_version: &lt; TUYA IPC SDK V:4.9.18 >
- media_version: 4.6.34.2022-08-25T20:33:01,7044
- git log: 2022-08-26, dev_by_cs_master, 4c60d0627,  by cuishang
- Device ID: 107805475
- ONVIF version: 1.0
- URI: http://192.168.0.x:8000/onvif/device_service
- There is a hardcoded Admin user "WeEyE".  This was outside the project scope, so I did not investigate it, but if the camera was internet- or cloud-connected, that seems like a significant backdoor.
- The root password is hashed as `$6$4GbqAXEFqeauykeE$a6dqh2CoO6SucAplB/b4uvS5z0hN1Cb2r1pNWpsXL96vMqrrY42lFylXGNJm6RcY.3Lte/QS2.yyI4/pZDHAa1`, which also appears in other cameras.  Hashcat ran for a week on an RTX5060Ti but didn't crack the password, and I never found the solution online, so the default root password remains a mystery.  If you want shell access, you'll need to replace the password (see below).
- MainStream: VideoSourceConfig0 h.264 (2560x1440) 30fps
  - `rtsp://admin:password@192.168.0.x:8554/Streaming/Channels/101` (2560x1440)
- SubStream: VideoSourceConfig1 h.264 (640x360) 30fps
  - `rtsp://admin:password@192.168.0.x:8554/Streaming/Channels/102` (640x360)
- magic : 0x5354524e
- factory_name : PPSTRONG
- hardware_ver : M12Q_A4_V10_GC3
- model_name : Mini 12Q
- pro_no : 107805475
- p2p_uuid : TUYA_C3
- device_type : 522
- ram_size : 64
- flash_size : 8
- p2p_factory_type : 14

The bootloader is [U-Boot](https://u-boot.org/), documentation for which is [here](https://docs.u-boot.org/).

After setting up the camera using the LaView app, it connected to my WiFi network and worked well.  ONVIF is on port 8000 (vs. the typical use of port 80 or 8080), and RTSP is on port 8554 (vs. the standard port 554).  Telnet, ssh, and ftp are disabled.  Powering the camera up without an internet connection keeps all ports other than 6668 closed while the firmware continuously tries to connect to a Tuya server.  There is no fallback or failsafe - no Tuya connection, no ONVIF or RTSP.  The firmware includes hardcoded addresses, so simply remapping the Tuya addresses in your DNS server won't do the trick.

&nbsp;

### <ins>Flash layout</ins>
This camera's flash (spi0.0) contains 6 MTD partitions:
```
0: 0x000000-0x040000 : "BOOT"    // raw boot code (U-boot itself)
1: 0x040000-0x340000 : "sys"     // U-Boot legacy uImage, Linux-4.4.192V2.1
2: 0x340000-0x770000 : "app"     // Squashfs, little endian, v4.0, xz compressed
3: 0x770000-0x7e0000 : "cfg"     // JFFS2 (configuration data storage)
4: 0x7e0000-0x7f0000 : "enc"     // data
5: 0x7f0000-0x800000 : "sysflg"  // data
```
It appears that most of the elements of the camera's software have been customized, therefore things don't always work as expected, or as the documentation would indicate.  In particular:
- U-Boot's `update` command parses its arguments and goes through the motions of flashing, but does nothing.  This may be to discourage tinkering.
- BusyBox version 1.30.1 is included in the system, but is lacking various functions such as telnetd and ntpd, presumably to save space, or possibly to discourage tinkering.
- NTP is disabled, so pushing an NTP server address from the DHCP server does not help.

&nbsp;

### <ins>Boot trickery</ins>
These two tricks ended up not being useful for my immediate purposes, but I think there's value in documenting them for future reference by others.
- If `ppsFactoryTool.txt` exists in the root directory of the TF/μSD card, it is used to set the WiFi SSID and password.  The file contains one line, newline-terminated, with the format `module=wifi,,ssid=MySSID,,password=MyWiFiPassword,,`.  Note the use of double commas, they are necessary.  If you'd like to add a static IP address, append it as `module=wifi,,ssid=MySSID,,password=MyWiFiPassword,,ipaddr=1.2.3.4,,`.  You can also append a static gateway, as `module=wifi,,ssid=MySSID,,password=MyWiFiPassword,,ipaddr=1.2.3.4,,gateway=5.6.7.8,,`  Note that you can't specify a static gateway without also specifying a static IP address.

Holding the reset button down during boot, starting before powerup and ending about 5-10 seconds later, causes U-Boot to look for a special file in the root directory of the TF/μSD card.
- If `ppsMmcTool.txt` exists, it can be used to manipulate the U-Boot environment at startup.  Because they use the fileName parameter unsanitized, you can pass in additional U-Boot commands appended to the filename.  There are several different formats for the `ppsMmcTool.txt` file, but for our purposes, the following is useful: `style=upgrade,,writeAddr=0,,password=nothing,,writeLen=0,,fileName=X;command1;command2;command3,,`.  The file named in fileName should exist, even if it's empty.  However, it's useful to add U-Boot commands to that file, because starting at the first character of the filename, there is a hard 63-character limit that will truncate any long command line.  If the file named by `fileName` exists, its contents are read into memory at 0x82008000.  Note that the contents of the `fileName` file <ins>**must**</ins> be terminated with a binary NULL (0x00) after the newline.

What can we do with this glitch?
- We can dump the flash to the first 32KB of the TF/μSD card (see [here](https://github.com/guino/BazzDoorbell/issues/11#issue-772420637) for how to partition the card): `style=upgrade,,writeAddr=0,,password=nothing,,writeLen=0,,fileName=X;sf probe;sf read 80008000 0 800000;mmc write 80008000 1 8000,,`
- We can dump the U-Boot environment variables to the serial console: `style=upgrade,,writeAddr=0,,password=nothing,,writeLen=0,,fileName=X;env print -a,,`
- We can try to re-flash a partition (JFFS2, in this case):
  - File `X` contains: `zz=sf probe;mmc read 80008000 1 8000;sf update 80778000 770000 4000;`
  - File `ppsMmcTool.txt` contains: `style=upgrade,,writeAddr=0,,password=n,,writeLen=0,,fileName=X;env import 82008000;saveenv;run zz;,,`
  - This causes file X to be read into 0x82008000, where we import its contents into the U-Boot environment, then execute the variable (macro) `zz`, which reads the first 32KB of the TF/μSD card into 0x80008000, then updates the flash at 0x770000 (the JFFS2 partition).
  - This long command would not fit entirely in ppsMmcTool.txt, it would be truncated by the 63 character limit.  By using a separate file (`X`, in this case), we can create longer commands.
  - This won't actually alter the flash, because as noted above, the U-Boot `update` command seems to be impaired in this version.  I did not try separately erasing and writing a partition, but that might work.
- We can run any sequence of [U-Boot commands](https://docs.u-boot.org/en/stable/usage/index.html) that we want.

For the serial console, note that there are *two* sets of pads that look promising.  The ones you want are right next to the reset button.  Remember that this is 3v3 serial, not RS-232, so use something like a Raspberry Pi to make the connection.

&nbsp;

### <ins>Boot log</ins>
Typical boot log (truncated):
```
U-Boot 2013.10.0-V3.1.28_bchV1.0.00 (May 23 2022 - 17:31:46)
DRAM: 128MB
efuse_read:0x00000002
8MiB
sd detect gpio mode:8!
mmc_sd: 0

total partitions: 6
In:    serial
Out:   serial
Err:   serial
enable watchdog

Hit any key to stop autoboot:  2
..... 1
..... 0
mmc power off ...
sys: size:0x00300000, offset: 0x00040000
SF: 3145728 bytes @ 0x40000 Read: OK
## Booting kernel from Legacy Image at 80008000 ...
   Image Name:   Linux-4.4.192V2.1
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    2657464 Bytes = 2.5 MiB
   Load Address: 80008000
   Entry Point:  80008040
   Verifying Checksum ... OK
   XIP Kernel Image ... OK
   kernel loaded at 0x80008000, end = 0x80290cb8
using: FDT

Starting kernel ...

Uncompressing Linux... done, booting the kernel.
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 4.4.192V2.1 (linye@dell) (gcc version 4.9.4 (Buildroot 2018.02.7-g7ef6f7c) ) #3 Thu Jul 21 14:49:08 CST 2022
[    0.000000] CPU: ARM926EJ-S [41069265] revision 5 (ARMv5TEJ), cr=0005317f
[    0.000000] CPU: VIVT data cache, VIVT instruction cache
[    0.000000] Machine model: anyka,ak39ev33x dev board
[    0.000000] [pps_get_board_mem]  rmem size : 64 MiB
[    0.000000] Reserved memory: reserved region for node 'dma_reserved@0x81400000': base 0x81400000, size 64 MiB
[    0.000000] Memory policy: Data cache writeback
[    0.000000] ANYKA CPU AK39XXEV330 (ID 0x20160101)
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 32512
[    0.000000] Kernel command line: console=ttySAK0,115200n8 mtdparts=spi0.0:256K@0x0(BOOT),3072K@0x40000(sys),4288K@0x340000(app),448K@0x770000(cfg),64K@0x7E0000(enc),64K@0x7F0000(sysflg) mem=128M memsize=128M " pcbversion=B3Q_A4_V10 sensor=gc4653mipi model_name=Mini-12Q
[    0.000000] PID hash table entries: 512 (order: -1, 2048 bytes)
[    0.000000] Dentry cache hash table entries: 16384 (order: 4, 65536 bytes)
[    0.000000] Inode-cache hash table entries: 8192 (order: 3, 32768 bytes)
[    0.000000] Memory: 58336K/131072K available (3310K kernel code, 122K rwdata, 1032K rodata, 1104K init, 192K bss, 72736K reserved, 0K cma-reserved)
{...}
```

&nbsp;

### <ins>JFFS2 (NVRAM filesystem)</ins>
The JFFS2 partition contains a few files, most notably `tuya_config_enc.json`.  Earlier versions of similar cameras used a plaintext file, `tuya_config.json`, which could be easily overwritten with the desired options. `tuya_config_enc.json` is just an AES128-ECB encrypted version of `tuya_config.json`, using the key `_MEARI5656509999`, which is hardcoded into the ppsapp executable (with an XOR mask for "security").  There are many web sites that offer online AES128-ECB decoding, and using one of those, we can see that the JSON format is:
```
{"version":1,"sleep_mode":0,"alarm_fun_onoff":0,"alarm_fun_sensitivity":1,"alarm_fun_mode_switch":0,"alarm_fun_time_start":0,"alarm_fun_time_end":0,"flip_onoff":0,"light_onoff":0,"night_mode":0,"sound_detect_onoff":0,"sound_detect_sensitivity":0,"watermark_onoff":1,"event_record_time":60,"enable_event_record":2,"record_enable":1,"motion_trace":1,"motion_area_switch":0,"motion_area":"","motion_tracking":0,"cry_detection_switch":0,"humanoid_filter":0,"loudspeaker_vol_pct":100,"flight_pir_light_on_time":30,"flight_siren_volume":100,"flight_siren_duration":30,"flight_main_mode":0,"static_ip_enable":0,"onvif_enable":1,"onvif_pwd":"MyPassword","sound_light_switch":0,"polygon_area":""}
```
This version of ppsapp looks for `tuya_config.json` before looking for `tuya_config_enc.json`.  If the plaintext version exists, it reads and parses it, then writes its (encrypted) values out to `tuya_config_enc.json`, which it then reads back in, before deleting `tuya_config.json`.  In other words, if you place a valid `tuya_config.json` into the JFFS2 partition, it should permanently supersede any existing `tuya_config_enc.json`.  This is useful if you want to change options after taking the camera offline, since you can no longer use the app to make those changes.

You may notice that `"onvif_enable":1` is present in that JSON.  That is necessary to enable ONVIF once the camera is fully connected, but it is not sufficient to overcome the Tuya connectivity gate (meaning that just having `"onvif_enable":1` in the configuration won't open the ONVIF ports without a Tuya connection).

&nbsp;

### <ins>Squashfs (read-only application filesystem)</ins>
MTD partition 2 is a squashfs, which you can easily manipulate on a Linux system.  Assuming you extract MTD partition 2 to a file named `part2data`:
```
mkdir /tmp/MySquashfs
mount part2data /tmp/MySquashfs
mkdir /tmp/MySquashEdit
cp -pr /tmp/MySquashfs/* /tmp/MySquashEdit
(Make modifications to files in the /tmp/MySquashEdit tree)
mksquashfs /tmp/MySquashEdit part2modified -comp xz
```
At this point, assuming you haven't made it too big (limit here is 4288 KB), `part2modified` can be injected back into MTD partition 2 and flashed to the device.

Because the U-Boot `update` command seems to be impaired, I ended up using a CH341A programmer to flash the chip, using the settings for a Winbond W25Q64BV.  Using the clip in-circuit as-is causes problems, so I found that desoldering pin 8 and raising it just far enough off the pad to slip in a piece of insulating tape was good enough.

The squashfs partition contains all of the application-specific files, including device drivers, libraries, config files, program executables, etc.  At runtime, the root of the squashfs partition gets mounted at /opt/pps.  The file /opt/pps/app/binfiles/bin/ppsapp is the main executable, which performs both system administration and video streaming tasks.  It is also the gatekeeper that prevents anything useful from happening without a connection to a Tuya server.

The Tuya SDK initialization code tries to set up a MQTT connection to a Tuya server, and once that's in place, it will continue to initialize everything else, including ONVIF/RTSP.  Bypassing the "wait for MQTT connection" code allows ONVIF/RTSP to launch and keep running, regardless of the fact that all subsequent MQTT calls will fail.  [jilleb made this discovery on a similar camera](https://github.com/guino/Merkury1080P/issues/11#issuecomment-934362395).  In that case, the code made a function call to check the MQTT status.  In this LaView camera, a separate thread updates a global variable that the Tuya SDK initialization code can simply check.  If we change "while (variable != 1)" to "while (variable != 0)", we bypass the "wait for MQTT connection" gate.

<img width="1378" height="904" alt="image" src="https://github.com/user-attachments/assets/a5d6783d-e5b0-470e-b333-0f19198652cc" />

Unfortunately, in an effort to make tinkering like this more difficult, they made system administration less straightforward.  The ppsapp executable includes numerous `system()` calls to perform tasks that would normally be done in shell scripts, such as starting and stopping interfaces, loading and unloading device drivers, launching dhcpd, etc.  They also removed functionality from the busybox instance, so it can't provide ntpd or telnetd.  The system time is set to a value that comes from the Tuya server, not by NTP.  So, to round out the changes I needed to make for my own purposes, I did the following:
- Downloaded the AK3918_V330L dump from [an OpenIPC issue](https://github.com/OpenIPC/firmware/issues/1970) and extracted the BusyBox executable, which is compatible with this camera
- Added busybox to squashfs as /app/binfiles/bin/ntpd
- Also in /app/binfiles/bin, created a symlink from ntpd to telnetd
- Deleted unused driver from squashfs::/app/kofiles/ko/8188fu.ko  (otherwise, compressed filesystem would exceed partition size)
- Added several files to squashfs::/app/init
  - `ntp.conf`, containing the IP address of my local NTP server (e.g. `server 192.168.0.250`)
  - `TZ`, containing my local timezone (e.g. `EST5EDT`)
  - `_shadow`, a replacement shadow file containing a known root password (e.g. `root:$1$QBrxEKGk$W5IDD6e4.lb0oinzlorWj0:0:0:99999:7:::`) (that password is `TestPassword`)
  - `_copyTZ`, a shell script that simply sleeps for 45 seconds, then copies my `TZ` file to `/etc/TZ`.  This is necessary because something (busybox or otherwise) overwrites any existing /etc/TZ with a default version that uses CST.  The 45 second sleep is sufficient for the camera to connect to my network and ntpd to initialize, but a longer delay (60-75 seconds) would be more flexible and allow for troublesome WiFi connections.
  - `S59myInit`, a shell script that will automatically be executed at startup, after device and WiFi initialization but before ppsapp starts.  This script copies my ntp.conf to /etc, copies my _shadow to overwrite /etc/shadow, launches ntpd and telnetd as background processes, then launches _copyTZ as a background process.

&nbsp;

## Conclusion

At this point, I have a fully functional camera with a correct date/time stamp and a working telnet server with a known root password, all of which will operate without an internet connection.  I was able to flash the same binary image to all of the cameras I bought, and they all worked correctly, so apparently things like MAC addresses are not stored in the main flash.  As noted above, I am not going to post any binaries, or a hand-holding step-by-step guide to modifying the camera.  [@guino](https://github.com/guino)'s repositories have more than enough details for those with the necessary skills.
