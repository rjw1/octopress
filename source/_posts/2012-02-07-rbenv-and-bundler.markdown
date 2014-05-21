---
layout: post
title: "Rbenv and Bundler"
date: 2012-02-07 10:32
comments: true
categories: ruby dev
---

I have been an occasional user of [RVM](http://beginrescueend.com/) to date. I like the problems that it solves; the ability to switch between different versions of Ruby and isolate per-project installs of Gems. But I just can't gel with the way that it solves them. Aside from arguments about `curl | sh` installs and shell overrides, it just feels like there is too much magic involved. I'm not a fan of magic.

## Gimme the Rubies

[rbenv](https://github.com/sstephenson/rbenv) is a low-touch alternative to RVM. The core project does nothing more than switch between different versions of Ruby. It does this by pre-pending some shims to your shell's `$PATH`. There are a number of optional plugins that can be used for other features.

Installation is easy. Simply grab a clone of the project's repository into your home directory. You can checkout a tag for stability or just go wild on HEAD.

    git clone git://github.com/sstephenson/rbenv.git ~/.rbenv

There are two commands to activate rbenv in your normal shell. The first appends rbenv's binaries and shims to your `$PATH`, the second ensures the shims are up-to-date and enables Bash command completion. There is a more detailed [neckbeard explanation](https://github.com/sstephenson/rbenv#section_2.3) in the README if you are interested. You probably want to activate this for every shell.

``` sh ~/.profile
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
```

New versions of Ruby are installed into `~/.rbenv/versions/`. The default method for doing so is to fetch, build and install yourself. Personally my days of typing `./configure && make && make install` commands are over. Thankfully there is a [ruby-build](https://github.com/sstephenson/ruby-build) plugin that provides routines for common Ruby versions and an easy `rbenv install` command.

There's no need to install this into your system `/usr/local` directory as the README suggests. Instead create an rbenv plugins directory and clone the project into there. Same rules about working from a tag or HEAD apply.

    mkdir ~/.rbenv/plugins
    git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build

You will obviously need a compiler to build Ruby. On Debian and Ubuntu you can use a single meta-package to install `gcc`, `make`, `libc-dev`, etc.

    sudo apt-get install build-essential

It's also preferable to compile Ruby against some common libraries. Without these you'll experience strange behaviour like Gems failing to load and broken `irb` command history. By installing these at a system level `autoconf` will find automagically find them.

    sudo apt-get install \
        libreadline-dev \
        libssl-dev \
        zlib1g-dev

Now you can install some Rubies. These are the versions that I seem to use most frequently.

    rbenv install 1.8.7-p357
    rbenv install 1.9.2-p290
    rbenv install 1.9.3-p0

There are three methods for utilising these new Rubies.

 * `rbenv global` can be used to permanently override the system's default version of Ruby.
 * `rbenv shell` can be used to temporarily set the version of Ruby used in a single shell session.
 * `rbenv local` can be used to set the version of Ruby within a specific project directory.

The last is the most common use case, at least for me. The preference is stored in a `.rbenv-version` file, which you can choose to commit to your repository for other developers to take advantage of.

## Gimme the Gems

The last part of the puzzle for me was separation of per-project Gem installs. I want complete isolation of dependencies between different projects. Previously I was using RVM Gemsets and later introduce Bundler. I hadn't entirely grokked the benefits of Bundler right away, but I now understand that it's dependency resolution is far more sophisticated than `gem install` and that it actually does away with the need for Gemsets altogether.

[rbenv-bundler](https://github.com/carsomyr/rbenv-bundler) is a near-invaluable plugin that makes the rbenv shims Bundler-aware and alleviates you from needing to type `bundler exec` in front of every command. It is also easy to install straight from Github.

    git clone git://github.com/carsomyr/rbenv-bundler.git ~/.rbenv/plugins/bundler

Bundler is the only Gem that you need install the traditional way. You will need to do this once for each version of Ruby that you have. The rehash command updates the shim for the bundle binary.

    gem install bundler
    rbenv rehash

The following config will tell Bundler to install all Gems into a single
path in your home directory. This isolates them from your normal system or
rbenv Ruby install when you're not using Bundler.

``` sh ~/.bundle/config
---
BUNDLE_PATH: ~/vendor/bundle
BUNDLE_DISABLE_SHARED_GEMS: "1"
```

Now you can call Bundler as normal to install all of your project's dependencies from it's respective `Gemfile`. Be sure to call rehash afterwards so that any new binaries are available through rbenv shims.

    bundle install
    rbenv rehash

It's worth noting that once you've install a version of a Gem for one
project it won't need to be re-downloaded for subsequent projects. At some
stage you can use this to run Bundler without being connected to the
internet.

    bundle install --local

## Putting it together

I want to test out some things against two different versions of [Vagrant](http://vagrantup.com). One from the 0.8 series and the other from the recent 0.9 series. I also want to use Ruby 1.9.3 instead of my system's installation of 1.8.7. This I can do quite simply with the switch of a directory context.

``` sh Vagrant 0.8.10 and Vbguest 0.0.3
dan@dan-MacPro:~$ mkdir ~/projects/vagrant/0.8
dan@dan-MacPro:~$ cd ~/projects/vagrant/0.8
dan@dan-MacPro:~/projects/vagrant/0.8$ rbenv local 1.9.3-p0
dan@dan-MacPro:~/projects/vagrant/0.8$ cat Gemfile
source "http://rubygems.org"
gem "vagrant", "~> 0.8"
gem "vagrant-vbguest", "~> 0.0.3"
dan@dan-MacPro:~/projects/vagrant/0.8$ bundle
Fetching source index for http://rubygems.org/
Installing archive-tar-minitar (0.5.2) 
Installing erubis (2.7.0) 
Installing ffi (1.0.11) with native extensions 
Installing i18n (0.6.0) 
Installing json (1.5.4) with native extensions 
Installing net-ssh (2.1.4) 
Installing net-scp (1.0.4) 
Installing thor (0.14.6) 
Installing virtualbox (0.9.2) 
Installing vagrant (0.8.10) 
Installing vagrant-vbguest (0.0.3) 
Using bundler (1.0.21) 
Updating .gem files in vendor/cache
Your bundle is complete! It was installed into ./vendor
dan@dan-MacPro:~/projects/vagrant/0.8$ rbenv rehash
dan@dan-MacPro:~/projects/vagrant/0.8$ vagrant --version
Vagrant version 0.8.10
```

``` sh Vagrant 0.9.5 and Vbguest 0.1.0
dan@dan-MacPro:~$ mkdir ~/projects/vagrant/0.9
dan@dan-MacPro:~$ cd ~/projects/vagrant/0.9
dan@dan-MacPro:~/projects/vagrant/0.9$ rbenv local 1.9.3-p0
dan@dan-MacPro:~/projects/vagrant/0.9$ cat Gemfile 
source "http://rubygems.org"
gem "vagrant", "~> 0.9"
gem "vagrant-vbguest", "~> 0.1.0"
dan@dan-MacPro:~/projects/vagrant/0.9$ bundle
Fetching source index for http://rubygems.org/
Installing archive-tar-minitar (0.5.2) 
Installing ffi (1.0.11) with native extensions 
Installing childprocess (0.3.1) 
Installing erubis (2.7.0) 
Installing i18n (0.6.0) 
Installing json (1.5.4) with native extensions 
Installing log4r (1.1.10) 
Installing net-ssh (2.2.2) 
Installing net-scp (1.0.4) 
Installing vagrant (0.9.5) 
Installing virtualbox (0.9.2) 
Installing vagrant-vbguest (0.1.1) 
Using bundler (1.0.21) 
Updating .gem files in vendor/cache
Your bundle is complete! It was installed into ./vendor
dan@dan-MacPro:~/projects/vagrant/0.9$ rbenv rehash
dan@dan-MacPro:~/projects/vagrant/0.9$ vagrant --version
Vagrant version 0.9.5
```

As a final step I wanted to make sure that my system was free of any globally installed Gems from the dark days of `sudo gem install`. I don't have any need for these on my workstation. But you should double check before doing the same.

    /usr/bin/gem list --no-versions | xargs sudo /usr/bin/gem uninstall
