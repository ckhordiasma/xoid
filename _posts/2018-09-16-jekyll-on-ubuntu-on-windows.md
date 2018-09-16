---
layout: post
title: "Jekyll on Ubuntu on Windows"
date: 2018-09-16
---
Doing more work with Ubuntu on windows. I accidentally overwrote my Ubuntu on Windows installation, deleting all of my linux files! Oh well. Since I'm starting from scratch, I am working on getting things back to working again, starting with Jekyll and Github/Gitlab.

Now, there are two ways of using Jekyll with Github. On Github, you can just have the source Jekyll files (the *.md files, the _layouts and _posts directories, and NOT the _site folder), commit/push those onto the Github repo, and github's servers will worry about doing all the `jekyll serve` stuff that needs to be done. 

This is very convenient. However, if you want to be able to use any special Jekyll plugins, you will only be able to do so if Github's server version of Jekyll has those plugins! So in terms of plugin functionality, you are at the mercy of whatever Github has.

Gitlab is a little different. They have this thing called Gitlab CI (gitlab continuous integration) which basically allows you to tell the server to run commands after each commit onto the hosted git repository. So it is possible not only to just post Jekyll source code onto Gitlab, but you can also make sure the correct plugins are installed (I think).

Here are the links I looked at: [gitlab documentation](https://docs.gitlab.com/ee/user/project/pages/getting_started_part_four.html), [some guy's blog](https://www.chenhuijing.com/blog/hosting-static-site-gitlab-pages/)

The other way of using Jekyll with Github/Gitlab is to use jekyll locally to generate the static site, and then push those files onto Github/Gitlab. At this point, you are just basically using Github/Gitlab as a site hosting service, not as a version control system; however, it is a lot simpler and easier to use.

OK, so with that background information, it's clear that with either way, it is useful and/or necessary to have Jekyll isntalled on my computer, and the main purpose of this post is to document what I went through to install Jekyll (in case I need to install it again in the future).

I am following the trusty instructions [here](https://jekyllrb.com/docs/installation/windows/#installation-via-bash-on-windows-10) in order to install Jekyll on Windows w/ Ubuntu. I am going to go through every execution in the instructions, as well as any additional execution I needed to do. 

Before starting, I cloned my git website repo with `git clone https://github.com/ckhordiasma/ckhordiasma.github.io.git`.

The first ruby instruction was to update the update repositories. `sudo apt-get update -y && sudo apt-get upgrade -y`

Next I had to add a repository with a special ruby build specifically for Windows with Ubuntu, and then install that version of ruby. Note that what was actually installed was ruby and its developmental version. I'm not sure what the `build-essential` and `dh-autoreconf` mean.

```terminal
sudo apt-add-repository ppa:brightbox/ruby-ng
sudo apt-get update
sudo apt-get install ruby2.5 ruby2.5-dev build-essential dh-autoreconf
```
Next is updating ruby gems and installation of jekyll and bundler (the last two commands there are checking jekyll version number and making so that bundle doesn't prompt for sudo). 

You may be wondering what a ruby gem is. It seems to just be a fancy name for a ruby application or library. Jekyll is a ruby gem (application), and so is bundler. Bundler is a ruby gem which tracks the version numbers of dependency gems when you are developing ruby project. 

```terminal
sudo gem update
sudo gem install jekyll bundler
jekyll -v
bundle config path vendor/bundle
```
All right, so after I did all this, I could CD into my website folder and run `jekyll serve` and see my website! hooray. 
HOWEVER, the jekyll website says that the proper way to run jekyll is actually `bundle exec jekyll serve` to make sure all dependencies are OK. When I tried that, I got the following error: `Could not locate Gemfile or .bundle/ directory`. I next tried `bundle install` and got the error `Could not locate Gemfile`. So I need a gemfile. Turns out `bundle init` will generate a gemfile for me. I then tried `bundle install; bundle exec jekyll serve` again and got the error 
```
bundler: failed to load command: jekyll (/usr/local/bin/jekyll)
Gem::Exception: can't find executable jekyll for gem jekyll. jekyll is not currently included in the bundle, perhaps you meant to add it to your Gemfile?
  /var/lib/gems/2.5.0/gems/bundler-1.16.4/lib/bundler/rubygems_integration.rb:462:in `block in replace_bin_path'
  /var/lib/gems/2.5.0/gems/bundler-1.16.4/lib/bundler/rubygems_integration.rb:482:in `block in replace_bin_path'
  /usr/local/bin/jekyll:23:in `<top (required)>'
``` 
Ok, that makes it look like I need to include jekyll in my gemfile. I added `gem 'jekyll'` near the bottom of my Gemfile file and tried serving again. This worked! Now I can see my website at 127.0.0.1:4000 .

Now going back to the two different types of using Jekyll with Git. I currently use Github, and I don't have any fancy jekyll plugins or anything. Therefore, I am going to just upload my source Jekyll code into my Github repo and let Github worry about generating my site for me. For now.

So in doing this, I need to update my `.gitignore` file to make sure extraneous jekyll/bundler stuff doesn't get added to my github repository. I also added `.sass-cache` in there in case I do sass stuff later. Here is my `.gitignore` file now:
```
_site/
.sass-cache
#ignoring bundler generated stuff
vendor/
bundle
```

ok! I think I am done for now. 

