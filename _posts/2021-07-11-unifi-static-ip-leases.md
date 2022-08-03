---
title: 'UniFi Static IP Leases'
date: '2021-07-11T19:51:56+00:00'
author: Guy Lewin
layout: post
permalink: /unifi-static-ip-leases/
tags:
    - dhcp
    - ip
    - jq
    - ssh
    - static
    - ubiquiti
    - unifi
---

In order to organize my UniFi-controlled network, I tried to look at all the static IP allocations I made using the UniFi portal.

Some lookups online suggested I use the "Insights" page on the UniFi portal, but I get a "No WiFi Networks found" error when I do:

![](/assets/images/posts/unifi-static-ip-leases/1.png)

I figured there’s a way to get the info through SSHing into the machine itself. Running `grep` recursively throughout the filesystem made me find `/config/ubios-udapi-server/ubios-udapi-server.state` which is a large JSON file containing device configuration. The DHCP static leases were listed under `services` -&gt; `dhcpServers` -&gt; `staticLeases`. I wrote this small one-liner to retrieve the mapping as JSON array:

```bash
cat /config/ubios-udapi-server/ubios-udapi-server.state | jq '[.services.dhcpServers[0].staticLeases[] | {ip: .addresses[0], mac: .id}] | sort_by(.ip | split(".") | map(tonumber))'
```

 The output of this script is a JSON array of objects containing `ip` and `mac`, for example:

```json
[
  {
    "ip": "192.168.1.2",
    "mac": "00:11:22:33:44:55"
  },
  {
    "ip": "192.168.1.3",
    "mac": "aa:bb:cc:dd:ee:ff"
  }
]
```

Eventually I learned that the old UniFi network UI has a working Insights page containing all the static leases. In order to view this page, I had to go to the "System Settings" tab within the Network settings page and disable "New User Interface":

![](/assets/images/posts/unifi-static-ip-leases/2.png)

Once I did that, I could visit the old Insights page and select the following filters to view all assigned static IP leases:

![](/assets/images/posts/unifi-static-ip-leases/3.png)