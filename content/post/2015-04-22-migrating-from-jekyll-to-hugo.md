---
title: "fastboot: FAILED (remote: partition does not exist)"
date: "2016-03-17"
description: I had a little heart attack when my phone wasn't booting anymore after fastboot failed. Fortunately there was a quick fix for this.
image:
tags:
  - fastboot
  - Android
  - Xiaomi MI4
  - MIUI
---

If you ever tried to update your phone's ROM and got this error `FAILED (remote: partition does not exist)`,
this is the way to fix it.  

Tested on Xiaomi MI4 and `cancro_global_images_V7.1.2.0.KXDMICK_20160125.0000.13_4.4_globalz`.  

Open up the script you're using to flash the device and add the line highlighted
in this snippet.

```bash  
fastboot $* getvar product 2>&1 | grep "^product: *MSM8974$"  
if [ $? -ne 0 ] ; then echo "Missmatching image and device"; exit 1; fi  
fastboot $* getvar board_version 2>&1 | grep "^board_version: *4.4"  
if [ $? -eq 0 ] ; then echo "Missmatching board version"; exit 1; fi  
fastboot $* getvar board_version 2>&1 | grep "^board_version: *5.[0-9]"  
if [ $? -eq 0 ] ; then echo "Missmatching board version"; exit 1; fi  

fastboot $* flash partition `dirname $0`/images/gpt_both0.bin # <- ADD THIS
-------------------------------------------------------------

fastboot $* flash tz `dirname $0`/images/tz.mbn  
fastboot $* flash dbi `dirname $0`/images/sdi.mbn  
```
> Shell script


```
fastboot %* getvar product 2>&1 | findstr /r /c:"^product: *MSM8974" || echo Missmatching image and device
fastboot %* getvar product 2>&1 | findstr /r /c:"^product: *MSM8974" || exit /B 1
fastboot %* getvar board_version 2>&1 | findstr /r /c:"^board_version: *4.4" && echo Missmatching board version
fastboot %* getvar board_version 2>&1 | findstr /r /c:"^board_version: *4.4" && exit /B 1
fastboot %* getvar board_version 2>&1 | findstr /r /c:"^board_version: *5.[0-9]" && echo Missmatching board version
fastboot %* getvar board_version 2>&1 | findstr /r /c:"^board_version: *5.[0-9]" && exit /B 1

fastboot %* flash partition %~dp0images\gpt_both0.bin # <- ADD THIS
-----------------------------------------------------

fastboot %* flash tz %~dp0images\tz.mbn
```
> Windows bat script
