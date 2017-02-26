---
title: "Important Blog Changes!"
tags: blog
category: meta
---

I've made a couple tweaks this site that I think warrant a post so that everyone is aware.

### Comments

Firstly, I've added [Staticman](https://staticman.net) comments to all blog posts by default. This is a really cool service that takes any comments you might want to write, and submits them to my blog's git repo as a pull request, where I can review it and accept it if it doesn't violate any common decency standards. I encourage you to check out their site, as it's a great solution to providing dynamic content for a statically generated site.

The thing I really like about this service over other choices for comments is that the comments data isn't treated differently from blog posts themselves. Services like Disqus and the like own and control all the data you post to them, and as a site owner, you become dependent on that service to keep your site's content complete. Some of them may have ways you can export your data, but there's no guarantee they will keep that feature available. The format the data is exported in also might not be portable to other commenting platforms, and it might be a pain to integrate it as plain HTML if you chose to do that.

Using these services also forces you to rely on them to serve javascript that you do not control, which can possibly track your readers without their consent, or be exploited by some bad actor if the service's security isn't up to snuff. I haven't done thorough investigations into how exactly different services handle these things but I dislike the fact that these problems are even a possibility. Even though in the grand scheme of things it's unlikely anything catastrophic would happen, I'd rather be safe than sorry.

### Analytics

The second change I've made follows closely from the rationale for the first. From the start, I've had Google Analytics tracking enabled on this site so I can see what posts might be popular. I checked that data somewhat frequently early on, but more recently I haven't really cared about it so much. I also was never 100% comfortable with adding it in the first place due to privacy concerns (as Google has the ability to know everything about what sites you visit if they all have Google Analytics running). I typically leave Google Analytics blocked with ublock origin, so it felt a bit hypocritical to have this enabled on my site when I don't opt to let other sites track my browsing in the same way.

With all this in mind, moving forward I've decided to disable Google Analytics completely. I'm OK with going without analytics for now, but if I ever decide to add it back, I'll likely go with something I roll out myself, or some off the shelf self-hosted solution so that I can preserve your privacy.

### What's next?

My goal with these changes is to never serve javascript or other content that is sourced from anywhere outside of domains that I own, so that nobody else can have control over what your browser does when visiting this site. I haven't done a complete audit of the theme I'm using here though, so something might slip through the cracks. If you notice anything, please let me know (you can do so in the fancy new comment boxes too 😄).

As for next steps, the highest priority task would be to enable https on this site somehow. Right now GitHub pages hosts it, but that can't do https over custom domains. I'll have to either look into hosting the blog directly someplace myself, or look into some proxying service to handle https (though that's less ideal). If anyone has experience with this sort of thing, I'd love to hear any advice you might have.

I hope you appreciate these changes and the discussion of the rationale behind them. Thanks for reading!
