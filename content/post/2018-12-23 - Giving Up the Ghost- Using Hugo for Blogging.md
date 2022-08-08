+++
title = "Giving Up the Ghost: Using Hugo for Blogging"
date = 2018-12-23T15:43:00-05:00
tags = ["meta"]
+++

Earlier this year, [I started using Ghost for my blog](/using-ghost-for-blogging). After a few months, I've decided to make another change in platform, this time to Hugo.

## Why Switch?

I have a handful of reasons for migrating away from Ghost.

### Hiding Markdown

The biggest reason I had to distance myself from Ghost was that they have started making Markdown harder to use.

Typically, I write my posts in Markdown using [Caret](https://caret.io), sync them in iCloud Drive, and then paste them into the Ghost editor.

Why don't I use the built-in editor? Portability, mainly. Having dealt with a migration from Jekyll to Ghost, I found that keeping my posts in a more-or-less universal format was nice. I also just enjoy writing prose using Markdown.

When I started using Ghost, the editor just used Markdown, which made things simple. The [Ghost 2.0 update](https://blog.ghost.org/2-0/) introduced a new editor that mimics Slack's annoying post editor[^1], and banished Markdown to a "dynamic card" menu a few clicks away. This change has me wondering how long it'll be until Ghost kills off Markdown support entirely.

### Questionable Practices for Self-Hosted Instances

Ghost primarily sells their managed blog instances, with a self-hosting option provided as what feels like a happy accident stemming from Ghost being open source.

Ghost's only officially-supported configuration has you install Node through the Nodesource APT repository (meh), followed by globally installing Ghost-CLI with `sudo npm` (bleh). NPM is not an amazing piece of software in my experience, and I'd rather not run it with `sudo` if I can help it. [They've already demonstrated once what can happen.](https://github.com/npm/npm/issues/19883)

### The Catalyst for Leaving

I had been planning to leave Ghost for a little while, I just didn't feel the need to rush it. Then I started having [random 502 errors](https://twitter.com/LA5ERR/status/1071918739128500224) after a Ghost update.

The behavior was pretty strange:

- My reverse proxy VM would start having issues connecting to the Ghost VM (hosted on the same physical machine)
- I'd SSH into the Ghost VM and do literally nothing else
- Suddenly the reverse proxy could connect again

I didn't feel like restoring a backup of the Ghost VM, nor did I wish to spend a lot of effort debugging the issue. Instead, I decided that it was as good a time as any to just change over to another platform.

## Why Hugo?

I wanted to go back to using a static site generator, and I wanted to try out [Netlify](https://netlify.com) for hosting it.

I decided I wanted a static site generator that was available as a standalone binary, which ruled out the Ruby gem hell of Jekyll, and the node_modules hell of Gatsby, Next, etc. (besides, I deal with JavaScript enough at my job).

I looked around to see what some other people were using, and found that 1Password  [recently starting using Hugo](https://blog.1password.com/better-faster-stronger---our-new-blog-and-how-we-made-it/), and that Quad (who I mentioned in my article about Ghost) recently [switched from Ghost to Hugo](https://blog.quad.moe/posts/moving-from-ghost-to-hugo/). (I promise I'm not just trying to copy Quad!)

I liked what I saw about Hugo: I can get it as a standalone binary via Homebrew, [it works well with Netlify](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/), it has good support for syntax highlighting, and I like how they handle content organization with things like [taxonomies](https://gohugo.io/content-management/taxonomies/).

I decided to give it a whirl.

## How?

The "how" of the migration was actually rather simple. I used [this tool](https://github.com/jbarone/ghostToHugo) to deal with my older posts and grabbed whatever static assets I needed from Ghost. I [configured Hugo to have permalinks similar to Ghost](https://github.com/klanchman/blog/blob/9ac0d487af7108108e8ad5b0e1df391dc0881c40/config.toml#L11) to avoid dealing with redirects for now. I chose [a theme](https://github.com/halogenica/beautifulhugo) that I liked.

Then I just had to set up Netlify. I built the site locally and manually deployed it to Netlify just to make sure it worked. It did, so I pointed CNAME record over to Netlify, let it set up Let's Encrypt automatically, configured [a redirect](https://github.com/klanchman/blog/blob/master/static/_redirects) for the site, and set up [automatic deployment from GitHub](https://github.com/klanchman/blog/blob/master/netlify.toml). It was all straightforward and works very well.

---

Overall I'm very happy with Hugo and Netlify for my blog. It should be able to handle more traffic than the puny VM I had it on before (not that I anticipate my blog will ever see much traffic). It's more pleasant to work with than Ghost or Jekyll. It's all in Git again, so I can drop the iCloud Drive portion of my workflow.

I hope to stay on Hugo for the foreseeable future, since jumping between blogging platforms is not how I typically like to spend my weekend. At some point I might even get around to updating my main website with Hugo and Netlify, too...

[^1]: Seriously Slack, just let me write Markdown in posts
