---
title: HTTPS Support
category: meta
tags: blog
date: 2017-09-20 01:00:00 -04:00
---

I just added HTTPS support to this site 😄

I was reading a [blog post by Jesse Squires](https://www.jessesquires.com/blog/building-a-site-with-jekyll-on-nfsn/) where he describes doing this for his own personal site in detail by migrating over to [NearlyFreeSpeech.net](https://nearlyfreespeech.net), and how it's been very successful for him. Since HTTPS support was something I've been very eager to do, this post gave me the push to actually get it done. I followed all the steps he laid out almost exactly except for the bash-specific parts. Since I like the [fish shell](https://fishshell.com) a lot, I tweaked the git hook script a bit to work with that, and used the fish way to set persistent universal environment variables instead of editing some `.bash-profile` file.

After a couple small technical hiccups, <https://www.klundberg.com> is being served from nearly free speech's servers without issue, and you are auto-directed to the https version if you try to visit the normal http url, which should ensure you're more secure as you read this site. Enjoy!
