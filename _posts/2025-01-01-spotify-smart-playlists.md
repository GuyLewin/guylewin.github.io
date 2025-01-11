---
title: 'Spotify Smart Playlists'
date: '2025-01-01T16:38:57-05:00'
author: Guy Lewin
layout: post
tags:
  - spotify
  - playlist
  - smart playlists
  - recently liked
---

Happy New Year! I'm excited to share some projects I've been working on over the past few years, starting with this one.

# Spotify Smart Playlists - Recently Liked Songs
## The Problem
As a music lover and long-time Spotify user (since 2015!), I've collected over 3,500 liked songs. However, my 128GB iPhone couldn't download them all for offline listening. I had to manually create smaller playlists for situations like flights or camping trips without service.

## The Inspiration
I remembered the [Smart Playlists feature from iTunes](https://support.apple.com/en-mo/guide/itunes/itns3001/windows), which allowed for dynamic playlist creation based on user-defined criteria. Searching for a similar solution in Spotify, I stumbled upon [Smarter Playlists](http://smarterplaylists.playlistmachinery.com/). Unfortunately, I couldn't get periodic runs to work and was very hesitant to share my Spotify tokens over HTTP (a security concern, especially in 2025).

## The Solution
I decided to create my own Python-based solution, leveraging an always-on Linux laptop at home. You can find the code and setup instructions on GitHub: [https://github.com/GuyLewin/spotify-recent-songs](https://github.com/GuyLewin/spotify-recent-songs). This code automatically maintains a playlist with my 1,500 most recently liked songs.

## The Outcome
With this playlist, I could keep my recently liked songs downloaded without consuming too much space. Although I've since upgraded to a new phone with more storage - I hope it helps others facing similar challenges.