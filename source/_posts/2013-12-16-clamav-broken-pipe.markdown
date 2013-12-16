---
layout: post
title: "ClamAV broken pipe"
date: 2013-12-16 08:23
comments: true
categories: ops ubuntu clamav pam_tmpdir apparmor
---

A while ago we upgraded ClamAV at $JOB from an [fpm][fpm] package of
dubious-quality to an upstream Ubuntu package. In doing so we found that one
of our applications, which submits data to `clamd` over a socket, had
stopped working and was logging `Broken pipe` errors.

The following text is a paraphrase of a commit message to one of our
internal repositories. It seems that this problem still isn't well
represented by Google-sauce, so hopefully this will be useful to others.

[fpm]: https://github.com/jordansissel/fpm

## Debugging

After some fiddling a colleague found that we could reproduce this on the
command line by streaming content in from a file:

    dcarley@preview-asset-master:~$ echo foo > foo
    dcarley@preview-asset-master:~$ clamdscan --stream foo
    ERROR: Can't send to clamd: Broken pipe
    /home/dcarley/foo: no reply from clamd
    ...

In the midst of testing some theories I restarted the service and suddenly
it started working again. I hypothesised that it might be a race condition
created by the upgrade process. However I wasn't able to reproduce problem
by performing the same upgrade on a [Vagrant][vagrant] VM.

[vagrant]: http://www.vagrantup.com/

Like many a desperate sysadmin I reached for `strace` to inspect the `clamd`
process while re-running the `clamdscan` command, which revealed:

    dcarley@preview-asset-master:~$ sudo strace -vfp $(pgrep clamd)
    ...
    [pid  5279] open("/tmp/user/0/clamav-e41b103da0dd009d881b8f4777da7902", O_RDWR|O_CREAT|O_EXCL|O_TRUNC, 0700) = -1 EACCES (Permission denied)

I rewound a little and read up on what ClamAV was actually doing. When
`clamd` receives a stream instead of being passed an open file handle, which
is the default behaviour, it buffers the contents to a temporary file and
then reads it back as it's own leisure. However the `/tmp/user/0` directory
that it's trying to write to is owned by `root:root` and the `clamd` process
running as `clamav:clamav` doesn't have permission.

It turns out that those `/tmp/user/${UID}` directories are created and two
environment variables are set (`$TMP` and `$TMPDIR`) by PAM's `pam_tmpdir`
for new sessions. If `TMPDIR` is set then `clamd` will use that as it's
temporary directory. Because we run Puppet from `cron` as `root`, that
`pam_tmpdir` module gets invoked each time and sets the environment
variables, which are passed over to `clamd` when Puppet calls
`/etc/init.d/clamav-daemon start`.

## Reproduction

The reason I wasn't able to reproduce the same behaviour in a local VM is
that Vagrant's provisioner for Puppet wraps the command with `sudo`, which
doesn't initiate a new "session" and the default setting of `env_reset`
prevents the environment variables being passed on from the calling session.

    dcarley@preview-asset-master:~$ id -u
    2918
    dcarley@preview-asset-master:~$ env | grep TMP
    TMPDIR=/tmp/user/2918
    TMP=/tmp/user/2918
    dcarley@preview-asset-master:~$ sudo env | grep TMP

In a similar vein, the reason that restarting the service fixed it was
because I used `service clamav-daemon restart`. Even when not using `sudo`
this wraps the init script with `exec env -i` which scrubs all environment
variables from the calling shell.

This is one reason it's preferable to use `service foo start` instead of
`/etc/init.d/foo start`. However the Puppet [service type][service-type] on
[Debian/Ubuntu][provider-deb], which inherits from a simpler
[init provider][provider-init], doesn't do this.

[service-type]: http://docs.puppetlabs.com/references/3.stable/type.html#service
[provider-deb]: https://github.com/puppetlabs/puppet/blob/3.3.2/lib/puppet/provider/service/debian.rb#L3
[provider-init]: https://github.com/puppetlabs/puppet/blob/3.3.2/lib/puppet/provider/service/init.rb#L14-15

## The fix

The simplest and most robust workaround for us was to hardset the temporary
directory in `clamd.conf(5)`. Setting this ensures that `$TMPDIR` never
takes precedence:

    TemporaryDirectory /tmp

## AppArmor

It's worth noting that we also experienced similar symptoms during the
package upgrade due to [AppArmor][apparmor]. Our old dubious-package didn't
contain any AppArmor profiles, but the Ubuntu package does out-of-the-box.
These are mostly sensible defaults and you certainly shouldn't disable them.

[apparmor]: http://en.wikipedia.org/wiki/AppArmor

The distro defaults for `clamd` are stored in
`/etc/apparmor.d/usr.sbin.clamd`. You can append to these using
`/etc/apparmor.d/local/usr.sbin.clamd`. For example:

    # Site-specific additions and overrides for usr.sbin.clamd.
    # For more details, please see /etc/apparmor.d/local/README.

    # Permit reads from local data in /srv
    /srv/** r,
