---
title: 'Iodine and Paid-Access Hotspot Situations'
date: '2019-01-19T14:20:06+00:00'
author: Guy Lewin
layout: post
permalink: /iodine-and-paid-access-hotspot-situations/
tags:
    - dns
    - facebook
    - hotspot
    - iodine
    - messenger
    - reddit
    - wifi
---

I travel a lot and find myself in many situations where I’m connected to a hotspot but have to pay to get access to the Internet.

If you haven’t heard about [iodine](https://code.kryo.se/iodine/) yet, it’s a solution that lets you tunnel IP packets over valid DNS requests. This is a solution just for these problems, as most paid-access hotspots allow valid DNS-requests and responses to go through.

There are many guides on setting up an iodine setup, including [this one](https://demgeeks.com/hack-get-free-wifi-on-paid-access-hotspots/) for example. The problem is - the established connection is usually too slow to actually do something with it, from my experience.

iodine lets you SSH into the installed iodine server (over DNS requests/responses). Usually people setup an SSH tunnel and use their personal computer regularly.

If that is too slow for you (like it is for me) - I recommend installing a bunch of utilities **on the server itself** so the server does all massive Internet traffic, and you just get the output through the SSH shell.

My setup is a cheap $10-a-month DigitalOcean server with the following programs installed:

- Reddit client for Terminal - <https://github.com/michael-lazar/rtv>
- Facebook Messenger client for Terminal - <https://github.com/mjkaufer/Messer>

Since I installed these I don’t need to create the SSH tunnel anymore, I just run them from the SSH shell and enjoy a relatively fast way of communicating for free.