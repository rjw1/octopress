---
layout: post
title: "Bootstrapping EC2 with Puppet"
date: 2012-08-14 07:37
comments: true
categories: ops dev aws ec2 centos puppet
---

In a previous article I described the process of creating CentOS AMIs in EC2. I omitted the detail of how this single generic system image is turned around into real functioning nodes. This consists of a few things.

A standard set of routines which you'll find in most AMIs:

1. Create a non-privileged `ec2-user`
1. Copy an SSH pubkey from the metadata API to the `ec2-user` account
1. Configure sudoers rules for the `ec2-user`
1. Disable SSH password authentication and root logins
1. Log SSH host key fingerprints

Plus some routines that are more specific to our use case:

1. Configure the node's hostname
1. Register the node's hostname in DNS
1. Configure the Puppet agent
1. Run the Puppet agent against a master

There are a handful of ways to skin this cat.

## TL;DR

All the code is available on GitHub at:

- [dcarley/puppet-ec2init](https://github.com/dcarley/puppet-ec2init)
- [dcarley/ec2ddns](https://github.com/dcarley/ec2ddns)

## CloudInit

CloudInit deserves a mention first. It's a framework that supports most of these routines. It's written in Python, making it relatively easy to debug and extend. It's available in many Ubuntu and Debian AMIs, as well as the AWS own-brand "Amazon Linux" which is EL based.

The downsides? Documentation is poor. It's probably more bloated and complex than it needs to be. It's hosted on Launchpad. And, most importantly, it's a pain to get working on RHEL and CentOS. Despite the obvious efforts that Amazon have invested internally.

I've used it in the past. I would probably use it again if it was already present in a vendor/upstream AMI and didn't require any customisation. But it no longer meets my needs here.

## Shell scripting

The second most popular approach is to use shell scripting, as indeed suggested by [Amazon's own documentation](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/AESDG-chapter-sharingamis.html#public-amis-install-credentials). On the face of it that might seem alright. Fetching an HTTP resource and writing a file isn't exactly Computer Science material, I can hear you say. You'd be right - it's not. Though as ever the devil is in the detail.

Soon enough you'll discover that the metadata API isn't always ready to serve the SSH pubkey before an EC2 instance has finished booting. Now you have to add a retry mechanism and some reporting for failures. You need to ensure that the files you create have the correct ownership, permissions and SELinux file context labels. You need to restart the SSH daemon every time the configuration file changes. For bonus points it'd be nice not to replace files if they already have the correct content.

Recognise that feature list? You may have just written your own crude Configuration Management framework. In shell script.

## Puppet

Puppet (or Chef for that matter) already has support for querying or managing these primitive types in an idempotent manner. It has many years experience in logging, error handling and dealing with weird edge cases. Facter has plenty of EC2 specific support. We already need to install Puppet to do the rest of our system configuration and we're comfortable with Puppet's code syntax. So why not use it here too..

I set about writing a single Puppet module with sub-classes for each of these tasks.

### Standard tasks

The first five were very simple to achieve. Here they are broken down by sub-class:

#### [ec2init::user](https://github.com/dcarley/puppet-ec2init/blob/master/manifests/user.pp)

The user and group can be managed with Puppet's standard `user{}` and `group{}` types.

The SSH public key is already available from Facter as `$::ec2_public_keys_0_openssh_key`. This can be written to `authorized_keys` with a simple `file{}` resource.

#### [ec2init::ssh](https://github.com/dcarley/puppet-ec2init/blob/master/manifests/ssh.pp)

The host's public SSH keys are also available from Facter, but not in a form that's an easily digestible by humans. New custom Facts [*sshrsafp* & *sshdsafp*](https://github.com/dcarley/puppet-ec2init/blob/master/lib/facter/sshfp.rb) were needed to convert these into hexadecimal fingerprints like `ssh(1)` produces. These could then be logged with Puppet's standard functions.

Changes to the two `sshd_config` values can be made with `augeas{}` and the service subsequently restarted.

#### [ec2init::sudo](https://github.com/dcarley/puppet-ec2init/blob/master/manifests/sudo.pp)

Sudo rules can be managed as simple file resources in the `conf.d` convention. `augeas{}` is used to ensure that `includedir` is enabled, which is the default for EL6 and later.

### Augeas

I said the dreaded word Augeas. I'm normally a quite vocal opponent of Augeas. I'll never forget [this quote on puppet-users](https://groups.google.com/d/msg/puppet-users/RnW4BeS-2Yw/L7dJBLhsL54J) which pretty much sums it up for me:

> I'll just say that Augeas is named after a man who let his stables fill up with so much crap that Hercules had to divert two whole rivers to pressure-hose it clean again.  You can't make this stuff up.

However I seem to have found the one exception to my hatred. I didn't want to manage the whole files here because it would have duplicated content in our normal Puppet modules. Augeas allowed me to specify the individual keys that were relevant to bootstrapping whilst not stepping on the toes of the modules that will be applied later.

### Additional tasks

The other four required a little more work. Most required some kind of outside influence that would be defined when creating EC2 instances, such as a node's hostname or Puppet environment. I followed the standard approach for this which is to populate an instance's user-data. Tags could have been used but they aren't so easy to read from within the instances.

Making this available to Puppet was fairly straightforward. Facter already presents a variable called `$::ec2_userdata`. JSON seemed like a good choice of data format because it is insensitive to whitespace changes and the Fact would be squashed into a single line. Two functions were borrowed from [puppetlabs-stdlib](https://github.com/puppetlabs/puppetlabs-stdlib) to parse and evaluate this data:

- [*parse_userdata()*](https://github.com/dcarley/puppet-ec2init/blob/master/lib/puppet/parser/functions/parse_userdata.rb): to parse the JSON into a Puppet's native hash structure.
- [*has_key()*](https://github.com/dcarley/puppet-ec2init/blob/master/lib/puppet/parser/functions/has_key.rb): to check for the presence of nested keys.

These hash keys are then evaluated for presence and assigned to variables in a central [*ec2init::params*](https://github.com/dcarley/puppet-ec2init/blob/master/manifests/params.pp) class. Behaviour of other classes are then conditional on the absence or presence of these variables.

#### [ec2init::puppet](https://github.com/dcarley/puppet-ec2init/blob/master/manifests/puppet.pp)

For a user-data field containing:

``` ruby
{
    "puppet": {
        "server": "puppet.example.com",
        "environment": "bar"
    }
}
```

Puppet's `puppet.conf` configuration is modified with simple `augeas{}` resources.

#### [ec2init::hostname](https://github.com/dcarley/puppet-ec2init/blob/master/manifests/hostname.pp)

For a user-data field containing:

``` ruby
{
    "hostname": "foo.bar.example.com"
}
```

There are a number of places to set the hostname on an EL system. The first is `/etc/sysconfig/network` which is used when networking is started through SysVinit and can be handled with `augeas{}`. The second is `/etc/hosts` which needs an IP entry for the new hostname and can be handled with the `host{}` type. Lastly we have to make the system aware of the change without restarting networking, which calls for an `exec{}` to `hostname(1)`.

In addition to this we also want to set a new domainname if we're able to determine it. This can be useful for referencing other nodes in the same location/demarcation by their unqualified names, for example using CNAME records to locate the correct Puppet master. One more custom function is used to attempt to extract the domainname from the hostname. If successful, `resolv.conf` is updated with `augeas{}` to use the new search domain and `dhclient.conf` is created with a `file{}` resource to supersede the search domain on subsequent DHCP leases.

#### [ec2init::ddns](https://github.com/dcarley/puppet-ec2init/blob/master/manifests/ddns.pp)

For a user-data field containing:

``` ruby
{
    "hostname": "foo.bar.example.com",
    "route53": {
        "aws_access_key_id": "FOOFOOFOOFOOFOOFOOFO",
        "aws_secret_access_key": "BARBARBARBARBARBARBARBARBARBARBARBARBARB"
    }
}
```

An external script is called to register the DNS entry. It takes arguments for the two AWS keys and custom hostname from user-data, along with the public hostname variable `$::ec2_public_hostname` from Facter. The AWS keys have a restrictive IAM policy that only allows them to talk to Route 53.

### Hostnames & DDNS

The reason for setting hostnames and registering entries in DNS mostly stems from our role-based naming convention. Databases are called `dbN`, message brokers `mqN`, web caches `wcN`, etc. Where `N` is an index of horizontal nodes of the same role. These have two uses:

1. **Functional:** Our `node{}` entries in Puppet are grouped by these role names. New capacity can be added and classified by creating an instance with the correct naming convention. The hostnames can also be used for inter-machine connectivity.
1. **Vanity:** It can be useful to refer to the role portion of the hostname when reviewing log files or collating metrics.

Setting up our own BIND (or something) server to do DDNS seemed like a lot of hassle. Amazon's Route 53 service has an API for easily creating and deleting records. All I'd need to do is write a small wrapper script and provide it some restricted credentials. I originally set about writing this in Ruby/Fog, but getting these to work on a minimal CentOS AMI proved too much work. I went back to using Python/Boto which are already available as packages.

Thus was born [dcarley/ec2ddns](https://github.com/dcarley/ec2ddns). It takes four arguments; an access key, secret key, the hostname you want to register, and the public hostname of the instance you're registering. It creates a CNAME entry with a very low TTL. Amazon perform split-horizon resolution on the public hostname that it points to, meaning that outside the EC2 network you'll get the public IP address and inside EC2 you'll get the private IP that isn't charged for same-AZ transfers. When delete is called the CNAME is changed to a non-existent destination so that traffic isn't sent to IPs that don't belong to us, whilst keeping a record that the node once existed and reducing negative caching time.

### DDNS Risks

It's worth noting that there is an inherent risk with passing credentials in through user-data. Access to the metadata API from each instance is unrestricted, meaning that any user or process in the VM could capture those credentials. The contents are also freely reported by Facter and sent back to the Puppet Master which may surface them in other reports.

There's not a lot you can do to prevent one instance from maliciously updating the DNS record of another instance. Though some risk can be mitigated by using separate IAM accounts, policies and hosted zones for each logical environment.

### RPM & RC

A relatively simple [SysVinit script](https://github.com/dcarley/puppet-ec2init/blob/master/puppet-ec2init.init) is used to ensure that the modules are applied on every boot. Unfortunately it was still necessary to check that the metadata API was reachable before the modules were applied, so curl is used to do this. Then `puppet apply` is called to run `ec2init::start` which includes the relevant classes. Once successful `puppet agent --onetime` is called to perform a normal run against our Puppet Master. When an instance is shut down it calls `ec2init::stop` which just de-registers the DDNS entry.

This is all wrapped up with a simple [SPEC file](https://github.com/dcarley/puppet-ec2init/blob/master/puppet-ec2init.spec) so that it can be installed as an RPM. I also included [rspec-puppet tests](https://github.com/dcarley/puppet-ec2init/blob/master/spec) for functions and classes where possible.

## In closing

I guess the moral of the story is that you can't always avoid re-inventing something yourself. But you can try your damnedest to re-use what's already available to you.

Once again all the code is available on GitHub at [dcarley/puppet-ec2init](https://github.com/dcarley/puppet-ec2init) and [dcarley/ec2ddns](https://github.com/dcarley/ec2ddns)
