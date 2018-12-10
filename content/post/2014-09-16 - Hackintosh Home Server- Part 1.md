+++
title = "Hackintosh Home Server: Part 1"
date = 2014-09-16T14:41:00Z
tags = ["homelab", "apple", "tech"]
series = "Hackintosh Home Server"
slug = "hackintosh-home-server-part-1"
+++

I've been using Windows 8 on the server for a while now. It worked fine for what we needed. I didn't care for the setup much, but I didn't need to mess with it much either, so I got along fine with it overall.

***But then I tried to host a website on it...***

I grabbed XAMPP for Windows and got to work migrating my website over. That wasn't really an issue. However, the way I generate blog posts for Jekyll involves running a simple script that I wrote that moves some files around and invokes `jekyll`, a Ruby gem...

Getting Ruby and the Jekyll gem installed on the server was OK, but it's easier on OS X or Linux. Still, all I had to do now was get my script to work remotely, since I generally write my blog posts away from home. Unfortunately, I never got this to work due to permissions issues related to how Cygwin & OpenSSH authenticated with public key auth on Windows. Back to the drawing board.

I weighed my options. I needed iTunes for Home Sharing, some kind of web server, some kind of file sharing that works with OS X and Windows, an easy-to-configure VPN server, and a universal remote backup solution. Linux could do everything except iTunes, and Windows was out, which left me with OS X...except I wasn't about to replace my server hardware with a brand new computer.

*There was only one thing left to do.*
