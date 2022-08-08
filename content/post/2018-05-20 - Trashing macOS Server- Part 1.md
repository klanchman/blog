+++
title = "Trashing macOS Server: Part 1 - File Server"
date = 2018-05-20T22:10:00-05:00
tags = ["homelab", "apple"]
series = "Trashing macOS Server"
+++

One of the most important things to have on a home server is some kind of file server. I chose to replace macOS Server's file server with Samba on Ubuntu. This ended up being one of the more difficult things to get correct.

{{% box %}}
This story is part of a series on migrating from macOS Server to Ubuntu Server.

You can find all of the other stories in the series [here](/series/trashing-macos-server).
{{% /box %}}

## The Goal

I have never had much luck with Samba. I could never get it working quite how I wanted. There were always permission issues, and sometimes it'd just go down randomly. Not only was my goal to make a stable, permission-issue-free Samba server, I also wanted it to work very closely to how the macOS Server file server works, namely that extended attribute files (the files that start with `._`) would not cause a mess if I access shared folders from the terminal on my Mac.

## Laying the Groundwork

First things first, we need to install Samba:

```
sudo apt install samba
sudo ufw allow in "Samba"
```

After that, it might be a good time to set up all the folders you plan to have shared. In my instance, I created a `/media/Shares` directory to hold all of my shared folders. I also created a group to easily manage users who can read & write to the shares. For the sake of this next code block, `server` is the main user on the machine, `shares-access` is a group that will have read/write permissions for shared folders, and `klanchman` is a member of `shares-access`. These commands should be adjusted to match your system and desired username for connecting to shared folders.

```
sudo adduser klanchman
sudo smbpasswd -a klanchman
sudo addgroup shares-access
sudo usermod server -aG shares-access
sudo usermod klanchman -aG shares-access

# Create the shared paths, chown to server:shares-access, chmod 770 - example:
# cd /media/Shares
# sudo mkdir Plex\ Media
# sudo mkdir Public
# sudo chown server:shares-access Plex\ Media/ Public/
# sudo chmod 770 Plex\ Media/ Public/

# Create private paths
# cd /media/Shares
# sudo mkdir klanchman
# sudo chown klanchman:klanchman klanchman
# sudo chmod 770 klanchman
```

Notice that we'll also be configuring some shared folders, like `klanchman`, that only allow a specific user to access them, rather than anyone in the `shares-access` group.

## Samba Configuration

Now it's time to configure Samba itself. This is the part that caused the most headache for me.

It's important to point out that I am accessing shared folders _exclusively_ with macOS clients. I think Linux clients should behave fine too, but if you also want to use Windows clients you may need to tweak some of this configuration.

First `sudo EDITOR /etc/samba/smb.conf`, replacing `EDITOR` with your favorite editor. Near the top, set the following if it's not already there:
```
[global]
security = user
```

Then, comment printer-related stuff (assuming you don't plan to share printers).

Then come the shares. I'll dump the config first, then explain in the next section a couple of things that are here, and _why_ they're here.

```
### My Shares ###
# Global options
guest ok = no
read only = no
create mask = 0660
directory mask = 0770
map archive = no
map hidden = no
map system = no
valid users = @shares-access

ea support = yes
vfs objects = fruit streams_xattr
fruit:aapl = yes
fruit:encoding = private
fruit:resource = file
# Prevent macOS from assigning its own permissions
fruit:nfs_aces = no

inherit acls = yes
inherit permissions = yes
acl group control = yes
nt acl support = yes
acl map full control = no

[Public]
  comment = Public
  path = /media/Shares/Public

[Plex Media]
  comment = Plex Media
  path = /media/Shares/Plex Media

[klanchman]
  comment = klanchman
  path = /media/Shares/klanchman
  force user = klanchman
  force group = klanchman
  valid users = klanchman
```

Save the file, and then restart Samba:
```
sudo systemctl restart smbd.service nmbd.service
```

### Explanation

If you want to know why/how the above stuff works, read on. Otherwise, you can skip to the next section.

---

So what is all of this doing? Let's look a little closer...

It's important to note that everything up to `[Public]` applies to all shares. This config makes some assumptions about how you want to use your file server, such as limiting who has access to the shares.

This block does most of the heavy lifting in preventing permissions issues:
```
create mask = 0660
directory mask = 0770
map archive = no
map hidden = no
map system = no
```
This says files should be created with `0660` permissions (i.e. `-rw-rw----`), and directories `0770`. A point of interest is the `map xxxx = no` directives. If these were set to `yes`, Samba would map the Windows archive/hidden/system bits to the execute bits on the Linux side. For instance, if you had a file with the archive bit enabled, the permissions on the Linux side would become `0760`. But that also means that macOS will see them that way, which isn't what we want!

The next block of config that helps with permissions issues says this:
```
inherit acls = yes
inherit permissions = yes
acl group control = yes
nt acl support = yes
acl map full control = no
```
Ultimately this will allow ACLs set on the Linux side to be respected by Samba. I use ACLs to some extent (I'll explain more in upcoming posts), so I wanted to make sure they worked properly in shared folders.

Last, the block that does some Apple-specific stuff:
```
ea support = yes
vfs objects = fruit streams_xattr
fruit:aapl = yes
fruit:encoding = private
fruit:resource = file
# Prevent macOS from assigning its own permissions
fruit:nfs_aces = no
```
The first line is easy, it enables extended attribute support. But what is all this `fruit` stuff? I'm glad you asked! The Samba doc for it is [here](https://www.samba.org/samba/docs/current/man-html/vfs_fruit.8.html). Basically it enabled enhanced support for Apple clients. So, a quick breakdown:
- `vfs objects` essentially loads the module
- `fruit:appl = yes` turns it on the extension
- `fruit:encoding = private` stores characters as the macOS client sends them
- `fruit:resource = file` keeps AppleDouble files (the `._` files) for extended attributes
- `fruit:nfs_aces = no` prevents macOS from doing stuff to permissions (in my experience)

You might be asking, "Why are you using `fruit:resource = file` if you don't want the `._` files to show up?" Number 1: the other options weren't going to work for me, namely `xattr`, since I'm running Ubuntu and ext4, not Solaris and ZFS. Number 2: in my experience the files aren't actually visible on the share, but the attributes stick around. Of particular interest, the files don't even seem to show up on Linux inside the share. Where are they? I don't actually know, but the attributes work so I don't really care!

So that's the gist of it. It doesn't look like much at a glance, but getting this to work the way I expected took many hours spread over multiple days. Documentation for some of these options was a little scattered, and first-hand accounts of using them numbered few. It was..._interesting_ getting this to work, but I'm quite happy with the result.

## Avahi Configuration

We're almost done! We've got a slick new Samba server set up, but nothing on the network knows about it. That's where Avahi comes in. Avahi is basically an implementation of Bonjour for Linux. It will broadcast the existence of our Samba shares to devices on the network.

Getting it set up is pretty easy:

```
sudo apt install avahi-daemon
```

Now we need to create a UFW rule.

`sudo EDITOR /etc/ufw/applications.d/avahi`
```
[Avahi]
title=Avahi
description=Avahi Zeroconf
ports=5353/udp
```

Then `sudo ufw allow avahi`.

Now we need to set up a service for Avahi to broadcast.

`sudo EDITOR /etc/avahi/services/smb.service`
```xml
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
 <name replace-wildcards="yes">%h</name>
 <service>
   <type>_smb._tcp</type>
   <port>445</port>
 </service>
 <service>
   <type>_device-info._tcp</type>
   <port>0</port>
   <txt-record>model=RackMac</txt-record>
 </service>
</service-group>
```
This tells Avahi we have a service running on port 445 called `_smb._tcp`. You may also notice `<txt-record>model=RackMac</txt-record>`. This tells Mac clients what to show as the icon for the server in Finder. I went with `RackMac`, which looks like an Xserve server. You can find other names by looking through `/System/Library/CoreServices/CoreTypes.bundle/Contents/Info.plist` on your Mac.

Lastly we just need to start Avahi:

```
sudo systemctl enable avahi-daemon
sudo systemctl start avahi-daemon
```

---

If all went well, you should now see your Ubuntu server in your Finder sidebar as a file server. You should be able to connect using the credentials you created with `smbpasswd` earlier. Next time we'll take a look at how to do Time Machine backups to the server!
