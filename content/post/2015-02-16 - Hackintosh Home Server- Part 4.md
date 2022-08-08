+++
title = "Hackintosh Home Server: Part 4"
date = 2015-02-16T03:05:00Z
tags = ["homelab", "apple"]
series = "Hackintosh Home Server"
+++

I left this off where I had a server with no OS, and a backup that was totally useless.

I decided trying to get the backup working was not worth my time, so I soldiered onwards trying to install Yosemite on the server. As luck would have it, the same distro I used for Mavericks now had a Yosemite version!

It took its good ol' time, but the installer eventually booted. I was able to get Yosemite installed, and I was even able to use the Clover bootloader with this installation. Hopefully any system updates later on will be able to done easily with Clover...

I had to remove a couple of kexts that were causing some bad behavior, and I had to add a kext for my ethernet adapter, but that was about all. The only other odd behavior was that the installer didn't seem to like my NVIDIA graphics card. I remedied this issue by simply using the onboard video during installation. Oddly, the NVIDIA card worked after installation...

Hackintosh computers are weird.
