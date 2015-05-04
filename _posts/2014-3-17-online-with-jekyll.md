---
layout: post
title: Online With Jekyll
tags: general
description: How I set up Jekyll on Windows 7 machine in a handful of easy, precise steps. 
---
<h3>Update - 5 May 2014</h3>
I got a new computer so was fearing going through the process of again getting Jekyll set up on the new machine.  However, I came across an [awesome Jekyll for Windows tutorial](http://jekyll-windows.juthilo.com/) that makes it completely painless.  Only slight hiccup is after running `ruby dk.rb init`, it generates a `config.yml` file.  You need to update that to point to your Ruby Dev Kit path before running `ruby dk.rb install`.  However, the error message from the latter makes it pretty clear what to do.  

<h3>Make Jekyll Play Nicely With Windows</h3>
This is the inaugural entry in my blog covering various software development topics, especially related to code design and dev practices.  To kick this off, I'm going to briefly describe the steps I had to follow to get Jekyll Bootstrap online on Windows 7 as I had a bit of trouble that some of the excellent tutorials didn't quite cover.

I had planned to just use Wordpress and purchase some domain hosting through someone like Bluehost as that seemed like a relatively well traveled path to get going.  But for some reason, I seemed to have some block at getting things set up.  I read [Mark Seemann's excellent blog](http://blog.ploeh.dk/) and he had a post about switching to Jekyll right at a year ago, but it didn't mean much to me as I wasn't familiar with Github, amongst other challenges.

I crossed the "lack of experience" issue off the list a few months back as I contributed some small patches to AutoFixture through Github and found Github to be truly awesome compared to the monolith that is TFS I normally use.  The idea of Github Pages sounded great:  easy content creation, push to Github, free; what's the downside?  The only remaining one is I don't consider myself a front-end developer and investing a bunch of time to re-learn the intricacies of CSS was more than I wanted to take on.  Wordpress has hundreds or thousands of themes available, but that ecosystem doesn't exist for Jekyll yet (although there are some).  

Ultimately though, a cool new colleague of mine broke the logjam and said she'd help me make it look awesome so all that was left was for me to get it online.

The only truly required piece is to create a Github repository in a very precise format.  The [Github Pages tutorial](http://pages.github.com/) was more than sufficient to take care of that.  Follow those steps, create a basic "index.html" page (with normal HTML markup), push it to the repository, wait about 10 minutes, and voila.  If you're truly skilled with some of Markdown dialects or never care to preview anything before pushing it, you could basically stop here.  But like most folks, I want to be able to iterate on my content and look-and-feel locally before pushing some god-awful mess onto the web.  For that, I needed a local install of Jekyll and therein lies the challenge.  

Jekyll is a Ruby gem and what very minimal experience I tried to obtain in Ruby a while back did not go well as I'm predominately a Windows user.  I have a Linux machine, but I almost exclusively use Windows for .NET development so the thought of using a separate machine for blogging was not palatable.  I recalled trying to get Ruby set up was not the easiest thing in the world compared to the "install and run" Microsoft approach.  

I followed [Madhur's tutorial](http://www.madhur.co.in/blog/2011/09/01/runningjekyllwindows.html) from a few years back essentially to the letter.  I downloaded Ruby 1.9.3 and DevKit-tdm-32-4.5.2-20111229-1559-sfx.  All of that installed fine, ran `gem install jekyll` and it installed Jekyll 1.4.3.  I already had Python 2.7 installed, ran `easy_install Pygments` which installed version 1.6.0.  I ran `jekyll new myblog`, switched directory, and now for the moment of truth: `jekyll serve`.  But alas, errors!

Unfortunately, Madhur's tutorial, while it listed some errors, didn't have one that covered what I was seeing:  posix-spawn-0.3.8 throwing an error because it couldn't locate the `which` command.  I Googled around, tried switching posix-spawn from 0.3.8 to 0.3.6, that didn't do any good.  I came across some Stack Overflow posts that Pygments 1.6 didn't play nicely on Windows so I removed it (by deleting some `pygmentize` files from the Python install's Scripts folder).  Then I ran `gem install pygments.rb --version "=0.5.0"` to get that specific version as a Ruby gem.  Jekyll still didn't work, but `gem list` revealed that I had both Pygments 0.5.0 and 0.5.4.  Running `gem uninstall pygments.rb --version "=0.5.4"` took care of that.  Yes you'll probably notice that I went from the Python Pygments to installing it as a Ruby gem.  

At this point I was feeling good so tried `jekyll serve` again, but still failed.  But I had moved forward from posix-spawn errors to fileutils errors.  A bit more Stack Overflowing turned up that Jekyll 1.4.3 seems to have some bug related to fileutils on Windows.  Simple enough, `gem uninstall jekyll` to remove 1.4.3, then `gem install --version "=1.4.2"` to install 1.4.2.    Run `jekyll serve`, hold my breath, and errors, this time due to needing wdm.  Quick `gem install wdm` installed 0.1.0.  Moment of truth., run `jekyll serve` and... **Success!**

### TL;DR - Make Jekyll Play Nice with Windows 7
To summarize the versions I had to use:

1. Ruby 1.9.3
2. Python 2.7.6
3. Pygments 0.5.0
4. Jekyll 1.4.2
5. wdm 0.1.0

From there, I've been using [Anna Debenham's guide](http://24ways.org/2013/get-started-with-github-pages/) to some basic Jekyll configuration.  

Hopefully if someone else is trying to get Jekyll going on Windows 7, this might save you some effort at least as of this writing.  





