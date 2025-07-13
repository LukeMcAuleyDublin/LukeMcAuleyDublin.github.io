---
layout: post
title:  "Using Jekyll"
date:   2025-03-21 18:46:02 +0100
categories: jekyll
---

As a programmer, I was in the market for something to quickly bootstrap a personal site. I am primarily a Ruby on Rails developer so when I found out about [Jekyll][jekyll-url] it interested me. It feels like a stripped down, bootstrapped version of rails that allows you to quickly spin up and deploy a simple blog-like site.

It was just a matter of a couple simple commands to get this site running. Since I use [ASDF Version Manager][asdf-vm] to manage all my machine's tool versions, here are the relatively few commands it took to get this site up.

{% highlight bash %}
asdf plugin add ruby

asdf install ruby latest

asdf set -u ruby latest

gem install bundler jekyll

jekyll new <my_site_name>

cd <my_site_name>

bundle exec jekyll serve
{% endhighlight %}

Another reason that interested me in Jekyll was how it uses markdown to render these posts. I am an [Obsidian][obsidian-url] proselytizer, and will jump at any opportunity to copy-paste some of my musings in that app and infect the internet with them.

On top of that, I can easily host this site on [Github Pages][github-pages-url] and point my domain at it.

The default theme is nice and simple to boot :) 

[jekyll-url]: https://jekyllrb.com/
[asdf-vm]: https://asdf-vm.com/
[obsidian-url]: https://obsidian.md/
[github-pages-url]: https://pages.github.com/
