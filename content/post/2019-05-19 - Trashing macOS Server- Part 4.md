+++
title = "Trashing macOS Server: Part 4 - Server Backups"
date = 2019-05-19T15:23:00-04:00
tags = ["homelab", "apple", "tech"]
series = "Trashing macOS Server"
+++

While I was using macOS Server, keeping files on the server itself backed up was easy with Time Machine. I wanted a similar solution for my Ubuntu server: frequent incremental backups that "just work".

<div style="background-color: #f0f0f0; padding: 16px; margin-bottom: 1.7em;">
    This story is part of a series on migrating from macOS Server to Ubuntu Server.
    <br /><br />
    You can find all of the other stories in the series <a href="/series/trashing-macos-server">here</a>.
</div>

## Finding a Tool

I came across a handful of tools for taking scheduled backups on Linux. [A thread on Ask Ubuntu](https://askubuntu.com/questions/2596/comparison-of-backup-tools) was particularly helpful in my quest. A handful of the tools listed there were more geared toward desktop users, but plenty of them were applicable to server users too.

## rsnapshot

After reading around for a little while, I decided on [rsnapshot](https://rsnapshot.org). It offers incremental backups using hard links, much like Time Machine. It's also configurable enough for my needs, mainly the ability to pick directories to include/exclude in backups, and to configure backup schedule and retention.

### Install

Installation goes just like you'd expect:

```
sudo apt install rsnapshot
```

### Configure

Configuration requires a little bit more explanation.

First, identify the directories you'd like to include in your backup. In my case, I want to back up `/media/Shares` and `/etc` to `/media/LocalBackup`. However, I'd like to exclude my Transmission download directory under `/media/Shares/Public` since it changes often and I don't care much about its content.

####  Excluding Directories

To deal with exclusions, first make a file in a directory higher up, in my case `/media/Shares/Public`. The contents of the file should be formatted like `rsync`'s `--exclude-from` argument expects. Here's how I set mine up:

`sudo EDITOR /media/Shares/Public/server-backup-excludes`

```
Public/0 Server Torrents
```

Follow that up with some permissions changes:

```
chown server:shares-access /media/Shares/Public/server-backup-excludes
chmod 660 /media/Shares/Public/server-backup-excludes
```

#### Deciding Schedule and Retention

We're almost ready to set up our `rsnapshot.conf` file. We know what we want to backup, and where we want the backups stored. The next thing we need to figure out is what the schedule and retention should look like. Here's what I decided to do:

|Time Period|Backups to Retain|
|-----------|-----------------|
|Bihourly   |12               |
|Daily      |7                |
|Weekly     |4                |
|Monthly    |3                |

What this means is that when a 13th bihourly backup is going to be made, existing ones are incremented first, and the last bihourly becomes the first daily backup. That ripples through all of the other time periods, too.

If you want to learn more about how rsnapshot rotates backups, or you want to configure a different schedule, check out [`man rsnapshot`](https://linux.die.net/man/1/rsnapshot) for more information about `retain` and how it works. Note that `retain` doesn't actually _schedule_ anything, it just gives you meaningful names for your backups.

#### rsnapshot Configuration File

Once you've decided on schedule and retention, it's time to crack open the config file.

`sudo EDITOR /etc/rsnapshot.conf`

**Note:** While editing this file, you must use hard tab characters! You **cannot** copy directly from this post, I used spaces in my config snippets below.

We want to change a few things in here. I'll start from the top.

First, set the directory where you want backups to live:

```
snapshot_root  /media/LocalBackup
```

Check your system just to be sure, but I also had to uncomment a couple of lines:

```
cmd_du  /usr/bin/du
cmd_rsnapshot_diff  /usr/bin/rsnapshot-diff
```

Next, set up your naming and retention. Here's how mine looks:

```
retain  hourly  12
retain  daily   7
retain  weekly  4
retain  monthly 3
```

Then, set up your `rsync` short args list. Here's mine:

```
rsync_short_args  -aAX
```

What do these args mean?

|Arg |Description                                                                                                                                                                                                 |
|----|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`-a`|Archive mode. This expands to `-rlptgoD`. So: recursive, copy symlinks as symlinks, preserve permissions, preserve modification times, preserve group, preserve owner, and preserve device and special files|
|`-A`|Preserve ACLs                                                                                                                                                                                               |
|`-X`|Preserve extended attributes                                                                                                                                                                                |

I feel like these are pretty sane defaults, but you may want to look at [`man rsync`](https://linux.die.net/man/1/rsync) to see if there are other options you'd like to include. You can also find arguments you may want to include in `rsync_long_args` as well. I left that option commented in my `rsnapshot` config.

Next, set up your exclude file, if you have one:

```
exclude_file  /media/Shares/Public/server-backup-excludes
```

Last, configure what files you're backing up. Here are the items that I have:

```
backup  /media/Shares  localhost/
backup  /etc/  localhost/
```

There are a lot of other options available, so browse through the file to see if there are others you'd like to tweak.

Now, test your config:

```
# Check syntax
sudo rsnapshot configtest

# Test your first backup level, in my case `bihourly`
# This will show the shell commands that would be run. Make sure it looks right
sudo rsnapshot -t bihourly
```

As I mentioned earlier, this didn't actually schedule anything, it just configured names for your backup levels.

### Schedule

This is the fun part! We're going to make a systemd job for each backup level we defined in our config.

First, we'll need a root service that all the scheduled jobs will call.

`sudo EDITOR /lib/systemd/system/rsnapshot@.service`

```
[Unit]
Description=rsnapshot (%I) backup

[Service]
Type=oneshot
Nice=10
ExecStart=/usr/bin/rsnapshot %I
```

This job is parameterized, and will call `rsnapshot` with whatever comes after the `@` in the scheduled jobs.

Next are all my scheduled jobs. You will likely want to tweak the `OnCalendar`[^1] parameter for your jobs to be at times that make sense for your system.

`sudo EDITOR /lib/systemd/system/rsnapshot-monthly.timer`

```
[Unit]
Description=rsnapshot monthly backup

[Timer]
# The first of the month at 2 AM
OnCalendar=*-*-01 02:00:00
Persistent=true
Unit=rsnapshot@monthly.service

[Install]
WantedBy=timers.target
```

`sudo EDITOR /lib/systemd/system/rsnapshot-weekly.timer`

```
[Unit]
Description=rsnapshot weekly backup

[Timer]
# Sundays at 2:15 AM
OnCalendar=Sun *-*-* 02:15:00
Persistent=true
Unit=rsnapshot@weekly.service

[Install]
WantedBy=timers.target
```

`sudo EDITOR /lib/systemd/system/rsnapshot-daily.timer`

```
[Unit]
Description=rsnapshot daily backup

[Timer]
# Every day at 3:30 AM
OnCalendar=*-*-* 03:30:00
Persistent=true
Unit=rsnapshot@daily.service

[Install]
WantedBy=timers.target
```

`sudo EDITOR /lib/systemd/system/rsnapshot-bihourly.timer`

```
[Unit]
Description=rsnapshot bihourly backup

[Timer]
# Every other hour at 45 minutes past the hour
OnCalendar=0/2:45:00
Persistent=true
Unit=rsnapshot@bihourly.service

[Install]
WantedBy=timers.target
```

Don't enable these jobs yet! There's one more thing to do.

### Initial Backup

Before we schedule our jobs, we need to take an initial backup. This may take a little while, so I recommend starting this in a `tmux` session and detaching it so you can disconnect from the server while it goes.

```
# Start a backup for your first level, `bihourly` in my case
sudo systemctl start rsnapshot@bihourly
```

### Start and Enable the Timers

Once your initial backup is complete, all that's left is to start and enable the systemd timers we made above.

```
sudo systemctl enable rsnapshot-bihourly.timer rsnapshot-daily.timer rsnapshot-weekly.timer rsnapshot-monthly.timer
sudo systemctl start rsnapshot-bihourly.timer rsnapshot-daily.timer rsnapshot-weekly.timer rsnapshot-monthly.timer
```

## Conclusion

At this point, you should have a "set and forget" backup solution for local files on your server. It's pretty space-efficient, thanks to the use of hard links. Here's how my disk usage looks for Shares and LocalBackup:

```
Filesystem                               Size  Used Avail Use% Mounted on
/dev/mapper/Shares--vg-Shares            3.6T  2.4T  1.1T  70% /media/Shares
/dev/mapper/LocalBackup--vg-LocalBackup  3.6T  2.5T  955G  73% /media/LocalBackup
```

---

With that, I've also come to the end of explaining my replacement for macOS Server using Ubuntu. I accomplished most of my goals for replacing tools, with the exception of Caching Server, iTunes, and NetInstall. I've found that not having Caching Server hasn't been a huge loss for me, and the same goes for NetInstall. I have iTunes set up on my Mac desktop, but I actually have the media library on the server to save space. This has been good enough for my needs.

Going forward, I'll try to write more posts in this series that talk about new and useful things I add to my server, and changes I make as Apple deprecates things and when I run into problems.

Thanks for reading!

[^1]: For more information on the syntax of `OnCalendar`, check out [`man systemd.time`, section "Calendar Events"](https://www.freedesktop.org/software/systemd/man/systemd.time.html#Calendar%20Events)
