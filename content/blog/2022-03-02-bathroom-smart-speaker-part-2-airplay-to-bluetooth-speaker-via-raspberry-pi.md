+++
title = "Bathroom Smart Speaker Part 2 - AirPlay to Bluetooth Speaker via Raspberry Pi"
description = "Adding AirPlay support to a smart speaker setup using shairport-sync, allowing Apple Music streaming and multi-room audio capabilities alongside existing Spotify Connect functionality."
date = 2022-03-02
aliases = ["2022/03/02/bathroom-smart-speaker-part-2-airplay-to-bluetooth-speaker-via-raspberry-pi.html", "/bathroom-smart-speaker-part-2-airplay-to-bluetooth-speaker-via-raspberry-pi/"]
template = "blog/page.html"
[extra]
author = "Guy Lewin"
[taxonomies]
tags = [
  "airplay",
  "apple music",
  "bluetooth",
  "home assistant",
  "raspberry pi",
  "shairport-sync",
  "spotify",
  "ue boom",
]
+++
In [part 1](https://lewin.co.il/bathroom-smart-speaker-using-ue-boom-raspberry-pi-spotify-and-home-assistant/) I wrote on how to create a smart speaker supporting Spotify Connect using a Raspberry Pi and a Bluetooth speaker. Since writing that post I started using Apple Music and wanted to enjoy simultaneous streaming of music to multiple speakers in my house. I wanted my bathroom speaker, which became smart by supporting Spotify Connect, to also be able to play Apple Music.

Apple Music casting uses [AirPlay](https://www.apple.com/airplay/) for casting audio between devices. It can be used to cast any sound from Apple devices, even when using Spotify, watching YouTube or having a phone call.

In order to add this capability to your smart speaker from the previous post - we’re going to use [shairport-sync](https://github.com/mikebrady/shairport-sync), which is an open-source audio player supporting AirPlay 1 (and AirPlay 2 partially, at the time of writing this post). The instructions in this post are partially based on [this GitHub comment by `bedrin`](https://github.com/mikebrady/shairport-sync/issues/200#issuecomment-520574102).

## Steps

### Installing shairport-sync

Execute the following commands from to install the most recent version of shairport-sync:

```bash
sudo apt-get update
sudo apt-get install build-essential git xmltoman autoconf automake libtool libpopt-dev libconfig-dev libasound2-dev avahi-daemon libavahi-client-dev libssl-dev libsoxr-dev
git clone https://github.com/mikebrady/shairport-sync.git
cd shairport-sync
autoreconf -fi
./configure --sysconfdir=/etc --with-alsa --with-soxr --with-avahi --with-ssl=openssl --with-systemd
make
sudo make install
```

### Giving shairport-sync Bluetooth Permissions

shairport-sync needs permissions to play sound over Bluetooth. Run these commands to add shairport-sync’s user (and `pi` for testing purposes) to the Bluetooth UNIX group which will permit it to play audio:

```bash
sudo adduser pi bluetooth
sudo adduser shairport-sync bluetooth
```

### Adding Additional ALSA Device

In the previous post we edited `/etc/asound.conf` to point to the Bluetooth speaker as the default device. shairport-sync requires a named device, so we’ll create another ALSA device alongside the default one to also map to the Bluetooth speaker.

Open `/etc/asound.conf` and copy the MAC address you filled after `defaults.bluealsa.device` (should be at line 4 in quotation marks). Replace the content of `/etc/asound.conf` with the following, while replacing `<UE Boom Bluetooth MAC>` with the MAC address you found in line 4:

```
pcm.!default "bluealsa"
ctl.!default "bluealsa"
defaults.bluealsa.interface "hci0"
defaults.bluealsa.device "<UE Boom Bluetooth MAC>"
defaults.bluealsa.profile "a2dp"

pcm.bathroom_bt {
 type plug
  slave {
    pcm {
      type bluealsa
      device "<UE Boom Bluetooth MAC>"
      profile "a2dp"
    }
  }
  hint {
    show on
    description "Bathroom BT"
  }
}
```

In the above configuration I named this device "bathroom\_bt" (line 7) with the description "Bathroom BT". You can replace this to fit your scenario, but make sure to replace it in the below shairport-sync configuration as well.

### shairport-sync Configuration

Now that the ALSA device is configured - we should configure shairport-sync to use this device as the sound output device. Replace the content of `/etc/shairport-sync.conf` with the following:

```
general =
{
	name = "Bathroom Speaker";
};

sessioncontrol =
{
	allow_session_interruption = "yes";
};

alsa =
{
	output_device = "bathroom_bt";
};
```

The `name` parameter under `general` (line 3) will be the displayed AirPlay name, set it to a suitable value. If you changed the name of the ALSA device (in `/etc/asound.conf` at line 7) change it under `output_device` at line 13 as well.

### Disable WiFi Power Management

Raspberry Pi’s WiFi can sometimes goes into power-saving mode, which can cause audio drops and glitches when acting as an AirPlay server. We can disable the WiFi power management by editing `/etc/rc.local` and adding the following line right before `exit 0` (if you followed my previous post - this should be at line 19):

```bash
iwconfig wlan0 power off
```

After adding this line, your `/etc/rc.local` should look like this:

```bash
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
fi

iwconfig wlan0 power off

exit 0
```

### Enable shairport-sync Service

Finalize your setup by enabling the `shairport-sync` service and performing a system reboot to load all the new configurations:

```bash
sudo systemctl enable shairport-sync
sudo reboot
```
