+++
title = "Trashing macOS Server: Part 3 - Plex, Transmission, and Flexget"
date = 2018-12-09T15:09:00-05:00
tags = ["homelab", "apple"]
series = "Trashing macOS Server"
+++

A little while back I automated my anime and manga habit using Plex, Flexget, and Transmission on my macOS server. Configuring these services on Ubuntu was plenty easy, but there were a couple of differences compared to the macOS server.

{{% box %}}
This story is part of a series on migrating from macOS Server to Ubuntu Server.

You can find all of the other stories in the series [here](/series/trashing-macos-server).
{{% /box %}}

## A Quick Note

This post will be a bit vague in some areas. My assumption is that if you're reading this post, you have a general idea of how you want these tools to work. I'm mainly going to talk about how I did initial setup.

## Plex

[Plex](https://www.plex.tv/) is a self-hosted, cross-platform media streaming application. Set up your media library on your own server, and then consume it from a huge range of devices. I'm a big fan of Plex for my anime shows and movies.

### Setting Up

Setting up Plex on Ubuntu is pretty painless. First, download the latest version. You can find the latest download link on [Plex's website](https://www.plex.tv/media-server-downloads/#plex-media-server).

```sh
curl -L -o plexmediaserver_<version>_amd64.deb https://downloads.plex.tv/plex-media-server/<version>/plexmediaserver_<version>_amd64.deb
```

Then install it with `dpkg`:

```sh
sudo dpkg -i plexmediaserver_<version>_amd64.deb
```

After that, we need to set up some firewall rules:

`sudo EDITOR /etc/ufw/applications.d/plexmediaserver`

```ini
# Config assumes Avahi is enabled already on 5353/udp, otherwise add it here
# Config omits some features I don't use
# See full list here: https://support.plex.tv/articles/201543147-what-network-ports-do-i-need-to-allow-through-my-firewall/

[plexmediaserver]
title=Plex Media Server
description=Plex Media Server
ports=32400/tcp|1900/udp|32410/udp|32412:32414/udp|32469/tcp
```

Last, we need to set up some ACLs for where the media is stored. In my case, I have an SMB share called `Plex Media`.

```sh
cd /media/Shares
# Set default ACL for new items, and apply that ACL. (The ACL will allow the `plex` user to read Plex Media directory and traverse all sub directories, but not execute files inside of it unless they are marked executable for some other user)
setfacl -Rdm user:plex:rX Plex\ Media/
setfacl -Rm user:plex:rX Plex\ Media/
```

### Migrating an Existing Library

If you have an existing Plex library you'd like to migrate, follow [Plex's guide](https://support.plex.tv/articles/201370363-move-an-install-to-another-system/) for migrating from one system to another.

As part of this step, I set up my Plex database on my `/media/Shares` mount (in a folder that is not actually shared), and symlinked that to `/var/lib/plexmediaserver/Library/Application\ Support/Plex\ Media\ Server`.

My reasoning for the symlink was twofold:
1. My database is growing ever larger, so I wanted it on a larger disk rather than on the boot drive
2. This location would be more convenient to grab in local backups

At this point, you should have your old library set up on your new server!

## Transmission

[Transmission](https://transmissionbt.com) is a cross-platform BitTorrent client. The Mac version is fantastic, and it has a couple of automation features around seeding that I haven't had luck with on other clients. Mainly for the latter reason, I chose to Transmission on my Ubuntu server.

### Install

Install the headless Transmission daemon:

```sh
sudo apt install transmission-daemon
```

### Configure

Guess what? More firewall rules!

This config assumes you're using port `9091` for the web UI. Replace `<peer_port>` with the port on which Transmissions will listen for peers. You might want to configure Transmission not to randomize the port on launch.

`sudo EDITOR /etc/ufw/applications.d/transmission`

```ini
[transmission]
title=Transmission
description=Transmission torrent daemon
ports=9091/tcp|<peer_port>
```

And then of course:

```sh
sudo ufw allow transmission
```

Guess what else? Permissions!

I save my downloads in my `Public` share under the directory `0 Server Torrents` (I like it at the top of the directory when sorted alphabetically).

```sh
cd /media/Shares
chmod g+s Public
setfacl -Rdm o::--- Public/
setfacl -m user:debian-transmission:rx Public
cd Public
mkdir 0\ Server\ Torrents
setfacl -Rdm user:debian-transmission:rwX 0\ Server\ Torrents
setfacl -Rm user:debian-transmission:rwX 0\ Server\ Torrents
```

From there, you'll probably want to edit Transmission's configuration:

```sh
sudo systemctl stop transmission-daemon
sudo EDITOR /etc/transmission-daemon/settings.json
# Do your thing
sudo systemctl enable transmission-daemon
sudo systemctl start transmission-daemon
```

### Removing Finished Torrents

If you're like me, you'd like finished torrents to automatically disappear from Transmission. This feature is built into the Mac app, but oddly not the Linux version. A common approach I saw was a timed job that uses `transmission-remote` to clean up.

First, set up a systemd service. This service will look for torrents that are 100% downloaded and done seeding and remove them.

`sudo EDITOR /lib/systemd/system/transmission-cleanup.service`

```ini
[Unit]
Description=Clean up finished torrents in Transmission

[Service]
Type=oneshot
ExecStart=/bin/sh -c "transmission-remote -l | grep 100% | grep ' Finished  ' | awk '{print $1}' | xargs -n 1 -I \% /usr/bin/transmission-remote -t \% -r"

[Install]
WantedBy=multi-user.target
```

Then set up a timer to run that service. Mine runs every hour.

`sudo EDITOR /lib/systemd/system/transmission-cleanup-hourly.timer`

```ini
[Unit]
Description=Clean up finished torrents in Transmission every hour
After=network.target

[Timer]
# Time between running
OnCalendar=hourly
Unit=transmission-cleanup.service

[Install]
WantedBy=multi-user.target
```

Lastly, start the timer!

```sh
sudo systemctl daemon-reload
sudo systemctl enable transmission-cleanup.service transmission-cleanup-hourly.timer
sudo systemctl start transmission-cleanup-hourly.timer
```

## Flexget

[Flexget](https://flexget.com) is a tool to help automate various things about your media library.

### Install

I'm not hugely familiar with the "right way" to do things in Python, so this may not be a great way of doing it, but it works fine for my needs, and I think is secure enough.

First, some initial work. We need `pip` for Python 3, and we need a user to run flexget:

```sh
sudo apt install python3-pip
sudo adduser srv-flexget
```

After that...permissions! I have a `flexget` directory in my `Public` share for flexget's config, and then `flexget` needs to be able to read where torrents end up.

```sh
cd /media/Shares
setfacl -m user:srv-flexget:rx Public/

cd Public
mkdir flexget
setfacl -dm user:srv-flexget:rwx flexget/
setfacl -m user:srv-flexget:rwx flexget/

setfacl -Rdm user:srv-flexget:rx 0\ Server\ Torrents/
setfacl -Rm user:srv-flexget:rx 0\ Server\ Torrents/
```

Then it's time to install flexget, which we will do as the `srv-flexget` user:

```sh
su -l srv-flexget
cd

# You may want other packages here
# For instance, I also use python-telegram-bot
pip3 install --user flexget transmissionrpc

# Here's where I link the config directory over to the Public share
# Feel free to omit this
ln -s /media/Shares/Public/flexget .flexget

EDITOR ~/.flexget/config.yml
# Insert your config
# Optional: If you are migrating an existing flexget install, you can copy the database over to maintain "seen" entries
flexget --test check
flexget --test execute

exit
```

### Scheduling with systemd

I have flexget configured to run once and exit. I let systemd handle the scheduling aspect of things.

First, we need a service that will run flexget:

`sudo EDITOR /lib/systemd/system/flexget.service`

```ini
[Unit]
Description=Flexget

[Service]
Type=oneshot
User=srv-flexget
Group=srv-flexget
ExecStart=/home/srv-flexget/.local/bin/flexget --loglevel info execute

[Install]
WantedBy=multi-user.target
```

Then make a timer. I have mine run every 15 minutes.

`sudo vim /lib/systemd/system/flexget.timer`

```ini
[Unit]
Description=Run Flexget every 15 minutes
After=network.target

[Timer]
# Time to wait after booting before we run first time
# OnBootSec=1min # If you do not enable flexget.service, uncomment this line
# Time between running each consecutive time
OnUnitActiveSec=15min
Unit=flexget.service

[Install]
WantedBy=multi-user.target
```

After that, enable and start things:

```sh
# This isn't required, but it gets log messages into journalctl. It will make the .service run at boot, so do not combine with OnBootSec in the .timer
sudo systemctl enable flexget

sudo systemctl enable flexget.timer
sudo systemctl start flexget.timer
```

## A Problem...

You now have an automated way to get your favorite media and load it into Plex. But now we have a bit of an issue. We have all kinds of important data on our server, but no nice way to keep it backed up. On a macOS server, you'd probably just use Time Machine.

I use Proxmox to keep the base VM[^1] backed up, but I only do that once a week. For all the shared files, I wanted to have hourly backups like Time Machine.

Next time, I'll talk about what I did to solve this problem!

[^1]: I only include the boot drive in the backup because my Proxmox backup disk is small! Besides, I found a way to do hourly backups of my shared files, so why include them in the Proxmox backup anyway?
