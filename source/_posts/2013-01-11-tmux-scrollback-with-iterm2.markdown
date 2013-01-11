---
layout: post
title: "tmux scrollback with iTerm2"
date: 2013-01-11 07:44
comments: true
categories: dev ops osx
---

## tmux scrollback 

If you regularly use [tmux](http://tmux.sourceforge.net/) then you might have a line like this in your config:

``` text ~/.tmux.conf
set -g terminal-overrides 'xterm*:smcup@:rmcup@'
```

The effect of this is that when the output of the inner terminal exceeds the terminal's height it is allowed to spill over into the outer terminal's scrollback history. So long as you don't change windows within the tmux session you can use the scrollbar of your local terminal to review the history.

Under the covers this disables the inner terminal's `smcup` and `rmcup` capabilities when `ENV['TERM'] =~ /^xterm/`. These capabilities are responsible for saving and restoring terminal history/state. By disabling them the output is allowed to spill over.

## iTerm2

It seems all is well until it comes to using iTerm2 on OSX. Suddenly scrolling back in the outer terminal shows history from prior to the start of tmux. There are no end of suggestions about how to fix this, including "disable the status bar" and "it should just work". Actually it's fairly simple.

Enable the option under `Preferences -> Profiles -> Terminal` called:

> Save lines to scrollback when an app status bar is present

To save doing this again in future you can copy and version control the config file at `~/Library/Preferences/com.googlecode.iterm2.plist`
