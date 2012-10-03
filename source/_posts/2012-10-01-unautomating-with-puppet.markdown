---
layout: post
title: "Unautomating with Puppet"
date: 2012-10-01 07:44
comments: true
categories: ops dev puppet immutable unautomating mysql riak
---

Github's recent outage and [nicely detailed post-mortem](https://github.com/blog/1261-github-availability-this-week) has generated a bit of chatter about whether "too much automation" is a thing and the dangers thereof:

- [Xaprb: Is automated failover the root of all evil?](http://www.xaprb.com/blog/2012/09/17/is-automated-failover-the-root-of-all-evil/)
- [Scale-Out: Automated Database Failover Is Weird but not Evil](http://scale-out-blog.blogspot.co.uk/2012/09/automated-database-failover-is-weird.html)
- [Kitchen Soap: A Mature Role for Automation: Part I](http://www.kitchensoap.com/2012/09/21/a-mature-role-for-automation-part-i/)

This a topic that most operations folk can sympathise with. You can't begin to automate something until you have a darn good idea of it's operating parameters. Then you need to be extra darn sure that whatever you've automated will do the Right Thing in the absence of any humans.

It's something that I'm acutely aware of whenever we bring new technology into our stack. For initial releases my response has often been "no, we can't automate that part". Not because it's impossible - it's just not possible to do *well*. Where the benchmark for "well" is being happy to throw it into production and go to bed with a pager.

## Examples

### The daring

Some things are risky to automate. Take MySQL's `innodb_log_file_size` setting for example. Parameterising it with Puppet for a new installation is easy. Subsequently modifying the value is much harder and involves:

1. Shutting down MySQLd cleanly.
1. Ensuring that there are no outstanding transactions in the logs.
1. Deleting the old logs.
1. Changing the configuration.
1. Starting MySQLd again.

I haven't seen an existing Puppet module that does this. Nor have I dared to write one of my own.

### The stupid

Even with all the code and testing in the world there are some routines that absolutely need human intervention. Riak's `ring_creation_size` setting is one of these. The value has to be the same for all cluster members and can't be subsequently changed without forming a new cluster and migrating the data across.

I don't want to undertake this by hand on a good day. I certainly don't want it happening while I sleep, regardless of how much testing my code has undergone.

### Bit of both

Continuing the theme, Riak uses Erlang cookies for two purposes..

Local administration tools such as SysVinit scripts and `riak-admin` use the cookie to authenticate and communicate with the local process. Changing this value involves:

1. Shutting down the Erlang process with the present cookie.
1. Changing the configuration.
1. Starting the Erlang process with the new cookie.

Communication between nodes relies on the cookie to verify that they belong to the same cluster. Thus changing the cookie also implies changing cluster membership and considering:

1. Is the node already in a cluster?
1. Is the node in the right cluster?
1. Is the node experiencing a transient problem?
1. Will leaving the present cluster affect availability?
1. Will joining a new cluster cause an untimely rebalance?

It seems both daring and a little stupid to manage that combination automatically. The simplest solution is to just ensure that the cookie isn't modified without some fail-safe human intervention.

## Solution

### Immutable values

Here's a simple pattern that I use for managing such situations in Puppet. It treats the value as immutable - once initially set it can't be changed. Sticking with the last example..

A custom fact reports the current value from the configuration file where available. Since the Erlang cookie is sensitive information and Puppet facts can be stored and reported in plain text, we take a simple hash of the cookie value.

``` ruby riak/lib/facter/riak_cookie_hash.rb
require 'digest/sha1'

Facter.add(:riak_cookie_hash) do
  setcode do
    config = '/etc/riak/vm.args'
    cookie = nil

    if File.exists?(config)
      File.readlines(config).each do |line|
        if line =~ /^-setcookie\s+(\S+)$/
          cookie = Digest::SHA1.hexdigest($1)
          break
        end
      end
    end

    cookie
  end
end
```

The manifest which manages the config file takes a externally-provided cookie parameter. An `if{}` condition checks for the presence of the fact above. If the fact isn't present then the package and default configuration probably haven't been installed yet, since facts are evaluated client-side before the catalog is compiled. The fact is then compared against a hash of the param. If they don't match then `fail()` is called to abort the catalog and prevent the configuration from being changed.

``` ruby riak/manifests/init.pp
class riak($cookie) {
  if ($::riak_cookie_hash) and (sha1($cookie) != $::riak_cookie_hash) {
      fail('Riak cookie is already present. Cowardly refusing to modify value')
  }

  file { '/etc/riak/app.config':
    ensure  => present,
    content => template('riak/etc/riak/app.config.erb'),
  }

  …
}
```

Now the process of changing an existing node's cookie requires a human to shutdown the node, edit or remove the configuration, and let the Puppet agent run again.

### Improvements

The use of Puppet's `fail()` function is quite brutal. It will prevent the entire catalog from being compiled and no resources will be managed. I've long pined after a way to deliberately fail a single resource on the client side - a soft failure of sorts. This would ensure that a single resource and those that depend on it wouldn't be modified, but the rest of the catalog would continue to be enforced. You could do something with an `if/else` condition and `notify{}` but it's not the same as generating an actual error.

I've now filed feature request [#16647](https://projects.puppetlabs.com/issues/16647) for this functionality.

The other thing to be aware of is that partial installs will require manual intervention. If the package has been pre-installed, or a Puppet agent bombs out between installing the package and copying the configuration file, then the config will need either removing or editing to continue. There is also a very minor risk that if the fact didn't copy or execute for some freak reason then the config would be unexpectedly replaced. But these are edge cases that probably aren't worth fretting about too much.
