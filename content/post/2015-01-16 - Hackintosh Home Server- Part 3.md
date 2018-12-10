+++
title = "Hackintosh Home Server: Part 3"
date = 2015-01-16T16:04:00Z
tags = ["homelab", "apple", "tech"]
series = "Hackintosh Home Server"
aliases = ["/hackintosh-home-server-part-3"]
+++

I mentioned a few months ago that I was hoping the home server would be able to be upgraded to Yosemite when the time came. Over my winter break, I decided to give it a shot.

Until now, the server was happily (albeit slowly) running using an OS X distro. Max resolution with the integrated Intel HD 3000 Graphics was 1280x1024, since OS X never supported HD 3000. Doing anything on screen or with screen sharing was insanely slow, but it worked.

However, I happened to add an NVIDIA ION graphics card to my pile of decommissioned computer parts recently, so I decided I'd toss it into the server and see what happened. To my surprise, without any extra intervention, it worked! It was still a little slow, but it was running at 1920x1080 and screen sharing wasn't abysmal anymore. Neat!

But it was still on Mavericks, and every other Mac in the house was running Yosemite. That just won't do.

Unfortunately, I was not using Clover as my bootloader, so a straight upgrade to Yosemite was not really in the cards. I decided I'd do a Time Machine backup of the server, wipe it, and do a fresh install of Yosemite with Clover. Of course, that didn't go as planned.

I set up my OS X installer USB following [this forum post](http://www.tonymacx86.com/yosemite-desktop-guides/144426-how-install-os-x-yosemite-using-clover.html). It booted, but it would hang before actually getting to the installer. I tried the Unibeast method instead, which had never worked before. Rest assured that it still didn't work. I didn't expect it to, so I wasn't let down or anything.

I decided that I didn't want to put any more time into the thing, and I'd just restore my Time Machine backup I so cleverly made. So I popped in my old installer for the working distro and booted it up.

For some reason Time Machine Restore was not an option, unlike on every other OS X installer I've used.

*There I was. I had a useless backup, and a server without an operating system...*
