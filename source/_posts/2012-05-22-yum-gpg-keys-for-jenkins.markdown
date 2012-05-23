---
layout: post
title: "Yum GPG keys for Jenkins"
date: 2012-05-22 21:29
comments: true
categories: ops centos rhel yum puppet jenkins
---

I use/mirror a number of third-party Yum repositories. The typical pattern for these is to import the vendor's public signing key using `rpm --import` and then setup a new entry `/etc/yum.repos.d` entry. Some vendors have `-release` packages that handle both steps. The trouble is, this creates a bit of a chicken/egg problem and doesn't lend itself so well to automation.

Instead I tend to favour having Puppet manage the keys and repositories. Since the GPG keys don't change very often we can safely import them into Puppet. Puppet pushes out the key as a file resource before setting up a Yum repository resource with the attribute `gpgkey = file:///`.

Here is what the resulting class for Jenkins looks like.

```
class yum::repos::jenkins {
    $yum_jenkins_url = extlookup('yum_jenkins_url', 'http://pkg.jenkins-ci.org/redhat-stable/')
    $yum_jenkins_gpg = '/etc/pki/rpm-gpg/RPM-GPG-KEY-jenkins'

    file { $yum_jenkins_gpg:
        source  => "puppet:///modules/yum${yum_jenkins_gpg}",
    }
    yumrepo { 'jenkins':
        descr       => 'Jenkins',
        baseurl     => $yum_jenkins_url,
        gpgcheck    => 1,
        gpgkey      => "file://${yum_jenkins_gpg}",
        require     => File[$yum_jenkins_gpg],
    }
}
```

## The problem

This pattern works fine for most vendors. It works fine for Jenkins on EL6. But it throws the following error on EL5.

``` sh
notice: /File[/etc/pki/rpm-gpg/RPM-GPG-KEY-jenkins]/ensure: defined content as '{md5}9fa06089848262c5a6383ec27fdd2575'
notice: /Stage[main]/Yum::Repos::Jenkins/Yumrepo[jenkins]/descr: descr changed '' to 'Jenkins'
notice: /Stage[main]/Yum::Repos::Jenkins/Yumrepo[jenkins]/baseurl: baseurl changed '' to 'http://pkg.jenkins-ci.org/redhat-stable/'
notice: /Stage[main]/Yum::Repos::Jenkins/Yumrepo[jenkins]/gpgcheck: gpgcheck changed '' to '1'
notice: /Stage[main]/Yum::Repos::Jenkins/Yumrepo[jenkins]/gpgkey: gpgkey changed '' to 'file:///etc/pki/rpm-gpg/RPM-GPG-KEY-jenkins'
err: /Stage[main]/Jenkins::Package/Package[jenkins]/ensure: change from absent to present failed: Execution of '/usr/bin/yum -d 0 -e 0 -y install jenkins' returned 1: warning: rpmts_HdrFromFdno: Header V4 DSA signature: NOKEY, key ID d50582e6
Traceback (most recent call last):
  File "/usr/bin/yum", line 29, in ?
    yummain.user_main(sys.argv[1:], exit_code=True)
  File "/usr/share/yum-cli/yummain.py", line 309, in user_main
    errcode = main(args)
  File "/usr/share/yum-cli/yummain.py", line 261, in main
    return_code = base.doTransaction()
  File "/usr/share/yum-cli/cli.py", line 410, in doTransaction
    if self.gpgsigcheck(downloadpkgs) != 0:
  File "/usr/share/yum-cli/cli.py", line 510, in gpgsigcheck
    self.getKeyForPackage(po, lambda x, y, z: self.userconfirm())
  File "/usr/lib/python2.4/site-packages/yum/__init__.py", line 3544, in getKeyForPackage
    keys = self._retrievePublicKey(keyurl, repo)
  File "/usr/lib/python2.4/site-packages/yum/__init__.py", line 3509, in _retrievePublicKey
    keys_info = misc.getgpgkeyinfo(rawkey, multiple=True)
  File "/usr/lib/python2.4/site-packages/yum/misc.py", line 375, in getgpgkeyinfo
    raise ValueError(str(e))
ValueError: unknown pgp packet type 17 at 706
```

Running `yum install jenkins` by hand results in exactly the same error. So we know it's not Puppet getting in the way. Following the [jenkins-ci.org instructions](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+on+RedHat+distributions) of running `rpm --import` and setting up the repository without `gpgkey` also works. So what's the difference?

## Some digging

The best place to start would be the code in the traceback. Looking at `__init__.py:getKeyForPackage()` we can see that if Yum is given a `gpgkey` attribute it will make sure that the key is imported. It determines the presence and some other information about the key by calling `misc.py:getpgpkeyinfo()` which in turn uses `pgpmsg.py` to do the actual parsing. At the top of that file is a list of numeric packet types as constants.

For Yum 3.2.22, which is used in EL5, these numerics only go as high as 14. Which would explain why it's complaining that 17 is unknown. By running `git blame` and `git describe` against [Yum's git repository](http://yum.baseurl.org/wiki/#Topullanonymouslyfromgitdothefollowing) we can see that the [additional packet types](http://yum.baseurl.org/gitweb?p=yum.git;a=commitdiff;h=4f50718ece4b8071ee380f2bbd03d0d16605183a) were first present in Yum 3.2.25. Which explains why we don't see the same problem in EL6 which uses Yum 3.2.29.

So what is packet type 17? The IANA assignments for [PGP Packet Types/Tags](http://www.iana.org/assignments/pgp-parameters/pgp-parameters.xml#pgp-parameters-2) tell us that it is a "User Attribute Packet". The assignments for [PGP User Attribute Types](http://www.iana.org/assignments/pgp-parameters/pgp-parameters.xml#pgp-parameters-3) also tell us that there's only one registered type of user attribute, suggesting that it must be an image.

## The solution

It seems that an image shouldn't be essential to our use of verifying package integrity. The good news is that we can remove it from the public key.

Start by obtaining a copy of the original public key.

``` sh
$ curl http://pkg.jenkins-ci.org/redhat-stable/jenkins-ci.org.key > RPM-GPG-KEY-jenkins
```

Import it into GPG. The version of GPG that you use shouldn't matter too much. I'm using a more recent version on my Mint desktop machine.

``` sh
$ gpg --import RPM-GPG-KEY-jenkins
gpg: /home/dan/.gnupg/trustdb.gpg: trustdb created
gpg: key D50582E6: public key "Kohsuke Kawaguchi <kk@kohsuke.org>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

List the available keys to determine it's identity. We're after that hex value `D50582E6`.

``` sh
$ gpg --list-keys
/home/dan/.gnupg/pubring.gpg
----------------------------
pub   1024D/D50582E6 2009-02-01
uid                  Kohsuke Kawaguchi <kk@kohsuke.org>
uid                  Kohsuke Kawaguchi <kohsuke.kawaguchi@sun.com>
uid                  [jpeg image of size 3704]
sub   2048g/10AF40FE 2009-02-01
```

Begin editing that key. We can see that there are three IDs within the key, the last one being the image that we want to remove. Mark it, delete it, and save the modifications.

``` sh
$ gpg --edit-key D50582E6
gpg (GnuPG) 1.4.11; Copyright (C) 2010 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.


pub  1024D/D50582E6  created: 2009-02-01  expires: never       usage: SC  
                     trust: unknown       validity: unknown
sub  2048g/10AF40FE  created: 2009-02-01  expires: never       usage: E   
[ unknown] (1). Kohsuke Kawaguchi <kk@kohsuke.org>
[ unknown] (2)  Kohsuke Kawaguchi <kohsuke.kawaguchi@sun.com>
[ unknown] (3)  [jpeg image of size 3704]

gpg> uid 3

pub  1024D/D50582E6  created: 2009-02-01  expires: never       usage: SC  
                     trust: unknown       validity: unknown
sub  2048g/10AF40FE  created: 2009-02-01  expires: never       usage: E   
[ unknown] (1). Kohsuke Kawaguchi <kk@kohsuke.org>
[ unknown] (2)  Kohsuke Kawaguchi <kohsuke.kawaguchi@sun.com>
[ unknown] (3)* [jpeg image of size 3704]

gpg> deluid
Really remove this user ID? (y/N) y

pub  1024D/D50582E6  created: 2009-02-01  expires: never       usage: SC  
                     trust: unknown       validity: unknown
sub  2048g/10AF40FE  created: 2009-02-01  expires: never       usage: E   
[ unknown] (1). Kohsuke Kawaguchi <kk@kohsuke.org>
[ unknown] (2)  Kohsuke Kawaguchi <kohsuke.kawaguchi@sun.com>

gpg> save
```

Export the modifications to a new ASCII key file. We'll give it a different filename to the original.

``` sh
$ gpg --export -a D50582E6 > RPM-GPG-KEY-jenkins.el5
```

Now we can send out the modified key to any EL5 nodes and use the original for everyone else. Voila, the Puppet run completes and the package is installed successfully.

```
$yum_jenkins_gpg = $::operatingsystemmajrelease ? {
    '5'     => '/etc/pki/rpm-gpg/RPM-GPG-KEY-jenkins.el5',
    default => '/etc/pki/rpm-gpg/RPM-GPG-KEY-jenkins',
}
```

If you're wondering, `operatingsystemmajrelease` is a [custom fact](https://gist.github.com/1778618). It works much like `lsbmajdistrelease` without the gazillion dependencies.
