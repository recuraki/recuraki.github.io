---
layout: post
title:  "PCIeのSMBusと仲良くなりたい"
date:   2023-11-10T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20231109: PCIeのSMBusと仲良くなりたい


```
sudo modprobe i2c-dev

sudo modprobe i2c-smbus
sudo apt install i2c-tools
```

```

ubuntu@optiplex:/dev$ sudo i2cdetect -l
i2c-0	smbus     	SMBus I801 adapter at f040      	SMBus adapter
i2c-1	i2c       	i915 gmbus dpc                  	I2C adapter
i2c-2	i2c       	i915 gmbus dpb                  	I2C adapter
i2c-3	i2c       	i915 gmbus dpd                  	I2C adapter
i2c-4	i2c       	AUX C/DDI C/PHY C               	I2C adapter
i2c-5	i2c       	AUX D/DDI D/PHY D               	I2C adapter
i2c-6	i2c       	AUX A/DDI E/PHY E               	I2C adapter

```

```
ubuntu@optiplex:/dev/mst$ sudo i2cdetect -F 0
Functionalities implemented by /dev/i2c-0:
I2C                              no
SMBus Quick Command              yes
SMBus Send Byte                  yes
SMBus Receive Byte               yes
SMBus Write Byte                 yes
SMBus Read Byte                  yes
SMBus Write Word                 yes
SMBus Read Word                  yes
SMBus Process Call               no
SMBus Block Write                yes
SMBus Block Read                 yes
SMBus Block Process Call         yes
SMBus PEC                        yes
I2C Block Write                  yes
I2C Block Read                   yes
```