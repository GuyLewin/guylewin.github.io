+++
title = "Bathroom Smart Speaker using UE Boom, Raspberry Pi, Spotify and Home Assistant"
description = "Building a smart speaker system that can play music on demand from Home Assistant and support Spotify Connect connectivity using a Raspberry Pi and Bluetooth speaker."
date = 2021-09-16
aliases = ["2021/09/16/bathroom-smart-speaker-using-ue-boom-raspberry-pi-spotify-and-home-assistant.html", "/bathroom-smart-speaker-using-ue-boom-raspberry-pi-spotify-and-home-assistant/"]
template = "blog/page.html"
[extra]
author = "Guy Lewin"
[taxonomies]
tags = [
  "home assistant",
  "pi-btaudio",
  "raspberry pi",
  "raspotify",
  "spotcast",
  "spotify",
  "ue boom",
]
+++
## Goal

We’re going to build a smart speaker system that can play tropical forest sounds (or any other Spotify playlist) on demand from Home Assistant, and support Spotify Connect connectivity to cast music from other Spotify clients (if you want to put your own music while showering).

## Why?

Who doesn’t like showering with music? And if you already put a speaker in your bathroom, why not put some tropical background sounds while you’re at it?

## Hardware Requirements

I’m using the following hardware for my setup:

- [UE ](https://www.ultimateears.com/en-us/wireless-speakers/boom-3.html)[Boom](https://www.ultimateears.com/en-us/wireless-speakers/boom-3.html) (I’m actually using UE Boom 2 but any UE Boom would work).
- [Raspberry Pi Zero W](https://www.raspberrypi.org/products/raspberry-pi-zero-w/) (any Raspberry Pi with Bluetooth will work. Make sure it’ll also have WiFi since most bathrooms don’t have an Ethernet port).
    - microSD card and power adapter for the Raspberry Pi.
- Optional: [Aqara Motion Sensor](https://www.aqara.com/us/motion_sensor.html) + [ConBee II](https://www.phoscon.de/en/conbee2) (if you want to turn music on / off based on motion).

## Additional Requirements

- [Home Assistant](https://www.home-assistant.io/) installation in LAN.
- Spotify Premium account.

## Steps

### Preparing the Raspberry Pi

Install [Raspberry Pi OS](https://www.raspberrypi.org/software/) on your device by using software such as [Raspberry Pi Imager](https://www.raspberrypi.org/documentation/computers/getting-started.html#using-raspberry-pi-imager). I chose to install the "Raspberry Pi OS with desktop" flavor.

### Installing Raspotify

Once the RPi is installed and connected to your local network, connect via SSH and run the following command to install [Raspotify](https://github.com/dtcooper/raspotify) - the Spotify connect server for Raspberry Pi:

```bash
# Install Raspotify
curl -sL https://dtcooper.github.io/raspotify/install.sh | sh
```

After installation, edit `/etc/default/raspotify` to change some parameters:

- Uncomment (remove the # at the beginning) the line that starts with `DEVICE_NAME` and name your Spotify connect server (e.g. `Bathroom Speaker`)
- Uncomment (remove the # at the beginning) the line that starts with `OPTIONS` in order to specify your Spotify premium credentials. This is crucial to allow Home Assistant to play music on this speaker

The file should look like this (**while changing every `<>` parameter**):

```
# /etc/default/raspotify -- Arguments/configuration for librespot

# Device name on Spotify Connect
DEVICE_NAME="<Speaker Name>"

# The displayed device type in Spotify clients.
# Can be "unknown", "computer", "tablet", "smartphone", "speaker", "tv",
# "avr" (Audio/Video Receiver), "stb" (Set-Top Box), and "audiodongle".
#DEVICE_TYPE="speaker"

# Bitrate, one of 96 (low quality), 160 (default quality), or 320 (high quality)
#BITRATE="160"

# Additional command line arguments for librespot can be set below.
# See `librespot -h` for more info. Make sure whatever arguments you specify
# aren't already covered by other variables in this file. (See the daemon's
# config at `/lib/systemd/system/raspotify.service` for more technical details.)
#
# To make your device visible on Spotify Connect across the Internet add your
# username and password which can be set via "Set device password", on your
# account settings, use `--username` and `--password`.
#
# To choose a different output device (ie a USB audio dongle or HDMI audio out),
# use `--device` with something like `--device hw:0,1`. Your mileage may vary.
#
OPTIONS="--username '<Put Spotify Premium Username Here>' --password '<Put Spotify Premium Password Here>'"

# Uncomment to use a cache for downloaded audio files. Cache is disabled by
# default. It's best to leave this as-is if you want to use it, since
# permissions are properly set on the directory `/var/cache/raspotify'.
#CACHE_ARGS="--cache /var/cache/raspotify"

# By default, the volume normalization is enabled, add alternative volume
# arguments here if you'd like, but these should be fine.
#VOLUME_ARGS="--enable-volume-normalisation --volume-ctrl linear --initial-volume=100"

# Backend could be set to pipe here, but it's for very advanced use cases of
# librespot, so you shouldn't need to change this under normal circumstances.
#BACKEND_ARGS="--backend alsa"
```

### Installing pi-btaudio

pi-btaudio is a suite of packages that allow your RPi to connect to UE Boom automatically, and make sure the connection stays active.

Start by going over the [prerequisites](https://github.com/bablokb/pi-btaudio#prerequisites), make sure to write down the MAC address of your UE Boom. In order to connect to a UE Boom, you must press its Bluetooth pairing button for it to accept new connections.

Once you’re done, follow [their installation steps](https://github.com/bablokb/pi-btaudio#installation) to install the tools. Edit the configuration file in `/etc/asound.conf` with the following content (**make sure you replace `<UE Boom Bluetooth MAC>` with the one found during prerequisites**):

```
pcm.!default "bluealsa"
ctl.!default "bluealsa"
defaults.bluealsa.interface "hci0"
defaults.bluealsa.device "<UE Boom Bluetooth MAC>"
defaults.bluealsa.profile "a2dp"
```

### Raspotify Watchdog Restarter

Providing credentials as options to Raspotify ensures Home Assistant will be able to connect and play music remotely. But after someone else uses Spotify Connect to cast music into Raspotify - it’s unable to reconnect using the credentials provided in the configuration file.

To solve this, I wrote a small API server in Python to restart Raspotify once I want to turn it off (when motion sensor turns off).

Start by creating a `~/watchdog` directory and placing the following content in `~/watchdog/server.py` (**replace `<Secret Restart Password QueryString>` with some long string that should be kept secret**):

```python
import os
from http.server import HTTPServer, BaseHTTPRequestHandler

class MyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path != "/?<Secret Restart Password QueryString>":
            self.send_response(401)
            self.end_headers()
            self.wfile.write(bytes("Bye bye", "utf-8"))
            return
        # send 200 response
        self.send_response(200)
        # send response headers
        self.end_headers()
        # send the body of the response
        os.system("sudo systemctl restart raspotify.service")
        self.wfile.write(bytes("Restarted", "utf-8"))

httpd = HTTPServer(('', 9876), MyHandler)
httpd.serve_forever()
```

Then create the file `/lib/systemd/system/restart_raspotify.service` with the following content:

```
[Unit]
Description=Restart raspotify service
After=multi-user.target
Conflicts=getty@tty1.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/pi/watchdog/server.py
StandardInput=tty-force

[Install]
WantedBy=multi-user.target
```

To enable the newly created service, run:

```bash
systemctl enable restart_raspotify.service
systemctl start restart_raspotify.service
```

### Spotcast on Home Assistant

Spotcast enables Home Assistant to play music on Spotify Connect-enabled devices, like our Raspotify system. Follow the [installation steps](https://github.com/fondberg/spotcast#installation) to install on your Home Assistant instance. After installing, you need to configure Home Assistant to authenticate using your Spotify Premium account by following the [configuration steps](https://github.com/fondberg/spotcast#configuration).

After restarting Home Assistant, you’ll be able to use Spotcast to play music on your speaker through Raspotify by calling this Home Assistant service (**replace `<Speaker Name>` with what you specified in `/etc/default/raspotify`, and `<Spotify Playlist URI>` with the Spotify-formatted URI such as `spotify:album:2PPfl28ysMbOkl2DBJ5Dr4` - a good rainforest nature album**):

```yaml
data:
  device_name: "<Speaker Name>"
  random_song: true
  uri: "<Spotify Playlist URI>"
  start_volume: 15
  force_playback: true
service: spotcast.start
```

### Raspotify Restart via Home Assistant

Edit Home Assistant’s `configuration.yaml` file (by [installing the File Editor](https://www.home-assistant.io/getting-started/configuration/) or by any other method) and insert the following to the configuration (**replace &lt;Secret Restart Password QueryString&gt; with what you chose in `~/watchdog/server.py`**):

```yaml
rest_command:
  restart_raspotify:
    url: "http://<Raspberry Pi IP>:9876/?<Secret Restart Password QueryString>"
```

After another Home Assistant restart, you’ll be able to invoke the following service to restart Raspotify and make it re-authenticate using the configured credentials:

```yaml
service: rest_command.restart_raspotify
```

### Connecting It All Together in Home Assistant

I configured an automation to trigger when the motion sensor is activated - it will play the rainforest nature album (`spotify:album:2PPfl28ysMbOkl2DBJ5Dr4`). When the motion sensor is deactivated - it runs restart\_raspotify to reset Raspotify to the configured credentials and be ready for the next person to walk in.

If anyone wants the full Home Assistant setup, write a comment and I’ll upload that as well.

### Adding AirPlay Support

Follow the instructions in the [follow-up post](https://lewin.co.il/bathroom-smart-speaker-part-2-airplay-to-bluetooth-speaker-via-raspberry-pi/).

## Bonus - Turning UE Boom On via Raspberry Pi

UE Boom is listening for Bluetooth commands even when turned off. You can use your phone (if paired with the speaker) to turn the speaker on remotely. You can do the same with Raspberry Pi after it’s connected. To do so, first install NodeJS (I chose version 14.15.4 while writing this, there’s probably a newer version available when you’re reading this). There’s no official NodeJS release for Raspberry Pi Zero W, but this unofficial installation works great for me:

```bash
#!/bin/bash
export NODE_VER=14.15.4
if ! node --version | grep -q ${NODE_VER}; then
  (cat /proc/cpuinfo | grep -q "Pi Zero") && if [ ! -d node-v${NODE_VER}-linux-armv6l ]; then
    echo "Installing nodejs ${NODE_VER} for armv6 from unofficial builds..."
    curl -O https://unofficial-builds.nodejs.org/download/release/v${NODE_VER}/node-v${NODE_VER}-linux-armv6l.tar.xz
    tar -xf node-v${NODE_VER}-linux-armv6l.tar.xz
  fi
  echo "Adding node to the PATH"
  PATH=$(pwd)/node-v${NODE_VER}-linux-armv6l/bin:${PATH}
fi
```

After installing NodeJS, run the following script whenever you want to turn your UE Boom on. **Make sure you replace 00:11:22:33:44:55 with your UE Boom’s Bluetooth MAC address, and AABBCCDDEEFF to your Raspberry Pi Zero W’s Bluetooth MAC address, the same client address you used for pairing.**

```bash
#!/bin/sh

set -ue

HANDLE=0x0003
VALUE=AABBCCDDEEFF01

MAC=00:11:22:33:44:55

gatttool -b $MAC --char-write-req --handle=$HANDLE --value=$VALUE
```
