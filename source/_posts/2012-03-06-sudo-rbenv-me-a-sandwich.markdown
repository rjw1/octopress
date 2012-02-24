---
layout: post
title: "sudo rbenv me a sandwich"
date: 2012-03-06 07:52
comments: true
categories: dev ruby
---

In the name of practising-what-you-preach, I like to be in the habit of configuring my Linux workstations with Puppet in the same way that I do for servers. Though I confess I have fallen out of habit recently and accumulated a bunch of tools that haven't made it back to the manifests. Having just trashed my laptop, in an unrelated incident, I fancied the opportunity to check out [chef-solo](http://wiki.opscode.com/display/chef/Chef+Solo) for bringing a fresh install of Mint Linux up to speed.

The Chef package currently available from Ubuntu's own package repositories is version 0.8, but I wanted to use the latest and greatest 0.10. That's simple to solve with my new-found love for rbenv and Bundler. Except one hitch - both `chef-solo` and it's cousin `puppet apply` need to be run as `root` in order to manage system resources.

## First attempts

If we simply run it with `sudo` then it unsurprisingly fails to find the `chef-solo` binary:

``` sh
$ sudo chef-solo --version
sudo: chef-solo: command not found
```

If we attempt to tell it where the binary shim is then we lose the magic of Bundler:

``` sh
$ sudo $(rbenv which chef-solo) --version
/usr/lib/ruby/1.8/rubygems.rb:779:in `report_activate_error': Could not find RubyGem chef (>= 0) (Gem::LoadError)
    from /usr/lib/ruby/1.8/rubygems.rb:214:in `activate'
    from /usr/lib/ruby/1.8/rubygems.rb:1082:in `gem'
    from /home/dan/projects/chef-solo/vendor/bin/chef-solo:18
```

If we attempt to pass it through the Bundler shim then we get this rather obtuse error caused by it using the system install of Ruby:

``` sh
$ sudo $(rbenv which bundle) exec chef-solo --version
ruby: symbol lookup error: /home/dan/projects/chef-solo/vendor/gems/json-1.6.1/ext/json/ext/json/ext/parser.so: undefined symbol: rb_intern2
```

## The problem

The reason this does not work out-of-the-box is that most operating systems distribute sudo compiled with the `secure_path` option enabled. This throws away the caller's `$PATH` environment variable and replaces it with a predefined list of search paths that are to be considered safe.

``` sh
$ sudo sudo -V | grep PATH
Value to override user's $PATH with: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/X11R6/bin
```

If you search around a little you'll find a handful of people complaining that this is a "bug" or "annoying feature" of some OSs like Ubuntu. On the contrary, it is there to protect you. Those predefined paths are considered safe because only the root user can drop new binaries into those locations. If your normal PATH variable has questionable contents like `./`, intentionally or otherwise, then they could supersede normal system binaries and be unwittingly executed as a privileged user.

Do not, whatever you do, disable this feature with `!secure_path`. It's enabled by default for a good reason. You can subvert it by resetting the variable for a given command. This is what the `rvmsudo` function provided by RVM does to pass a variety of environment variables over regardless of `secure_path` and `env_reset`. It's not ideal though.

## More attempts

Since we only want `rbenv` to work in the sudo session let's concentrate on that. We only need to pass on rbenv's binary and shim directories. This is effectively what rbenv's own init/setup does.

What I had originally hoped to do was prepend these to the value of `secure_path`. This is possible, but it's complicated somewhat by shell quoting:

``` sh
function rbsudo() {
    ARGS="$@"
    RBENV_ROOT=$(rbenv root)
    RBENV_PATH="${RBENV_ROOT}/shims:${RBENV_ROOT}/bin"
    sudo sh -c "env PATH=\"${RBENV_PATH}:\${PATH}\" $ARGS"
}
```

That works for simple cases. But pre-quoted arguments get stripped of their quotes:

``` sh
$ rbsudo ruby --version
ruby 1.9.3p0 (2011-10-30 revision 33570) [x86_64-linux]
$ rbsudo ruby -e "puts 'hello'"
    
```

We can mitigate this slightly by identifying arguments that contain whitespace and were thus previously quoted, then re-quoting them:

``` sh
function rbsudo() {
    ARGS=$1
    shift 1
    for ARG in "$@"; do
        [[ $ARG =~ [[:space:]] ]] && ARG="\"$ARG\""
        ARGS="$ARGS $ARG"
    done
    RBENV_ROOT=$(rbenv root)
    RBENV_PATH="${RBENV_ROOT}/shims:${RBENV_ROOT}/bin"
    sudo sh -c "env PATH=\"${RBENV_PATH}:\${PATH}\" $ARGS"
}
```

Which gets us close, but not perfect:

```sh
$ rbsudo ruby -e "puts 'hello'"
hello
$ rbsudo ruby -e 'puts "hello"'
-e:1:in `<main>': undefined local variable or method `hello' for main:Object (NameError)
```

We could preserve those with some more hackery but at this point I decided it probably wasn't worth it. How about using the same `eval command` trick that `rvmsudo` does. Sure eval is pretty evil and it in-turn means that we can't evaluate the `$PATH` within the sudo session. However a predefined list seems like the next best thing:

``` sh
function rbsudo {
    RBENV_ROOT=$(rbenv root)
    ROOT_PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ROOT_PATH="${RBENV_ROOT}/shims:${RBENV_ROOT}/bin:${ROOT_PATH}"
    eval command sudo env PATH=\"\$ROOT_PATH\" \"\$@\"
}
```

Now this works as expected:

``` sh
$ rbsudo ruby -e "puts 'hello'"
hello
$ rbsudo ruby -e 'puts "hello"'
hello
```

## Final solution

rbenv provides us with two niceties to wrap this up with. The first is plugin support which means that instead of using a Bash function we can create a sub-command of `rbenv sudo` simply by dropping a shell script into the right place. Furthermore all rbenv plugins have access to an `RBENV_ROOT` variable which saves us from calling out to `$(rbenv root)`. So we're left with the following code:

``` sh ~/.rbenv/plugins/rbenv-sudo/bin/rbenv-sudo
# Bail on all errors and undefined variables.
set -eu

# Construct our new PATH.
ROOT_PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ROOT_PATH="${RBENV_ROOT}/shims:${RBENV_ROOT}/bin:${ROOT_PATH}"

# Execute the command.
eval command sudo env PATH=\"\$ROOT_PATH\" \"\$@\"
```

I've made plugin this available on Github as [dcarley/rbenv-sudo](https://github.com/dcarley/rbenv-sudo). Any comments or pull requests are welcomed.
