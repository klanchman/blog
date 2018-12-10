+++
title = "Hackintosh Home Server: Part 2"
date = 2014-10-13T02:53:00Z
tags = ["homelab", "apple", "tech"]
series = "Hackintosh Home Server"
aliases = ["/hackintosh-home-server-part-2"]
+++

(Oh wow, it's already been nearly a month since my last post? So I've been busy. Sue me.)

As we know from last time, I came to the conclusion that running OS X on the server was my only real option for what I wanted to do, but I didn't want to get new hardware. Usually if you want to run OS X on non-Apple hardware, you have to carefully pick components that will work *just* right with OS X. I didn't have OS X in mind when I picked out the components for this server, though.

Nevertheless, I figured I'd give it a try.

I grabbed a distro of OS X and proceeded to install it on the server. Imagine my surprise when it actually booted up! I was floored to say the least. However, being me, I managed to destroy it in less than an hour.

**Take 2**

I reinstalled and promptly made a Time Machine backup *before* I ran any updates. This time everything went without (much of) a hitch! Everything I needed working seemed to be fine. The biggest issue was that I was stuck at a resolution of 1280x1024 and didn’t have any graphics acceleration, since the version of Intel integrated graphics I have isn’t supported by OS X. Whatever. It just makes it really laggy to use screen sharing, but it’s not a huge deal.

So, by the end now, I have all sorts of goodies running on this thing. I have my typical array of things, like file sharing, a media server, a VPN server (which was amazingly easy to configure), and centralized backup (using Time Machine for the Macs and Crashplan for everything else!). However, I also got an easy-to-use web server, Git repository hosting, and other fun stuff! I’m loving it so far.

I just hope upgrading to Yosemite will be doable when the time comes…
