---
layout: post
title: "CentOS AMIs from Kickstart"
date: 2012-04-17 07:53
comments: true
categories: ops dev aws ec2 centos
---

I've been doing some work on AWS EC2 again recently. Since there aren't any "official" CentOS AMIs and I'm dubious about using some of the public images, we had to build our own. Previously we have rolled these using instructions like [this](http://bodgitandscarper.co.uk/amazon-ec2/building-centos-5-images-for-ec2/) and [this](http://wiki.sysconfig.org.uk/display/howto/Build+your+own+Core+CentOS+5.x+AMI+for+Amazon+EC2). They worked okay but it sure was long-winded process. So much so, that I struggled to pick it up where I last left off.

## Enter Kickstart

I couldn't help thinking that there was a better way. We build all internal machines using [Kickstart](http://fedoraproject.org/wiki/Anaconda/Kickstart). The format is simple and well understood, so why can't we use that on the cloud?

Apparently I wasn't alone in thinking this. Jeremy Katz experienced the same frustrations and wrote [ami-creator](http://velohacker.com/2010/12/13/announcing-ami-creator/). It's a light shim over python-imgcreate which comes from Fedora's [livecd-tools](http://fedoraproject.org/wiki/FedoraLiveCD) package. It takes a pretty conventional kickstart file, with a few minor variations, and spits out an image file. This image can then be uploaded to EC2 as an AMI.

The requirements for our image were as follows:

- CentOS 6
- 64-bit
- EBS backed
- Very minimal package set
- Puppet pre-installed
- SELinux in enforcing mode

Jeremy's original use was intended for CentOS 5 as S3 backed instance-store images. This required some code changes. Most of which have been submitted upstream, but are all available in [this branch](https://github.com/dcarley/ami-creator/tree/ebs).

My resulting kickstart file and some SELinux specific changes are described further below. You can find the complete [kickstart file on Github](https://github.com/dcarley/ami-creator/blob/dcarley/ks-centos6.cfg).

## Create master with Fog

I'll be using [Fog](https://github.com/fog/fog) to do all of the lifting on AWS. It's a Ruby library that programmatically exposes all of AWS's functionality, which suits both automation and documentation perfectly. I've personally found it to be so effective that I rarely feel the need to touch the AWS website.

My only criticism of Fog would be that the usage documentation is a little lacking or confusing in places. It took me a little while to piece together the right syntax for some of the calls. Though to be fair, some of those problems are just manifestations of API oddities at Amazon. I hope this article will supplement Google results for the other helpful examples I found.

To kick off we need to give Fog our AWS credentials. You can use the master credentials found on the AWS account page or create a more restrictive account with IAM.

``` yaml ~/.fog
:default:
    :aws_access_key_id:     YOUR_KEY
    :aws_secret_access_key: YOUR_SECRET
```

Next we define some variables that will remain constant throughout. AMIs are tied to AWS regions, so if you want to use your resulting AMI in more than one region then you'll have to build those separately or block-transfer the images over by some means. It is assumed that you already have an SSH keypair setup on EC2.

``` ruby
@region     = "eu-west-1"
@key_name   = "dcarley_keypair"
@ami_name   = "centos62-x86_64"

require 'fog'
fog = Fog::Compute.new(
    :provider => "AWS",
    :region => @region
)
```

EBS volumes and the instances they can attach to are tied to availability zones, so we need to perform our initial build all within the same zone. This won't affect what zone you use the resulting AMI in. Pick a zone at random.

``` ruby
@av_zone = fog.describe_availability_zones.body["availabilityZoneInfo"].sample["zoneName"]
```

The new image will be built on an EC2 server. This suits because we have direct and fast access to the EBS volume that the image needs to be transferred to. We also have a wide range of existing OS images to build from. We'll use Fedora 16 to build our image because it has a recent copy of python-imgcreate. Happily the Fedora Project have published some [official AMIs](https://fedoraproject.org/wiki/Cloud_images) that we can use. We'll use an `m1.small` instance instead of `t1.micro` because it dramatically reduces the build time.

``` ruby
f16ami = fog.images.all("image-id" => "ami-2df4c959").first

master = fog.servers.create(
    :tags => {"Name" => "amimaster"},
    :flavor_id => "m1.small",
    :image_id => f16ami.id
    :key_name => @key_name,
    :availability_zone => @av_zone
)

master.wait_for { ready? }
master.reload.dns_name
```

SSH into the public hostname of your new instance with the username `ec2-user` and the SSH key previously specified.

Install some dependencies that we're going to use during the build process.

    sudo yum install \
        git                         # To fetch ami-creator  \
        python-imgcreate            # To build our image \
        compat-db45                 # To rebuild CentOS 5 RPM DBs \
        compat-db47                 # To rebuild CentOS 6 RPM DBs

We need to break two cardinal rules while building the image. Firstly the whole process must be run as root due to the various mounts and chroots. Secondly we have to flip SELinux into `permissive` mode due to some [problems with livecd-tools](https://bugzilla.redhat.com/show_bug.cgi?id=735598).

    sudo -i
    setenforce 0

Grab a copy of ami-creator. Presently from my own branch.

    git clone -b dcarley git://github.com/dcarley/ami-creator.git

## Create image

Now we start to define our kickstart config. This part looks pretty standard. DHCP is enabled because that's how EC2 does IP allocation. SELinux is set to enforcing mode.

The root password is set to an unusable hash to prevent remote root logins. It is important to remember that not setting a `rootpw` option here results in a passwordless account that can be logged in or elevated to by anyone.

``` sh ks-centos6.cfg
keyboard uk
lang en_GB.UTF-8
timezone UTC

network --device eth0 --bootproto dhcp
firewall --enabled --port ssh
selinux --enforcing
auth --useshadow  --enablemd5
rootpw --iscrypted *
```

Since PV-GRUB takes care of most bootloading and we only want a single partition we can comment out the traditional options here.

At this point, if you are creating an EBS AMI, then the partition size only needs to be big enough to take the initial installation. We'll set the real size of the AMI's partition later on.

``` sh ks-centos6.cfg
bootloader --timeout=0
#clearpart --all --initlabel
#zerombr
#autopart

part / --size 1024 --fstype ext4
```

A standard set of Yum repositories are defined. By including the updates repository we can ensure the image contains any security updates available at the time of building. For this reason it might be useful to recreate your image every now and then.

``` sh ks-centos6.cfg
repo --name=CentOS6-Base --mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=os
repo --name=CentOS6-Updates --mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=updates
repo --name=CentOS6-Addons --mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=extras
repo --name=EPEL --baseurl=http://download.fedoraproject.org/pub/epel/6/$basearch/
```

Because the installation is performed on the Fedora host outside of a chroot we can include additional repos from local file paths if you wish to inject additional packages.

``` sh ks-centos6.cfg
repo --name=Local --baseurl=file:///root/local_repo/
```

The package sets are pretty minimal. We avoid pulling in anything more than is required for the OS to boot and network. Some packages are explicitly defined here because we'll use their binaries during the build process, but they could be safely omitted. We also need some packages specifically for use on EC2.

``` sh ks-centos6.cfg
%packages --nobase
@core
audit
selinux-policy-targeted
system-config-securitylevel-tui

coreutils
e2fsprogs
passwd
policycoreutils
chkconfig
rootfiles
openssh-server
openssh-clients

# EC2ify
grub
kernel-xen
dhclient
iputils
curl
sudo
%end
```

I'm not a fan of scripting `%post` content. However there are a few cleanup operations that we need to perform once the packages are installed.

Because our Fedora 16 host uses a newer version of RPM than both CentOS 5 and 6, we need to perform a little fix-up of the guest's Berkley DBs. This is done by dumping them into a flatfile format with BDB 4.8 and re-importing them using BDB 4.7. The `--nochroot` flag is used to perform the operation on the host instead of the guest.

If you were building a CentOS 5 image then you would use `db45_load` instead which is just-about backwards compatible with BDB 4.3

``` sh ks-centos6.cfg
%post --erroronfail --nochroot
rm -f ${INSTALL_ROOT}/var/lib/rpm/__db*
for RPMDB in ${INSTALL_ROOT}/var/lib/rpm/*; do
    db_dump $RPMDB | db47_load ${RPMDB}.47 && mv ${RPMDB}.47 $RPMDB
done
%end
```

RPM 4.9 in Fedora 16 also uses Btrees for secondary indexes which is not backwards compatible with older versions of RPM. According to the [release notes](http://www.rpm.org/wiki/Releases/4.9.0#RPMdatabaseconfigurationandformat) the database format remains unchanged and the indexes can be rebuilt with older versions of RPM.

On CentOS 6 this generates an error about the `Name` database but this doesn't appear to cause any problems in the final image.

``` sh ks-centos6.cfg
%post --erroronfail
rpm --rebuilddb
%end
```

If you wanted to include any scripts to setup an additional user, import SSH keys from EC2's meta-data service, configure sudo, configure SSH, or install security updates, then you could do so here. I'm going to omit this process and explain in a separate article about how I handle these.

Lastly we need to relabel the guest's filesystem to ensure that SELinux will work. This should normally be handled by python-imgcreate itself but there appear to be [some issues](https://bugzilla.redhat.com/show_bug.cgi?id=773339) in the way that it works, so we have to:

- Temporarily hide any `selinuxfs` bind mounts from the host to ensure that we use the guest's file context definitions
- Remove the bind mount for host's Yum cache directory to ensure that it is labelled correctly and then replaced with a fake mount to satisfy python-imgcreate's umount operations.
- Label the guest filesystem correctly using both `file_contexts` and `file_contexts.homedirs`

This must be performed as the very last `%post` script in your kickstart file. If not, you may create files that end up being unlabelled.

``` sh ks-centos6.cfg
%post --erroronfail
mount -t tmpfs -o size=1 tmpfs /sys/fs/selinux
mount -t tmpfs -o size=1 tmpfs /selinux
umount /var/cache/yum

/sbin/setfiles -F -e /proc -e /sys -e /dev -e /selinux /etc/selinux/targeted/contexts/files/file_contexts /
/sbin/setfiles -F /etc/selinux/targeted/contexts/files/file_contexts.homedirs /home/ /root/

umount -t tmpfs /sys/fs/selinux /selinux
mount -t tmpfs -o size=1 tmpfs /var/cache/yum
%end
```

Build that image!

    ./ami-creator/ami-creator -n centos62-x86_64 -c ks-centos6.cfg --cache yum_cache_62

This will pop out a `.img` file. You can inspect the contents, if you so wish.

    mount -o loop,ro centos62-x86_64.img /mnt

## Transfer image to EBS

Now we can set about moving that image file into something that resembles a VM.

Create a new EBS volume. We go with 10G, which is a reasonable balance between storing enough log files and moving more persistent data out to separate EBS volumes.

``` ruby
vol = fog.volumes.create(
    :size => 10,
    :tags => {"Name" => @ami_name},
    :availability_zone => @av_zone
)
vol.wait_for { ready? }
```

Attach the new volume to our Fedora instance. The block device name of `sdi` is somewhat arbitrary, so long as it's higher than any existing EBS and ephemeral mappings. It will actually appear as `xvdi` due to Xen device renaming in more recent kernels.

``` ruby
fog.attach_volume(master.id, vol.id, "/dev/sdi")
vol.wait_for { ready? }
```

Create a single bootable Linux partition that spans the entire EBS volume.

    sfdisk /dev/xvdi << EOF
    0,,83,*
    ;
    ;
    ;
    EOF

Block transfer the image into the new partition. Although the image is sparse up to 1G specified in the kickstart's `part` option those zeros still take just as long to copy. Hence the reason for making the image as small as possible.

    time dd if=centos62-x86_64.img of=/dev/xvdi1 bs=8M

Mark the filesystem as clean and then extend it to span the entire 10G partition. This saves ~90% of the transfer time and forgoes precisely matching up the partition sizes.

    e2fsck -f /dev/xvdi1
    resize2fs /dev/xvdi1

I'm pretty sure that block devices don't go through the kernel's buffer cache and that `sync(1)` doesn't handle these either. However it makes me feel ever so slightly safer that our kernel isn't holding onto anything. This can be omitted if you know better.

    sync

## Register the new AMI

To provide some form of versioning for our instance we append the current datetime to the name. You might want to write a longer description of any package versions that have gone into your image.

``` ruby
require 'date'
@ami_desc = @ami_name + "-" + DateTime.now.strftime("%Y%m%d%H%M")
```

Registering an AMI requires that you provide a snapshot of an EBS volume, rather than the volume itself. This is presumably for the sake of consistency. Snapshot creation can take a little while to complete.

``` ruby
snap = fog.snapshots.create(
    :name => @ami_name,
    :description => @ami_desc,
    :volume_id => vol.id)
snap.wait_for { ready? }
```

Create a conventional looking block mapping for our root device and any ephemeral volumes that may be available. We specify that a root volume should be destroyed at the same time an instance is terminated, to prevent clutter. This mapping can be overridden for individual instances when you bring them up. You can learn more about the format from the EC2 documentation for [Block Device Mapping Concepts](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/block-device-mapping-concepts.html) and [RegisterImage](http://docs.amazonwebservices.com/AWSEC2/latest/APIReference/ApiReference-query-RegisterImage.html).

``` ruby
block_map = [
    {"DeviceName" => "/dev/sda",
        "SnapshotId" => snap.id,
        "VolumeSize" => snap.volume_size,
        "DeleteOnTermination" => true},
    {"DeviceName" => "/dev/sdb", "VirtualName" => "ephemeral0"},
    {"DeviceName" => "/dev/sdc", "VirtualName" => "ephemeral1"}
]
```

Find the most recent pv-grub kernel for use with EBS images. We want `hd00` rather than `hd0`.

``` ruby
aki = fog.images.all(
    "Owner" => "amazon",
    "image-type" => "kernel",
    "architecture" => "x86_64",
    "manifest-location" => "ec2-public-images-eu/pv-grub-hd00_*"
).last
```

Register the new AMI.

``` ruby
ami = fog.register_image(
    @ami_name,
    @ami_desc,
    "/dev/sda1",
    block_map,
    {
        "KernelId" => aki.id,
        "Architecture" => "x86_64"
    }
)
```

Unfortunately `Fog#register_image` gives us an `Excon::Response` object instead of `Fog::Compute::AWS::Image`. So we have to perform an extra step to lookup the real object.

``` ruby
ami = fog.images.all("image-id" => ami.body["imageId"]).first
ami.wait_for { ready? }
```

## Take it for a spin

Bring up a new instance using our AMI.

``` ruby
server = fog.servers.create(
    :tags => {"Name" => @ami_desc},
    :image_id => ami.id,
    :flavor_id => "t1.micro",
    :key_name => @key_name,
    :availability_zone => @av_zone
)
server.wait_for { ready? }
server.reload
```

## Debugging

Debugging AMIs can be a tedious process. A lot of useful information can often be gleamed from the "Get System Log" button in AWS console which yields a read only view of your image's console after boot. Although it sometimes takes a while to populate after an instance has booted.

Typically it's always one, or both, of the following..

### Device naming

The key things to remember are that EBS images require a AKI labelled `pv-grub-hd00` and a Grub menu entry with `root (hd0,0)`. Whereas S3 instance-store images require a `pv-grub-hd0` AKI and `root (hd0)` Grub entry.

Beware that some newer kernels, like anything since EL5, will present block devices of `/dev/xvdN` instead of `/dev/sdN`. The best way to work around this it to use filesystem labels in your Grub and `fstab` entries.

The quickest way to troubleshoot these kind of problems is to mount the EBS volume on the Fedora instance, make some changes by hand, repeat the snapshot process and try booting. When you're happy that it boots okay you can repeat the image creation correctly.

### Post boot scripts

Boot scripts that are specific to EC2 initialisation are probably the most frustrating to troubleshoot. Especially the transfer of SSH public keys, since they are integral to logging into the machine for investigation. Often the only sane route is to temporarily set a root password and enable remote root logins to your image. Log into a freshly booted instance and check the logs. Tweak your code and try rebooting a few times.

Treat it like installing new hardware in a PC. Let it run a few times before you close the lid. Because if you confidently screw the lid on first time then it's sure to fail POST.
