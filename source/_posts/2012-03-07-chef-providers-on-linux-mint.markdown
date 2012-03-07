---
layout: post
title: "Chef providers on Linux Mint"
date: 2012-03-07 08:13
comments: true
categories: chef ops dev
---

I was attempting to install a package on Linux Mint using chef-solo when I was greeted by this rather strange error:

``` ruby
[Tue, 06 Mar 2012 17:53:26 +0000] INFO: Processing package[openssh-client] action install (ssh::client line 1)
[Tue, 06 Mar 2012 17:53:26 +0000] ERROR: package[openssh-client] (ssh::client line 1) has had an error
[Tue, 06 Mar 2012 17:53:26 +0000] ERROR: package[openssh-client] (chef-solo/cookbooks/ssh/recipes/client.rb:1:in `from_file') had an error:
package[openssh-client] (ssh::client line 1) had an error: Chef::Exceptions::Override: You must override load_current_resource in #<Chef::Provider::Package:0x00000003039a00>
chef-solo/vendor/gems/chef-0.10.8/lib/chef/provider.rb:51:in `load_current_resource'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource.rb:439:in `run_action'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/runner.rb:45:in `run_action'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/runner.rb:81:in `block (2 levels) in converge'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/runner.rb:81:in `each'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/runner.rb:81:in `block in converge'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource_collection.rb:94:in `block in execute_each_resource'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource_collection/stepable_iterator.rb:116:in `call'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource_collection/stepable_iterator.rb:116:in `call_iterator_block'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource_collection/stepable_iterator.rb:85:in `step'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource_collection/stepable_iterator.rb:104:in `iterate'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource_collection/stepable_iterator.rb:55:in `each_with_index'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/resource_collection.rb:92:in `execute_each_resource'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/runner.rb:76:in `converge'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/client.rb:312:in `converge'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/client.rb:160:in `run'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/application/solo.rb:192:in `block in run_application'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/application/solo.rb:183:in `loop'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/application/solo.rb:183:in `run_application'
chef-solo/vendor/gems/chef-0.10.8/lib/chef/application.rb:67:in `run'
chef-solo/vendor/gems/chef-0.10.8/bin/chef-solo:25:in `<top (required)>'
chef-solo/vendor/bin/chef-solo:19:in `load'
chef-solo/vendor/bin/chef-solo:19:in `<main>'
```

Thankfully the first result for the search term `chef "override load_current_resource"` is the bug [CHEF-1152](http://tickets.opscode.com/browse/CHEF-1152). This details a problem with Scientific Linux not being recognised as a derivative of Redhat Enterprise Linux.

As a result of this, Chef doesn't know which providers should be used on that platform to install packages, start services, etc. The ticket also includes a patch which adds the relevant support to the file `lib/chef/platform.rb`

``` ruby
def platforms
  @platforms ||= {
..
    :scientific => {
      :default => {
        :service => Chef::Provider::Service::Redhat,
        :cron => Chef::Provider::Cron,
        :package => Chef::Provider::Package::Yum,
        :mdadm => Chef::Provider::Mdadm
      }
    },
```

Now that we know what's caused that error we have a better idea of what's happening on Mint. A search for `chef providers mint` turns up bug [CHEF-2697](http://tickets.opscode.com/browse/CHEF-2697) which notes that it is due to be fixed in version 0.10.10. But what if we want to fix it now without modifying the Chef codebase?

The first step is to find out what Ohai thinks our platform is:

``` ruby
irb(main):001:0> require 'ohai'
irb(main):002:0> o = Ohai::System.new
irb(main):003:0> o.all_plugins
irb(main):004:0> p [o.platform, o.platform_version]
["linuxmint", "12"]
```

A search of `Chef::Platform` confirms that there are no references to `linuxmint`. There is an entry for Ubuntu from which Mint is derived. Now we could cheat and modify `/etc/lsb-release`, where that string comes from, to pretend that we are Ubuntu. But *don't* do that, it is likely to cause more problems than it solves.

[Chef::Platform](http://rubydoc.info/gems/chef/0.10.8/Chef/Platform) gives us two helpful methods [set()](http://rubydoc.info/gems/chef/0.10.8/Chef/Platform#set-class_method) and [find_provider()](http://rubydoc.info/gems/chef/0.10.8/Chef/Platform#find_provider-class_method). These can be used to register Mint in the same way as Ubuntu. We loop over the four resource types in question, looking up the default value for Ubuntu and setting the default for Mint.

``` ruby solo.rb
require 'chef/platform'

[:package, :service, :cron, :mdadm].each do |resource_type|
    Chef::Platform.set(
        :platform => :linuxmint,
        :resource => resource_type,
        :provider => Chef::Platform.find_provider(:ubuntu, :default, resource_type)
    )
end
```

I'm only using chef-solo so I've placed this in my `solo.rb`. However from my understanding this could also be done in `client.rb`, a library within a cookbook, or an attributes file. Hopefully that should get you going until 0.10.10 rolls around.
