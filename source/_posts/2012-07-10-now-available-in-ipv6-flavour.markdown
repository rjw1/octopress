---
layout: post
title: "Now available in IPv6 flavour"
date: 2012-07-10 07:14
comments: true
categories: ops ipv6 linode
---

A friend recently reminded me that IPv6 is still a thing. Contrary to what my ISP might have me know. That got me thinking, I must check that stuff out..

This site is hosted at Linode. Proudly hosted, I might add. I've been using them for a couple of years now and only have good things to say about them. Hint: you can sign yourself up with my [referral code](http://www.linode.com/?r=26427b7455866072fef0fdfc038358b931d3d741).

Linode started rolling out IPv6 connectivity over a year ago and completed support for all geographies earlier this year. Enabling IPv6 on an existing instance is really simple. At the click of a button your primary interface will receive a single routable address from [autoconfiguration](http://en.wikipedia.org/wiki/IPv6_address#Stateless_address_autoconfiguration), which is functionally similar to DHCP but actually part of the core IPv6 specification.

I was keen to obtain a few more addresses in addition to SLAAC in order to separate out different services. Traditionally the premium of IPv4 address space has meant paying "administrative" charges for additional addresses. Because IPv6 is no longer bound by such sizing constraints most providers are happy to assign additional blocks at no extra cost, within reason.

Requesting additional addresses from Linode currently requires opening a support request. I groaned a little inside as I filled in the form at 12:05. By 12:08 I had a response with a block of new addresses that I could stand-up. Linode had once again out done themselves. They also have a helpful [knowledgebase article](https://library.linode.com/networking/ipv6) that describes how to configure the secondary addresses (not alias interfaces) for a number of different operating systems. Though do remember to configure your firewall accordingly!

Linode's administration console also supports PTRs for IPv6 addresses. So I can continue to feel like a kid on IRC:
```
07:32 -!- dcarley [dan@carley.co]
07:32 -!-  server   : calvino.freenode.net [Milan, IT]
07:32 -!-  hostname : carley.co 2a01:7e00::1b:c000 
```

Something that this has reaffirmed to me is that IPv6 will forever struggle to be taken seriously until it actually becomes available to the general populous. I'm still unable to get IPv6 connectivity in my home or office, and tunnelling is far more difficult than it needs to be. There remains a chicken-and-the-egg problem that without a critical mass of users and providers it just won't be considered a priority feature. Possibly until it is too late.
