+++
title = "MatrixPortal Concierge"
description = "Building an always-on RGB LED info panel for the home — weather, what to wear, and now-playing music — using the Adafruit Matrix Portal S3, CircuitPython, and Home Assistant."
date = 2026-04-26
template = "blog/page.html"
[extra]
author = "Guy Lewin"
[taxonomies]
tags = [
  "circuitpython",
  "adafruit",
  "matrix-portal",
  "led-matrix",
  "home-assistant",
  "weather",
  "esp32",
]
+++

I've been wanting a tiny always-on display somewhere in our apartment - something I can look at before leaving the house without needing my phone. After spotting the [Adafruit Matrix Portal S3](https://www.adafruit.com/product/5778) + their [64×32 RGB LED matrix](https://www.adafruit.com/product/2276), I had an excuse to finally build one.

## The Problem

Every morning, the same routine: I check the weather on my phone, and decide how many layers I need. In NYC, every day can be different, and I wanted something to just tell me what to wear.

## The Solution

A 64×32 panel is small (just 2,048 pixels), so the display has to be deliberate. I went with two compact rows of information:

- A temperature range for the day (low–high)
- A weather condition icon (sun / rain / snow)
- Up to four clothing icons - coat, jacket, long sleeves, long pants, umbrella, hat - chosen by configurable temperature and precipitation thresholds

Weather data comes from [Open-Meteo](https://open-meteo.com/) (free, no API key, very reasonable). The clothing logic is just a handful of `if temp < threshold` rules - exactly what my brain does when looking at the Weather app.

![Weather display](https://github.com/GuyLewin/matrixportal-concierge/raw/main/readme_images/weather.png)

## The Expansion: Now Playing

Once it was up on the wall, I realized the panel was idle 99% of the time. So I added a second mode: POST a song and artist over HTTP, and the display switches to a scrolling now-playing view with a music note icon. The font is fixed-width and the panel only fits ~12 characters per line, so anything longer scrolls sideways on a loop until it fits. After 30 minutes (configurable) it reverts to weather automatically.

![Now-playing display](https://github.com/GuyLewin/matrixportal-concierge/raw/main/readme_images/media.png)

The integration with Home Assistant is just a `rest_command` triggered by a state automation on my media player entity - any time `media_title` changes while playing, it pushes the new song to the panel. Stops playback automatically clears it.

## Home Assistant Integration

The board exposes a small HTTP API: `/on`, `/off`, `/status`, `/data`, `/media`, `/media/stop`, `/media/status`. That's enough to wire it into Home Assistant as a switch (with a time-of-day automation that turns it off at night) and a few sensors (current temperature range, weather condition, clothing list). I also have an automation that turns the panel off when the TV turns on - the bright RGB matrix in our peripheral vision is distracting during a movie - and back on when the TV goes off. Full configuration examples are in the README.

## The Outcome

It's been on the wall for a few months now. My wife and I check it before heading out or when wondering "what song is this?", and I get to feel smug about this actually being useful.

You can find the complete code, hardware notes, and Home Assistant integration on GitHub: [https://github.com/GuyLewin/matrixportal-concierge](https://github.com/GuyLewin/matrixportal-concierge).
