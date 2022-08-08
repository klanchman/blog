+++
title = "Trashing macOS Server: Extra 1 - Revisiting Time Machine"
date = 2019-12-23T11:49:00-05:00
tags = ["homelab", "apple"]
series = "Trashing macOS Server"
+++

I mentioned in my [post about Time Machine](/trashing-macos-server-part-2-time-machine) that Samba 4.8 supports Time Machine, but that it wasn't available on Ubuntu 16.04, so I went with Time Machine over AFP using Netatalk. After months of backing up over AFP, I started having some problems with backups becoming corrupt.

{{% box %}}
This story is part of a series on migrating from macOS Server to Ubuntu Server.

You can find all of the other stories in the series [here](/series/trashing-macos-server).
{{% /box %}}

## The Problem

After a while of backing up 3 computers to my server over AFP, one of the computers began showing a message saying that Time Machine had verified my backup and that it had to make a new backup. Once this message was shown, it refused to back up.

I found myself consulting [a StackExchange post](https://apple.stackexchange.com/questions/18482/repair-time-machine-sparsebundle-that-will-no-longer-mount) and a few blog posts, and after combining a few fixes, my computer happily began backing up again.

Within a month or so, it happened again. I fixed it again. Then a _different_ computer began showing the same message. Obviously there was a problem, and I didn't feel like `fsck`-ing all my backups on a monthly basis until 2020 when Samba 4.8 would arrive for Ubuntu LTS.

I felt like it would be a good idea to try Time Machine over SMB with Samba, but I wasn't about to fiddle with the Samba installation that handles all my shared files.

## A Dedicated Time Machine Server

When all of this was happening, Ubuntu 18.10 was available, and it had Samba 4.8. I decided to spin up a new VM that would be only for Time Machine backups (and Windows File History for the one Windows machine on the network). Setup for this was pretty similar to [setting up Samba for my shared files](/trashing-macos-server-part-1-file-server). Really, only one config change was needed:

`sudo EDITOR /etc/samba/smb.conf`

```
[Backups]
  comment = Time Machine Backups
  path = /media/NetworkBackup/Time Machine
  fruit:time machine = yes

[File History]
  comment = Windows File History
  path = /media/NetworkBackup/File History
```

The global config for this server is the same as what I have for my file shares. The only difference for Time Machine is setting `fruit:time machine = yes` on the share that hosts Time Machine backups.

## Conclusion

That's it! Samba has been working much better for my Time Machine backups, insofar as they haven't started becoming corrupt spontaneously. I knew I'd have to make this change eventually, I was just expecting to do it with Ubuntu 20.04 LTS.
