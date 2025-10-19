+++
title = "TorrentLeechSeeder and DLink DSL-6740U File Upload"
description = "Automating file uploads to seed torrents using TorrentLeechSeeder with a D-Link DSL-6740U router, including router configuration and upload script automation."
date = 2018-02-17
aliases = ["2018/02/17/torrentleechseeder-and-dlink-dsl-6740u-file-upload.html", "/torrentleechseeder-and-dlink-dsl-6740u-file-upload/"]
template = "blog/page.html"
[extra]
author = "Guy Lewin"
+++

I recently worked on 2 useful mini-projects that I wanted to share, in case it’s useful to anyone.

## TorrentLeechSeeder

My home Internet connection is slow. Especially the upload. It was hurting my upload/download ratio in the popular BitTorrent tracker BitTorrent tracker [TorrentLeech](http://torrentleech.org).

I started choosing torrents to download by the amount of seeders / leechers they have, but doing this manually took too much time.

So I wrote a script that scrapes TorrentLeech searching for the most "efficient" torrents to download and then seed, using "aria2c" for downloading/seeding.

You can read more and clone the code at GitHub:

<https://github.com/GuyLewin/TorrentLeechSeeder.git>

## DLink DSL-6740U File Upload

As I previously mentioned - my home Internet connection is **\*very\*** slow.

I wanted to upload tcpdump to see what’s taking all my bandwidth, but there was no nc, wget or anything useful - only httpd.

So I wrote a small guide with snippets of how to upload any files (including the right statically-compiled tcpdump binary).

You can find it here on GitHub:

<https://github.com/GuyLewin/DSL-6740UFileUpload>
