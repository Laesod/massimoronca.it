---
title: Disable captive network support in OS X
date: "2015-05-20"
description: 
draft: true
tags:
- OS X
- captive network support
- captive portals
- WI-FI authorization
---

iOS4+ and OS X (10.7+) Devices have a feature called Captive Network Support, which when you connect to an access point tries to download:

http://www.apple.com/library/test/success.html

to see if the device is connected to the internet. If it doesn't get the success response it assumes you are behind a captive portal and pops a webkit window so you can do the portal dance.  This is mostly useful if you are using thick-client apps, since if you're using a browser you're going to see the portal page as soon as you go anywhere.

To disable it, set this preference:

sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.captive.control Active -boolean false`