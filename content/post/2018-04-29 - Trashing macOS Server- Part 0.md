+++
title = "Trashing macOS Server: Part 0 - Setup"
date = 2018-04-29T12:45:00-05:00
tags = ["homelab", "apple", "tech"]
series = "Trashing macOS Server"
slug = "trashing-macos-server-part-0-setup"
+++

Before I could replace macOS Server, I had to prepare some kind of virtual machine to house all the new stuff. Since the rest of my homelab VMs are running Ubuntu Server 16.04, that's what I chose to use for the new server, Kestrel.

<div style="background-color: #f0f0f0; padding: 16px; margin-bottom: 1.7em;">
    This story is part of a series on migrating from macOS Server to Ubuntu Server.
    <br /><br />
    You can find all of the other stories in the series <a href="/series/trashing-macos-server">here</a>.
</div>

## Installing Ubuntu Server 16.04

There's really not much to say here. I went through the "normal" procedure for preparing a base installation of Ubuntu Server. When I got to the point of installing packages, I only opted to grab the SSH server, no web server or file server or anything. I knew I'd _need_ those soon, but I wanted to install them all by hand as the need arose.

## Basic Configuration

Before I started installing all my software, I got a couple of administrative items taken care of. That is:

- Install updates
- Enable the firewall
  - `sudo ufw enable`
  - `sudo ufw allow in “OpenSSH”`
- Set a static IP address

## Disks

I also created a plan for disk management and mounted the disks I'd need. My plan looked something like this:

- Create another virtual disk on vSphere on a regular HDD. Put it into a folder named the same as the VM folder on the boot datastore.
- `sudo apt install scsitools`
- `sudo rescan-scsi-bus`
- `sudo fdisk -l` - confirm new disk is available
- Format: `sudo cfdisk /dev/sdx`
  - `gpt` table, new partition, whole disk, change type to `Linux LVM`, write
  - If using a disk that was already formatted, and you need to wipe a GPT partition table from it: `sudo gdisk /dev/sdx`, then `x, z, (y, y)`, then proceed with `cfdisk`
- LVM
  - `sudo pvcreate /dev/sdxN`
  - `sudo vgcreate NAME-vg /dev/sdxN ...` (`...` denotes other PVs to include in the new VG)
    - On my system...
      - Each "type" of storage will have a VG, e.g. Shares, Network Backups, Local Backups
      - Each VG will have only one LV inside that we can expand to fit the VG when more storage is needed
  - `sudo lvcreate -l 100%FREE -n LV_NAME VG_NAME`
  - `sudo mkfs.ext4 /dev/VG_NAME/LV_NAME`
- Mount
  - `sudo mkdir /media/MOUNT_NAME`
  - `sudo mount /dev/VG_NAME/LV_NAME /media/MOUNT_NAME`
  - `sudo vim /etc/fstab`
    - `/dev/VG_NAME/LV_NAME   /media/MOUNT_NAME   ext4    defaults,acl,user_xattr        0       2`
- Reboot, make sure nothing blows up and your new LV is mounted

My configuration with vSphere made use of local RDM disks, but that doesn't really affect the general idea.

Here's how my LVM setup looked in the end:

```
$ sudo pvs
  PV         VG               Fmt  Attr PSize  PFree
  /dev/sda3  Kestrel-vg       lvm2 a--  24.02g    0
  /dev/vda1  Shares-vg        lvm2 a--   3.64t    0
  /dev/vdb1  NetworkBackup-vg lvm2 a--   3.64t    0
  /dev/vdc1  LocalBackup-vg   lvm2 a--   3.64t    0

$ sudo vgs
  VG               #PV #LV #SN Attr   VSize  VFree
  Kestrel-vg         1   2   0 wz--n- 24.02g    0
  LocalBackup-vg     1   1   0 wz--n-  3.64t    0
  NetworkBackup-vg   1   1   0 wz--n-  3.64t    0
  Shares-vg          1   1   0 wz--n-  3.64t    0

$ sudo lvs
  LV            VG               Attr       LSize
  root          Kestrel-vg       -wi-ao---- 22.02g
  swap_1        Kestrel-vg       -wi-ao----  2.00g
  LocalBackup   LocalBackup-vg   -wi-ao----  3.64t
  NetworkBackup NetworkBackup-vg -wi-ao----  3.64t
  Shares        Shares-vg        -wi-ao----  3.64t
```

As you can see, I have 4 VGs - `Kestrel-vg` is the actual system, `LocalBackup-vg` is for backups of the actual server and files it hosts, `NetworkBackup-vg` is for Time Machine backups from computers on the network, and `Shares-vg` is where shared files go.

Currently I just have 1 physical disk for each group, and essentially 1 volume inside each. Eventually the idea is that if I need more physical storage, I'll just toss in another disk and add that space to the VG & LV that needs it. It's not extremely sophisticated, but I don't need anything fancy.

---

And that's it! With that, I'm ready to start installing what I need on Kestrel.
