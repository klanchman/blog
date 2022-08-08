+++
title = "Using Ghost for Blogging"
date = 2018-03-18T23:39:00Z
tags = ["meta", "homelab"]
image = "/img/post-images/ghost.png"
+++

I decided to try out Ghost for blogging. I think I like it.

Previously I was using Jekyll, merging what it generated into the rest of my website, and tossing that onto GitHub Pages. This worked well enough, but wasn't very convenient. I figured considering how infrequently I posted things on my blog, it wasn't a big deal. But lately I've found myself wanting to post things, but not wanting to deal with the hassle of remembering _how_ to post things.

I noticed that someone I follow named [Quad](https://twitter.com/kuwaddo) has a blog using Ghost. I thought it looked cool, so I decided to give it a shot. What better way than to fire up a new Proxmox VM on my server at home?

An exercise in frustration later[^1], Ghost was installed and working basically as expected. There is probably an oddity or two in my configuration since I run most of my stuff behind an nginx reverse proxy that handles my SSL stuff, but it doesn't seem to be impacting anything yet.

Migrating (read: coping & pasting) my old posts into Ghost was easy enough, and surprisingly I can set dates in the past for new posts. Adding cover images for everything didn't feel worth it, so those old posts aren't very pretty. I'll try not to care.

There were a couple of minor things that bothered me in the default Ghost theme, Casper. I searched around for a while for a new theme, only to find out that the Ghost theme landscape is just as annoying as the WordPress one. It's hard to discover themes, and the ones that are easier to find cost money. I didn't particularly care for the ones I found, free or otherwise. Instead, I forked Casper and changed the few things I wanted to. I added syntax highlighting for code blocks and made post images a little less gaudy. I might tweak more in the future, but I like how it looks for now.

With Ghost set up, I hope to start posting updates a little more frequently. Maybe not monthly, but considering my last post was about 3 years ago, anything is an improvement.

[^1]: `sudo npm <do_something>` is awful, and I hate that Ghost seems to require it. Maybe I could have gotten `nvm` to work eventually, but I got impatient.
