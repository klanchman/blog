+++
title = "Trashing macOS Server: Introduction"
date = 2018-04-15T15:00:00-05:00
tags = ["homelab", "apple", "tech"]
series = "Trashing macOS Server"
image = "/img/post-images/trashing-macos-server.png"
+++

A few years back I wanted a home server that would mesh well with my other Apple devices. I thought it'd be a good idea to use macOS Server. For the most part, it went well, but a few recent developments scared me into finding an alternative.

{{% box %}}
This story is part of a series on migrating from macOS Server to Ubuntu Server.

You can find all of the other stories in the series [here](/series/trashing-macos-server).
{{% /box %}}

## Backstory

My humble homelab started out with the lowest-end Mac mini (the $500 one) and a few external hard drive enclosures. I chose to run macOS Server because I figured it would "just work" with the rest of my Apple devices (it did). I chose to run it on a Mac mini because they are cheap and don't take up a ton of space.

The lowest-end Mac mini is abysmally slow, and Apple is notoriously bad about updating Mac mini hardware—at the time of writing, the last Mac mini hardware update was in October 2014. Eventually, the sluggishness of the Mac mini, and the lack of USB ports for adding more external storage, pushed me to buy a Dell PowerEdge T330 server. I bought it with eight 3.5" drive bays, 24 GB RAM, an upgraded CPU, and a plan: install ESXi, virtualize my macOS Server, and virtualize some other things while I was at it.

Thanks to my previous experience virtualizing macOS on vSphere with non-Apple hardware, my plan worked well for about a year. Then I saw some bad news.

## Reason 1: Unlocker

[Unlocker](https://github.com/DrDonk/unlocker) is an excellent project that allows VMware software to virtualize macOS guests on non-Apple hardware. I had been using the ESXi version with much success. I was looking into upgrading my ESXi installation one day, when I noticed something bad had happened: the Unlocker project had dropped ESXi support.

I completely understand the decision to drop support. VMware had been hardening ESXi with each release, making it more difficult to keep Unlocker working. Eventually, it got to the point where Unlocker could not reliably work on ESXi. This meant that I could either upgrade ESXi and lose my ability to run macOS, or continue running macOS and never again get so much as a security patch for ESXi.

## Reason 2: Apple's Commitment to macOS Server

Apple's [continued efforts to phase out macOS Server](https://support.apple.com/en-us/HT208312)[^1] had also been getting me worried. When I started using macOS Server, Apple was already showing their hand. The Server app hadn't had a real update in a while already. It received a visual overhaul at some point, but that was about it. Apple's removal of pieces of Server in fall 2017, and announcement of more planned removals for fall 2018, also served as an impetus for me to find an alternative

## Looking for Alternatives

It was time to ditch macOS Server in favor of something I could run on a Linux or Windows guest. At the time, my macOS VM was handling a handful of services on my home network:

- macOS Server
  - File sharing
  - Time Machine backups (both for devices on the network, as well as the server itself)
  - Caching Server
  - iTunes (as a central place to drop new music into my iCloud Music Library, and to keep a local copy of all my music)
  - NetInstall (faster than using Internet Recovery)
- Plex Media Server
- [Flexget](https://flexget.com) + Transmission
- GitLab Runner (continuous integration build server for iOS projects)

I can't build iOS projects on anything other than macOS, but that was also one of the least-used services I had configured on the VM. I could also live without Caching Server and NetInstall, though they would be nice to have. Time Machine backups were the most important question I had to answer. I also wanted to make sure that my file sharing configuration worked as frictionlessly as possible. I had previously experienced issues with sharing files from Windows and Linux servers, especially with macOS clients.

After much research and tinkering, I ended up with an Ubuntu Server 16.04 VM that could handle most of my use cases.

In future posts, I will cover how I configured each of the services!

[^1]: In case the link goes dead or changes: Apple removed Caching Server, File Sharing Server, and Time Machine Server from macOS Server in fall 2017, bundling that functionality directly into the OS. In fall 2018, Apple plans to remove "open source services such as Calendar Server, Contacts Server, the Mail Server, DNS, DHCP, VPN Server, [NetBoot/NetInstall,] and Websites" from macOS Server.
