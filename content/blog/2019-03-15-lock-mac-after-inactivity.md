+++
title = "Lock Mac After Inactivity"
description = "Configuring automatic screen locking on macOS after periods of inactivity using System Preferences and screensaver settings for improved security."
date = 2019-03-15
aliases = ["2019/03/15/lock-mac-after-inactivity.html", "/lock-mac-after-inactivity/"]
template = "blog/page.html"
[extra]
author = "Guy Lewin"
[taxonomies]
tags = [
  "configuration profile",
  "inactivity",
  "lock",
  "mac",
  "manual-screensaver-cron",
  "screensaver",
]
+++
Mac comes with a shortcut for locking the desktop session - `⌘ + Ctrl + Q`. It is also possible to [define another keyboard shortcut for that](https://maclovin.org/blog-native/2017/high-sierra-set-a-global-shortcut-to-lock-screen).

In my case, I wanted the screen to lock automatically after I’m idle for a certain period of time. That’s also easily configured by following these steps:

1. Launch System Preferences -&gt; Desktop &amp; Screen Saver -&gt; Screen Saver
2. Change the "Start after:" value in the left bottom corner to the requested idle time (in my case - 1 minute).
3. On System Preferences -&gt; Security &amp; Privacy -&gt; General - mark "Require password **immediately** after sleep or screen saver begins"

This was a good solution, but my workplace setup a group policy (by enforcing a configuration profile) that sets the "Start after" time in the screen saver pane to 20 minutes, which is *way* too long for me.

To solve that, I wrote this small cron task that checks your inactivity time, and launches the screen saver manually if enough time has passed. You can find the code + installation instructions [here](https://github.com/GuyLewin/manual-screensaver-cron).
