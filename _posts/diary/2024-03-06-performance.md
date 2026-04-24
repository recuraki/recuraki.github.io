---
layout: post
title:  "20240306日記: NVMeが壊れた"
date:   2024-03-06T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20240306日記: NVMeが壊れた

急にWindowsが立ち上がらなくなった。

- Ubuntu Desktopをインストールして起動してTry Ubuntuで起動
```
sudo apt install openssh-server
passwd
 -> 適当に設定
sudo /etc/init.d/ssh restart
```
- これで他のPCからログインできる。ssh ubuntu@172.16.0.1など。
- https://github.com/AltraMayor/f3/tags から適当なpkgをダウンロード。

```
wget https://github.com/AltraMayor/f3/archive/refs/tags/v8.0.tar.gz
tar zxfv v8.0.tar.gz
sudo apt update
sudo apt install make
sudo apt -y install gcc
sudo  apt install libudev-dev
sudo apt install smartmontools
sudo apt install nvme-cli
make
make f3probe
sudo ./f3write /export
sudo ./f3read /export
```
## f3probe
```
ubuntu@ubuntu:~/f3-8.0$ sudo ./f3probe --destructive --time-ops /dev/nvme0n1
F3 probe 8.0


Copyright (C) 2010 Digirati Internet LTDA.
This is free software; see the source for copying conditions.

WARNING: Probing normally takes from a few seconds to 15 minutes, but
         it can take longer. Please be patient.

Bad news: The device `/dev/nvme0n1' is damaged

Device geometry:
	         *Usable* size: 0.00 Byte (0 blocks)
	        Announced size: 3.64 TB (7814037168 blocks)
	                Module: 4.00 TB (2^42 Bytes)
	Approximate cache size: 0.00 Byte (0 blocks), need-reset=no
	   Physical block size: 512.00 Byte (2^9 Bytes)

Probe time: 50.7ms
 Operation: total time / count = avg time
      Read: 0us / 0 = 0us
     Write: 50.0ms / 4096 = 12us
     Reset: 0us / 0 = 0us
```

## smartctl
```
ubuntu@ubuntu:~/f3-8.0$ sudo smartctl /dev/nvme0 --all
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-6.2.0-26-generic] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Number:                       CT4000P3PSSD8
Serial Number:                      xxxxxxxxx
Firmware Version:                   P9CR40A
PCI Vendor/Subsystem ID:            0xc0a9
IEEE OUI Identifier:                0x00a075
Controller ID:                      1
NVMe Version:                       1.4
Number of Namespaces:               1
Namespace 1 Size/Capacity:          4,000,787,030,016 [4.00 TB]
Namespace 1 Formatted LBA Size:     512
Namespace 1 IEEE EUI-64:            6479a7 6d60000018
Local Time is:                      Tue Mar 12 12:38:15 2024 UTC
Firmware Updates (0x12):            1 Slot, no Reset required
Optional Admin Commands (0x0017):   Security Format Frmw_DL Self_Test
Optional NVM Commands (0x005e):     Wr_Unc DS_Mngmt Wr_Zero Sav/Sel_Feat Timestmp
Log Page Attributes (0x06):         Cmd_Eff_Lg Ext_Get_Lg
Maximum Data Transfer Size:         64 Pages
Warning  Comp. Temp. Threshold:     85 Celsius
Critical Comp. Temp. Threshold:     95 Celsius

Supported Power States
St Op     Max   Active     Idle   RL RT WL WT  Ent_Lat  Ex_Lat
 0 +     6.00W  0.0000W       -    0  0  0  0        0       0
 1 +     3.00W  0.0000W       -    0  0  0  0        0       0
 2 +     1.50W  0.0000W       -    0  0  0  0        0       0
 3 -   0.0250W  0.0000W       -    3  3  3  3     5000    1900
 4 -   0.0030W       -        -    4  4  4  4    13000  100000

Supported LBA Sizes (NSID 0x1)
Id Fmt  Data  Metadt  Rel_Perf
 0 +     512       0         1
 1 -    4096       0         0

=== START OF SMART DATA SECTION ===
SMART overall-health self-assessment test result: FAILED!
- available spare has fallen below threshold
- media has been placed in read only mode

SMART/Health Information (NVMe Log 0x02)
Critical Warning:                   0x09
Temperature:                        40 Celsius
Available Spare:                    0%
Available Spare Threshold:          5%
Percentage Used:                    1%
Data Units Read:                    46,954,492 [24.0 TB]
Data Units Written:                 28,871,767 [14.7 TB]
Host Read Commands:                 419,343,766
Host Write Commands:                358,937,031
Controller Busy Time:               2,444
Power Cycles:                       395
Power On Hours:                     1,103
Unsafe Shutdowns:                   40
Media and Data Integrity Errors:    0
Error Information Log Entries:      4,662
Warning  Comp. Temperature Time:    0
Critical Comp. Temperature Time:    0
Temperature Sensor 1:               40 Celsius
Temperature Sensor 2:               59 Celsius
Temperature Sensor 8:               40 Celsius

Error Information (NVMe Log 0x01, 16 of 16 entries)
Num   ErrCount  SQId   CmdId  Status  PELoc          LBA  NSID    VS
  0       4662     4  0x51c7  0x4041  0x004   7814036656     1     -
  1       4661     4  0x51c6  0x4041  0x004   7814036144     1     -
  2       4660     4  0x61c5  0x4041  0x004   7814035632     1     -
  3       4659     4  0x91c4  0x4041  0x004   7814035120     1     -
  4       4658     4  0x41c7  0x4041  0x004   7814036656     1     -
  5       4657     4  0x41c6  0x4041  0x004   7814036144     1     -
  6       4656     4  0x51c5  0x4041  0x004   7814035632     1     -
  7       4655     4  0x81c4  0x4041  0x004   7814035120     1     -
  8       4654     4  0x31c7  0x4041  0x004   7814036656     1     -
  9       4653     4  0x31c6  0x4041  0x004   7814036144     1     -
 10       4652     4  0x41c5  0x4041  0x004   7814035632     1     -
 11       4651     4  0x71c4  0x4041  0x004   7814035120     1     -
 12       4650     4  0x21c7  0x4041  0x004   7814036656     1     -
 13       4649     4  0x21c6  0x4041  0x004   7814036144     1     -
 14       4648     4  0x31c5  0x4041  0x004   7814035632     1     -
 15       4647     4  0x61c4  0x4041  0x004   7814035120     1     -
 ```

## nvme-cli
```
ubuntu@ubuntu:~/f3-8.0$ sudo nvme  list
Node                  SN                   Model                                    Namespace Usage                      Format           FW Rev
--------------------- -------------------- ---------------------------------------- --------- -------------------------- ---------------- --------
/dev/nvme0n1          xxxxx         CT4000P3PSSD8                            1           4.00  TB /   4.00  TB    512   B +  0 B   P9CR40A
ubuntu@ubuntu:~/f3-8.0$ sudo nvme id-ctrl -H /dev/nvme0n1
NVME Identify Controller:
vid       : 0xc0a9
ssvid     : 0xc0a9
sn        : 2242E67C541A
mn        : CT4000P3PSSD8
fr        : P9CR40A
rab       : 1
ieee      : 00a075
cmic      : 0
  [3:3] : 0	ANA not supported
  [2:2] : 0	PCI
  [1:1] : 0	Single Controller
  [0:0] : 0	Single Port

mdts      : 6
cntlid    : 0x1
ver       : 0x10400
rtd3r     : 0x124f80
rtd3e     : 0x2191c0
oaes      : 0
  [31:31] : 0	Discovery Log Change Notice Not Supported
  [27:27] : 0	Zone Descriptor Changed Notices Not Supported
  [15:15] : 0	Normal NSS Shutdown Event Not Supported
  [14:14] : 0	Endurance Group Event Aggregate Log Page Change Notice Not Supported
  [13:13] : 0	LBA Status Information Notices Not Supported
  [12:12] : 0	Predictable Latency Event Aggregate Log Change Notices Not Supported
  [11:11] : 0	Asymmetric Namespace Access Change Notices Not Supported
  [9:9] : 0	Firmware Activation Notices Not Supported
  [8:8] : 0	Namespace Attribute Changed Event Not Supported

ctratt    : 0
  [15:15] : 0	Extended LBA Formats Not Supported
  [14:14] : 0	Delete NVM Set Not Supported
  [13:13] : 0	Delete Endurance Group Not Supported
  [12:12] : 0	Variable Capacity Management Not Supported
  [11:11] : 0	Fixed Capacity Management Not Supported
  [10:10] : 0	Multi Domain Subsystem Not Supported
  [9:9] : 0	UUID List Not Supported
  [7:7] : 0	Namespace Granularity Not Supported
  [5:5] : 0	Predictable Latency Mode Not Supported
  [4:4] : 0	Endurance Groups Not Supported
  [3:3] : 0	Read Recovery Levels Not Supported
  [2:2] : 0	NVM Sets Not Supported
  [1:1] : 0	Non-Operational Power State Permissive Not Supported
  [0:0] : 0	128-bit Host Identifier Not Supported

rrls      : 0
cntrltype : 1
  [7:2] : 0	Reserved
  [1:0] : 0x1	I/O Controller
fguid     :
crdt1     : 0
crdt2     : 0
crdt3     : 0
nvmsr     : 0
  [1:1] : 0	NVM subsystem Not part of an Enclosure
  [0:0] : 0	NVM subsystem Not part of an Storage Device

vwci      : 0
  [7:7] : 0	VPD Write Cycles Remaining field is Not valid.
  [6:0] : 0	VPD Write Cycles Remaining

mec       : 0
  [1:1] : 0	NVM subsystem Not contains a Management Endpoint on a PCIe port
  [0:0] : 0	NVM subsystem Not contains a Management Endpoint on an SMBus/I2C port

oacs      : 0x17
  [10:10] : 0	Lockdown Command and Feature Not Supported
  [9:9] : 0	Get LBA Status Capability Not Supported
  [8:8] : 0	Doorbell Buffer Config Not Supported
  [7:7] : 0	Virtualization Management Not Supported
  [6:6] : 0	NVMe-MI Send and Receive Not Supported
  [5:5] : 0	Directives Not Supported
  [4:4] : 0x1	Device Self-test Supported
  [3:3] : 0	NS Management and Attachment Not Supported
  [2:2] : 0x1	FW Commit and Download Supported
  [1:1] : 0x1	Format NVM Supported
  [0:0] : 0x1	Security Send and Receive Supported

acl       : 0
aerl      : 3
frmw      : 0x12
  [5:5] : 0	Multiple FW or Boot Update Detection Not Supported
  [4:4] : 0x1	Firmware Activate Without Reset Supported
  [3:1] : 0x1	Number of Firmware Slots
  [0:0] : 0	Firmware Slot 1 Read/Write

lpa       : 0x6
  [6:6] : 0	Telemetry Log Data Area 4 Not Supported
  [5:5] : 0	LID 0x0, Scope of each command in LID 0x5, 0x12, 0x13 Not Supported
  [4:4] : 0	Persistent Event log Not Supported
  [3:3] : 0	Telemetry host/controller initiated log page Not Supported
  [2:2] : 0x1	Extended data for Get Log Page Supported
  [1:1] : 0x1	Command Effects Log Page Supported
  [0:0] : 0	SMART/Health Log Page per NS Not Supported

elpe      : 15
npss      : 4
avscc     : 0x1
  [0:0] : 0x1	Admin Vendor Specific Commands uses NVMe Format

apsta     : 0x1
  [0:0] : 0x1	Autonomous Power State Transitions Supported

wctemp    : 358
 [16:0] : 85 C (358 Kelvin)	Warning temperature (WCTEMP)

cctemp    : 368
 [16:0] : 95 C (368 Kelvin)	Critical temperature (CCTEMP)

mtfa      : 100
hmpre     : 8192
hmmin     : 8192
tnvmcap   : 0
unvmcap   : 0
rpmbs     : 0
 [31:24]: 0	Access Size
 [23:16]: 0	Total Size
  [5:3] : 0	Authentication Method
  [2:0] : 0	Number of RPMB Units

edstt     : 30
dsto      : 1
fwug      : 4
kas       : 1
hctma     : 0x1
  [0:0] : 0x1	Host Controlled Thermal Management Supported

mntmt     : 273
mxtmt     : 358
sanicap   : 0x40000002
  [31:30] : 0x1	Media is not additionally modified after sanitize operation completes successfully
  [29:29] : 0	No-Deallocate After Sanitize bit in Sanitize command Supported
    [2:2] : 0	Overwrite Sanitize Operation Not Supported
    [1:1] : 0x1	Block Erase Sanitize Operation Supported
    [0:0] : 0	Crypto Erase Sanitize Operation Not Supported

hmminds   : 512
hmmaxd    : 16
nsetidmax : 0
endgidmax : 0
anatt     : 0
anacap    : 0
  [7:7] : 0	Non-zero group ID Not Supported
  [6:6] : 0	Group ID does not change
  [4:4] : 0	ANA Change state Not Supported
  [3:3] : 0	ANA Persistent Loss state Not Supported
  [2:2] : 0	ANA Inaccessible state Not Supported
  [1:1] : 0	ANA Non-optimized state Not Supported
  [0:0] : 0	ANA Optimized state Not Supported

anagrpmax : 0
nanagrpid : 0
pels      : 0
domainid  : 0
megcap    : 0
sqes      : 0x66
  [7:4] : 0x6	Max SQ Entry Size (64)
  [3:0] : 0x6	Min SQ Entry Size (64)

cqes      : 0x44
  [7:4] : 0x4	Max CQ Entry Size (16)
  [3:0] : 0x4	Min CQ Entry Size (16)

maxcmd    : 0
nn        : 1
oncs      : 0x5e
  [8:8] : 0	Copy Not Supported
  [7:7] : 0	Verify Not Supported
  [6:6] : 0x1	Timestamp Supported
  [5:5] : 0	Reservations Not Supported
  [4:4] : 0x1	Save and Select Supported
  [3:3] : 0x1	Write Zeroes Supported
  [2:2] : 0x1	Data Set Management Supported
  [1:1] : 0x1	Write Uncorrectable Supported
  [0:0] : 0	Compare Not Supported

fuses     : 0
  [0:0] : 0	Fused Compare and Write Not Supported

fna       : 0x1
  [3:3] : 0	FormatNVM Broadcast NSID (FFFFFFFFh) Supported
  [2:2] : 0	Crypto Erase Not Supported as part of Secure Erase
  [1:1] : 0	Crypto Erase Applies to Single Namespace(s)
  [0:0] : 0x1	Format Applies to All Namespace(s)

vwc       : 0x7
  [2:1] : 0x3	The Flush command supports NSID set to FFFFFFFFh
  [0:0] : 0x1	Volatile Write Cache Present

awun      : 255
awupf     : 0
icsvscc     : 1
  [0:0] : 0x1	NVM Vendor Specific Commands uses NVMe Format

nwpc      : 0
  [2:2] : 0	Permanent Write Protect Not Supported
  [1:1] : 0	Write Protect Until Power Supply Not Supported
  [0:0] : 0	No Write Protect and Write Protect Namespace Not Supported

acwu      : 0
ocfs      : 0
  [1:1] : 0	Controller Copy Format 1h Not Supported
  [0:0] : 0	Controller Copy Format 0h Not Supported

sgls      : 0
 [15:8] : 0	SGL Descriptor Threshold
 [1:0]  : 0	Scatter-Gather Lists Not Supported

mnan      : 0
maxdna    : 0
maxcna    : 0
subnqn    :
ioccsz    : 0
iorcsz    : 0
icdoff    : 0
fcatt     : 0
  [0:0] : 0	Dynamic Controller Model

msdbd     : 0
ofcs      : 0
  [0:0] : 0	Disconnect command Not Supported

ps    0 : mp:6.00W operational enlat:0 exlat:0 rrt:0 rrl:0
          rwt:0 rwl:0 idle_power:0.6720W active_power:0.00W
ps    1 : mp:3.00W operational enlat:0 exlat:0 rrt:0 rrl:0
          rwt:0 rwl:0 idle_power:0.4360W active_power:0.00W
ps    2 : mp:1.50W operational enlat:0 exlat:0 rrt:0 rrl:0
          rwt:0 rwl:0 idle_power:0.3450W active_power:0.00W
ps    3 : mp:0.0250W non-operational enlat:5000 exlat:1900 rrt:3 rrl:3
          rwt:3 rwl:3 idle_power:0.0810W active_power:0.00W
ps    4 : mp:0.0030W non-operational enlat:13000 exlat:100000 rrt:4 rrl:4
          rwt:4 rwl:4 idle_power:- active_power:-

ubuntu@ubuntu:~/f3-8.0$ sudo nvme format /dev/nvme0n1 -s 2
You are about to format nvme0n1, namespace 0xffffffff(ALL namespaces).
Namespace nvme0n1 has parent controller(s):nvme0

WARNING: Format may irrevocably delete this device's data.
You have 10 seconds to press Ctrl-C to cancel this operation.

Use the force [--force|-f] option to suppress this warning.
Sending format operation ...
NVMe status: NS_WRITE_PROTECTED: The command is prohibited while the namespace is write protected by the host.(0x2020)



ubuntu@ubuntu:~/f3-8.0$ sudo nvme sanitize-log /dev/nvme0
Sanitize Progress                      (SPROG) :  65535
Sanitize Status                        (SSTAT) :  0
Sanitize Command Dword 10 Information (SCDW10) :  0
Estimated Time For Overwrite                   :  4294967295 (No time period reported)
Estimated Time For Block Erase                 :  4294967295 (No time period reported)
Estimated Time For Crypto Erase                :  4294967295 (No time period reported)
Estimated Time For Overwrite (No-Deallocate)   :  4294967295 (No time period reported)
Estimated Time For Block Erase (No-Deallocate) :  4294967295 (No time period reported)
Estimated Time For Crypto Erase (No-Deallocate):  4294967295 (No time period reported)

```

## hdparam
```shell
ubuntu@ubuntu:~/f3-8.0$ sudo hdparm /dev/nvme0n1

/dev/nvme0n1:
 readonly      =  0 (off)
 readahead     = 256 (on)
 geometry      = 3815447/64/32, sectors = 7814037168, start = 0

ubuntu@ubuntu:~/f3-8.0$ sudo hdparm -Tt /dev/nvme0n1

/dev/nvme0n1:
 Timing cached reads:   45850 MB in  2.00 seconds = 22968.76 MB/sec
 Timing buffered disk reads: 1110 MB in  3.00 seconds = 369.66 MB/sec


ubuntu@ubuntu:~/f3-8.0$ sudo nvme smart-log /dev/nvme0n1
Smart Log for NVME device:nvme0n1 namespace-id:ffffffff
critical_warning			: 0x9
temperature				: 39 C (312 Kelvin)
available_spare				: 0%
available_spare_threshold		: 5%
percentage_used				: 1%
endurance group critical warning summary: 0x9
data_units_read				: 46956775
data_units_written			: 28871767
host_read_commands			: 419348225
host_write_commands			: 358937031
controller_busy_time			: 2444
power_cycles				: 395
power_on_hours				: 1103
unsafe_shutdowns			: 40
media_errors				: 0
num_err_log_entries			: 4698
Warning Temperature Time		: 0
Critical Composite Temperature Time	: 0
Temperature Sensor 1           : 39 C (312 Kelvin)
Temperature Sensor 2           : 59 C (332 Kelvin)
Temperature Sensor 8           : 39 C (312 Kelvin)
Thermal Management T1 Trans Count	: 0
Thermal Management T2 Trans Count	: 0
Thermal Management T1 Total Time	: 0
Thermal Management T2 Total Time	: 0
ubuntu@ubuntu:~/f3-8.0$ sudo nvme error-log /dev/nvme0n1
Error Log Entries for device:nvme0n1 entries:16
.................
 Entry[ 0]
.................
error_count	: 4698
sqid		: 7
cmdid		: 0x1282
status_field	: 0x2020(NS_WRITE_PROTECTED: The command is prohibited while the namespace is write protected by the host.)
phase_tag	: 0x1
parm_err_loc	: 0
lba		: 0
nsid		: 0x1
vs		: 0
trtype		: The transport type is not indicated or the error is not transport related.
cs		: 0
trtype_spec_info: 0
.................
```

## ついでに色々みておく
```
sudo apt install numactl
```

## numactl
```
ubuntu@optiplex:~$ sudo numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1 2 3
node 0 size: 15893 MB
node 0 free: 14820 MB
node distances:
node   0
  0:  10
  ```

## lsblk
```
ubuntu@ubuntu:~/f3-8.0$ lsblk -f
NAME    FSTYPE FSVER LABEL                UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0   squash 4.0                                                                   0   100% /rofs
loop1   squash 4.0                                                                   0   100% /snap/core20/1974
loop2   squash 4.0                                                                   0   100% /snap/bare/5
loop3   squash 4.0                                                                   0   100% /snap/core22/858
loop4   squash 4.0                                                                   0   100% /snap/firefox/2987
loop5   squash 4.0                                                                   0   100% /snap/gnome-3-38-2004/143
loop6   squash 4.0                                                                   0   100% /snap/gnome-42-2204/120
loop7   squash 4.0                                                                   0   100% /snap/gtk-common-themes/1535
loop8   squash 4.0                                                                   0   100% /snap/snap-store/959
loop9   squash 4.0                                                                   0   100% /snap/snapd-desktop-integration/83
loop10  squash 4.0                                                                   0   100% /snap/snapd/19457
sda     iso966 Jolie Ubuntu 22.04.3 LTS amd64
│                                         2023-08-08-01-19-05-00
├─sda1  iso966 Jolie Ubuntu 22.04.3 LTS amd64
│                                         2023-08-08-01-19-05-00                     0   100% /cdrom
├─sda2  vfat   FAT12 ESP                  F7DB-4D56
├─sda3
└─sda4  ext4   1.0   writable             399ca36f-2a90-4284-8b07-2bb6fec6014d   22.4G     0% /var/crash
                                                                                              /var/log
sdb
nvme0n1
├─nvme0n1p1
│
├─nvme0n1p2
│       vfat   FAT32                      1B2F-4C4E
├─nvme0n1p3
│       ntfs                              FB311356AE59B0EC
├─nvme0n1p4
│       ntfs                              9FBFFCD6E6D2D14B
└─nvme0n1p5
        ntfs         ボリューム           6C58C216CF9904FD
```

## lscpu
```
ubuntu@optiplex:~$ lscpu
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         39 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  4
  On-line CPU(s) list:   0-3
Vendor ID:               GenuineIntel
  Model name:            Intel(R) Core(TM) i5-6500 CPU @ 3.20GHz
    CPU family:          6
    Model:               94
    Thread(s) per core:  1
    Core(s) per socket:  4
    Socket(s):           1
    Stepping:            3
    CPU max MHz:         3600.0000
    CPU min MHz:         800.0000
    BogoMIPS:            6399.96
    Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fx
                         sr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts re
                         p_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx
                         est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer
                          aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault epb invpcid_single pti ssbd ib
                         rs ibpb stibp tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep b
                         mi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm
                         ida arat pln pts hwp hwp_notify hwp_act_window hwp_epp md_clear flush_l1d arch_capabilities
Virtualization features:
  Virtualization:        VT-x
Caches (sum of all):
  L1d:                   128 KiB (4 instances)
  L1i:                   128 KiB (4 instances)
  L2:                    1 MiB (4 instances)
  L3:                    6 MiB (1 instance)
NUMA:
  NUMA node(s):          1
  NUMA node0 CPU(s):     0-3
Vulnerabilities:
  Gather data sampling:  Not affected
  Itlb multihit:         KVM: Mitigation: VMX disabled
  L1tf:                  Mitigation; PTE Inversion; VMX conditional cache flushes, SMT disabled
  Mds:                   Mitigation; Clear CPU buffers; SMT disabled
  Meltdown:              Mitigation; PTI
  Mmio stale data:       Mitigation; Clear CPU buffers; SMT disabled
  Retbleed:              Mitigation; IBRS
  Spec rstack overflow:  Not affected
  Spec store bypass:     Mitigation; Speculative Store Bypass disabled via prctl and seccomp
  Spectre v1:            Mitigation; usercopy/swapgs barriers and __user pointer sanitization
  Spectre v2:            Mitigation; IBRS, IBPB conditional, STIBP disabled, RSB filling, PBRSB-eIBRS Not affected
  Srbds:                 Mitigation; Microcode
  Tsx async abort:       Mitigation; TSX disabled
```

## lstopo
```
ubuntu@optiplex:~$ sudo lstopo
Machine (16GB total)
  Package L#0
    NUMANode L#0 (P#0 16GB)
    L3 L#0 (6144KB)
      L2 L#0 (256KB) + L1d L#0 (32KB) + L1i L#0 (32KB) + Core L#0 + PU L#0 (P#0)
      L2 L#1 (256KB) + L1d L#1 (32KB) + L1i L#1 (32KB) + Core L#1 + PU L#1 (P#1)
      L2 L#2 (256KB) + L1d L#2 (32KB) + L1i L#2 (32KB) + Core L#2 + PU L#2 (P#2)
      L2 L#3 (256KB) + L1d L#3 (32KB) + L1i L#3 (32KB) + Core L#3 + PU L#3 (P#3)
  HostBridge
    PCIBridge
      PCI 01:00.0 (Ethernet)
        Net "enp1s0f0np0"
        OpenFabrics "mlx5_0"
      PCI 01:00.1 (Ethernet)
        Net "enp1s0f1np1"
        OpenFabrics "mlx5_1"
    PCI 00:02.0 (VGA)
    PCI 00:17.0 (SATA)
      Block(Removable Media Device) "sr0"
      Block(Disk) "sda"
    PCIBridge
      PCI 02:00.0 (Ethernet)
        Net "enp2s0"
    PCIBridge
      PCI 03:00.0 (Ethernet)
        Net "enp3s0"
```