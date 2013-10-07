---
layout: post
title:  "Tetherbooting iOS with opensn0w"
date:   2013-10-07 17:00:00
categories: deprecated projects
---

[opensn0w](http://github.com/winocm/opensn0w) is a dead project originally made by me way back when in 
2011. Its original purpose was to be an open-source duplicate of redsn0w (that never came to pass). 
Recently, I took the old source tree I had, fixed it up a bit, removed some old bundles and added iOS 
7 support to it.

This thing is a generic tethered booter for all devices that use `limera1n` or `steaks4uce`. If you have
another boot ROM exploit for a newer device, feel free to send in a pull request. ;P

# Installation

Apparently, it's hard for people to install this, here's the instructions for an Ubuntu/Debian system
at least. (Please note, this does _not_ work on Mac OS X or Windows yet. Support for those will be done
eventually, or if you feel like it, implement it).

{% highlight bash %}
$ sudo apt-get install build-essential automake autoconf git libusb-1.0-0-dev libcurl4-openssl-dev \
                       libreadline-dev
$ git clone git://github.com/winocm/opensn0w.git
$ cd opensn0w
$ sh autogen.sh
$ make
$ sudo make install
{% endhighlight %}

The binaries should be in `/usr/local/opensn0w/bin` and bundles in `/usr/local/opensn0w/bundles` by 
default. It's not too hard. (Install the package equivalents for your system if you're not running 
Ubuntu/Debian respectively.)

# Usage

Usage is simple, specify the `-p` flag to specify a bundle property list. The bundle property list
should at least have Keys/IVs for each Image3 file along with a relative path. The path for things like
`AppleLogo` is mainly irrelevant, as that's cosmetic. Make sure your user can write to `/dev/usb` 
otherwise run opensn0w as root. The `-v` option is used to display additional debug output.

{% highlight bash %}
$ sudo /usr/local/opensn0w/bin/opensn0w_cli -v -p /usr/local/opensn0w/bin/iPhone3,1_7.0_11AWhatever.plist
{% endhighlight %}

Of course, modify the command line to fit your needs and make sure your device is in DFU mode.

# Adaptation

The `opensn0w.conf` file defines both kernel and iBoot patches. Another opensn0w configuration file
can be specified using the `-f` option. IPSWs can also be selected using the `-i` option. If an IPSW is 
not found, it is downloaded automatically from Apple using the URL provided in the bundle property list.

BootArgs are specified using the configuration file also.

# 'Jailbreaking'

Well, you can put files onto your device using an SSH RAM disk... the rest is up to you, this tool does
provide everything you need to bootstrap arbitrary kernels/iBoots and patch them automatically.

(Why did I have to write this?)