+++
title = "QuietComfort 35 + MacBook + iPhone Bluetooth Connectivity"
description = "Automating Bluetooth connectivity management for Bose QuietComfort 35 headphones using Power Manager profiles and blueutil to automatically disconnect from MacBook when lid is closed."
date = 2019-03-14
aliases = ["2019/03/14/quietcomfort-35-macbook-iphone-bluetooth-connectivity.html", "/quietcomfort-35-macbook-iphone-bluetooth-connectivity/"]
template = "blog/page.html"
[extra]
author = "Guy Lewin"
[taxonomies]
tags = [
  "bluetooth",
  "blueutil",
  "bose",
  "controlplane",
  "iphone",
  "macbook",
  "power manager",
  "quietcomfort",
]
+++
I received QuietComfort 35 from work, and I loved it from the first moment I used it. It’s always connected to my work Mac, my personal Mac and to my iPhone.

But whenever I leave work / home and close my laptop lid - I would expect the seamless reaction for the headphones to switch automatically to my iPhone audio while disconnecting from my now-sleeping Mac.

The current behaviour is far from that. My Mac stays connected even with it’s lid shut in my backpack, and I have to manually open the Bose app on my phone and switch the Mac connection off.

I was looking for a solution to turn off Bluetooth when lid is shut - but sadly the only one I could find is [ControlPlane](https://www.controlplaneapp.com/), which apparently used to be an app that lets you perform actions on Mac system events (such as lid closed / opened). Sadly I gave it a try and it seems broken on newer Macs.

Today I found this amazing app - [Power Manager](https://www.dssw.co.uk/powermanager/). With this app, along with a small Bluetooth command line utility, my problem was solved easily.

You can download my Power Manager profile [here](https://www.dropbox.com/s/ax60yhy0anhn0qi/BluetoothSchedule.pm-schedule), please note it requires the installation of [blueutil](https://github.com/toy/blueutil), available via Homebrew. The profile basically runs `blueutil -p 0` before sleep (lid close), and `blueutil -p 1` on wake (lid open).
