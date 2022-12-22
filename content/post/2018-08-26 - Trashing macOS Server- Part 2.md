+++
title = "Trashing macOS Server: Part 2 - Time Machine"
date = 2018-08-26T14:34:00-05:00
tags = ["homelab", "apple"]
series = "Trashing macOS Server"
+++

I'm a big fan of having my computers make Time Machine backups to my server—it's more convenient than fumbling around with an external drive. Getting Time Machine to back up to a non-Apple machine requires a little bit of work, though.

{{% box %}}
This story is part of a series on migrating from macOS Server to Ubuntu Server.

You can find all of the other stories in the series [here](/series/trashing-macos-server).
{{% /box %}}

## A Few Notes

There are quite a few guides out there for getting Time Machine to talk to a Linux machine. I followed was [Sam Hewitt's excellent blog post](https://samuelhewitt.com/blog/2015-09-12-debian-linux-server-mac-os-time-machine-backups-how-to).

It's also worth noting that Apple supports Time Machine over SMB, and Samba 4.8 has support for Time Machine. Unfortunately, Ubuntu 16.04 doesn't have Samba 4.8, and even Ubuntu 18.04 does not. You can, however, [install Samba from source](https://www.reddit.com/r/homelab/comments/83vkaz/howto_make_time_machine_backups_on_a_samba/) if you'd rather avoid using AFP/Netatalk. I'll be trying in the future, but I don't want a custom Samba install for my main file server, and Netatalk is working fine for me at the moment.

## Installing Netatalk

We'll need to compile Netatalk from source. I had originally hoped to do this on another machine and move the built version onto the server, but it didn't work out for whatever reason (it's been a while, so I don't remember what went wrong). You can try doing that if you'd like, but this guide will just have you compile directly on the server.

First, you'll need to install some dependencies:
```sh
sudo apt install build-essential devscripts debhelper cdbs autotools-dev dh-buildinfo libdb-dev libwrap0-dev libpam0g-dev libcups2-dev libkrb5-dev libltdl3-dev libgcrypt11-dev libcrack2-dev libavahi-client-dev libldap2-dev libacl1-dev libevent-dev d-shlibs dh-systemd
```

After that, you can clone the source code and build it:
```sh
mkdir -p ~/work/netatalk-debian
cd ~/work/netatalk-debian
git clone https://github.com/adiknoth/netatalk-debian
cd netatalk-debian
debuild -b -uc -us
```

At this point, you should have some new debs in the directory above that you can install:
```sh
cd ..

# substitute <whatever> for the version of Netatalk you built
sudo dpkg -i libatalk<whatever>.deb
sudo dpkg -i netatalk<whatever>.deb
```

Now let's check if everything got installed right:
```sh
netatalk -V
afpd -V
```

You should see some basic information from these commands about what features are supported, etc. This guide assumes you already have Avahi installed from the previous post about Samba. If that's not the case, you might need to install a few more packages:

```sh
sudo apt install avahi-daemon libc6-dev libnss-mdns
```

## Configuring Netatalk

To configure Netatalk, you'll want to sort out where to _put_ the Time Machine backups first. On my machine, that looked something like this:

```sh
sudo mkdir /media/NetworkBackup
sudo chown server:shares-access /media/NetworkBackup
sudo chmod 770 /media/NetworkBackup
```

Next, go edit the Netatalk config file.
`sudo EDITOR /etc/netatalk/afp.conf`
```ini
## Comment out [homes] stuff! ##

[Backups]
time machine = yes
path = /media/NetworkBackup/Time Machine
; Size limit in MiB
vol size limit = 1000000
valid users = @shares-access
```

You'll notice I used allowed my `shares-access` group to connect to the Time Machine mount. You might need to change this to fit your needs.

After that, you'll need to create a new application config for UFW.
`sudo EDITOR /etc/ufw/applications.d/netatalk`
```ini
[netatalk]
title=netatalk
description=AFP server
ports=548/tcp
```

Then you can allow it through the firewall:
```sh
sudo ufw allow netatalk
```

Lastly you'll want to set up Avahi so that your clients know about the server.
`sudo EDITOR /etc/avahi/services/afpd.service`
```xml
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
 <name replace-wildcards="yes">%h</name>
 <service>
   <type>_afovertcp._tcp</type>
   <port>548</port>
 </service>
 <service>
   <type>_device-info._tcp</type>
   <port>0</port>
   <txt-record>model=RackMac</txt-record>
 </service>
</service-group>
```

After that, we just need to restart Avahi and fire up Netatalk!

```sh
sudo systemctl restart avahi-daemon
sudo systemctl enable netatalk
sudo systemctl start netatalk
```

## Connect to Netatalk from Time Machine

If all went well, you should be able to open Time Machine preferences on your Mac, click "Select Disk", and see your server and connect using an account that you granted access in your Netatalk config (in my case, that's anybody in the `shares-access` group).

If you can't connect to your server, you might need to tell Time Machine to allow using "unsupported" volumes like so:

```sh
defaults write com.apple.systempreferences TMShowUnsupportedNetworkVolumes 1
```

## A Word of Caution

Apple deprecated AFP a while back, and APFS doesn't support sharing over AFP. How long Apple is going to keep AFP around is anyone's guess. My point is, don't expect this to work forever. 😅

I have my fingers crossed it'll work until 2020, when the next LTS version of Ubuntu comes out (which _hopefully_ will have Samba >= 4.8). Worst case, I'll have to set up a second VM just for Time Machine backups if I don't want to run a custom Samba install on my main server.
