---
title: Jekyll frustrations
description: My recent jekyll woes
tags: Jekyll
category: meta
---

It's been awhile since I wrote a blog post, and upon wanting to come back and do another I noticed that jekyll had been updated to 3.0 (jekyll is what generates this site, which I chose since it's currently hosted on github pages which has native jekyll support). I typically am enthusiastic about upgrading to the most up to date versions of the software I use so I figured I'd give it a go. However I ran into a few roadblocks that I could not easily find answers to on duckduckgo/google so I figured I'd post them here in case anyone else hits them.

# Reinstalling and updating gems

The biggest issue stemmed from me upgrading to OS X 10.11, which due to system integrity protection (SIP) a lot of my gems were wiped out of their default installation spot. I had to reinstall things like bundler and jekyll but with the following amended command:

{% highlight bash %}
sudo gem install -n /usr/local/bin jekyll
{% endhighlight %}

Without the -n argument, gems try to install to /usr/bin which is off limits now from SIP. I would have hoped that Apple would have changed gem install to default there in their distribution of Ruby, but perhaps that's asking too much :(. 

# _config.yml and syntax coloring woes

Jekyll's previous config for 2.x needs a couple changes depending on what you have. I had to change the `use_coderay` parameter to `enable_coderay` to make things work; luckily the error message there was clear and told me exactly what to do.

Next was a really weird issue. Every time I tried to build I got this kind of error:

{% highlight bash %}
  Dependency Error: Yikes! It looks like you don't have pygments or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- pygments' If you run into trouble, you can find helpful resources at http://jekyllrb.com/help/! 
 ERROR: YOUR SITE COULD NOT BE BUILT:
        ------------------------------------
        pygments
{% endhighlight %}

Pygments is my default syntax colorizer. I didn't understand why this broke since it worked in jekyll 2.4 just fine. I tried reinstalling/updating the `pygments.rb` gem, updated pygments itself through `pip` (since it's actually a python library), and other things that I forget since it was way too late to be doing this, but nothing worked. I tried changing the `highlighter: pygments` config setting to `highlighter: rouge` (the other supported syntax coloring module) but then the same error came up with rouge instead of pygments.

The solution here which I stumbled upon was to change `pygments` in the config setting to `pygmentize`, which is the actual command line program that one runs to do the highlighting for some input. I suspect using `rougify` would also work as that's rouge's command line tool. It makes sense that the actual command is needed now so that someone could use any other highlighting tool that may not have been supported before, but an intelligent error message would be nice to explain that this is an outdated argument that should change, not tell me that pygments or rouge is not installed when it actually is.

# Liquid templating

Most of the above I did yesterday, so today's endeavors I can follow along with what I'm doing here more closely.

Once my highligher setting was squared away, I ran into another issue:

{% highlight bash %}
Liquid Exception: undefined method `t' for nil:NilClass in ~/workspaces/klundberg.github.io/_drafts/fun-with-optionals-part-1.md
{% endhighlight %}

A Google search here yielded some results, noting that the yaml front matter for others was malformed for them which could cause similar errors. This post which is currently being drafted had a title like this:

{% highlight bash %}
title: Fun With Optionals - Part 1 - Basics
{% endhighlight %}

perhaps the dashes are tripping up yaml, so I surrounded it with quotes:

{% highlight bash %}
title: "Fun With Optionals - Part 1 - Basics"
{% endhighlight %}

No dice. I wonder if there's some non-printing character somewhere in the front matter, so I rewrite it all:

{% highlight bash %}
Error reading file ~/workspaces/klundberg.github.io/_drafts/fun-with-optionals-part-1.md: (<unknown>): mapping values are not allowed in this context at line 2 column 34
jekyll 3.0.0 | Error:  undefined method `each' for nil:NilClass
{% endhighlight %}

Well, that's a more reasonable error that tells me more specifically what may be happening. My post's title now looks like this:

{% highlight bash %}
title: Fun With Optionals, Part 1: The Basics
{% endhighlight %}

It seems that the colon is not looked upon favorably here. If I change it to another comma:

{% highlight bash %}
  Liquid Exception: undefined method `t' for nil:NilClass in ~/workspaces/klundberg.github.io/_drafts/fun-with-optionals-part-1.md
jekyll 3.0.0 | Error:  undefined method `t' for nil:NilClass
{% endhighlight %}

One step forward, another step back (or perhaps a couple steps sideways). I strip out all the punctuation from the title, and from the description for good measure, but with no change. I then find [this link](https://github.com/jekyll/jekyll/issues/2678) which looks promising. When I comment the highlighter line completely in _config.yml:

{% highlight bash %}
             ERROR: YOUR SITE COULD NOT BE BUILT:
                    ------------------------------------
                    rouge
{% endhighlight %}

Argh! I don't specify a highlighter anywhere but it can't seem to find an installed highlighter it searches for by default for some odd reason. I'm getting very frustrated here to say the least.

I revert that change, then try the --trace option to see the actual backtrace ruby generates for the undefined method error. The line it fails on is this:

{% highlight ruby %}
raise SyntaxError.new(options[:locale].t("errors.syntax.invalid_delimiter".freeze,
                                                 :block_name => block_name,
                                                 :block_delimiter => block_delimiter))
{% endhighlight %}

So it seems it's trying to report an actual liquid syntax error, but :locale in options is nil so it can't get that far. I'm not sure how to make that :locale key populate, but this error lives in an `unknown_tag` method in the problem file within Liquid. Since my post that it's failing on contains some generics, I wondered if it was stray html angle brackets that were causing things to crash and burn. I had converted most of them to `&lt; &gt;` but there were a few stray set of brackets that I forgot to replace. Seems reasonable, right? Wrong, the error didn't change.

Ultimately the problem was with the highlight ending tags. I had inadvertently used `end highlight` instead of `endhighlight` to end my code highlight sections. I presume that Jekyll would have told me about this had it not stumbled when trying to raise the liquid syntax error. Fixing that caused it to generate the site without a hitch.

# Ugh

This whole process has soured me on Jekyll a bit. Some of it is definitely environment problems and some was just PEBKAC-related, but not getting reasonable errors to point me in the right direction is definitely a big issue. Jekyll is already fairly complex without these problems, and if there's a better static site generator out there that is a bit simpler for my needs I'll definitely have to try it out. If anyone reading this has some suggestions, please feel free to reach out to me on [twitter](https://twitter.com/kevlario).

Next time, something a bit more informative, I promise :)
