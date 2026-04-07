+++
title = "where to specify ethtool options in Fedora"
date = 2009-08-25
+++

Hi everybody - here's a super-quick update for you concerning « [ethtool](http://directory.fsf.org/project/ethtool/) », and how to use it to set options in Fedora properly.  Ethtool is a great little tool that can be used to configure all manner of network interface related settings - notably the speed and duplex of a card - on the fly and in real time.  One of the most common situations where ethtool would be used is at boot time, especially for cards which are finnicky, or have buggy drivers, or poor software support, or.. well, you get the idea.
Times were that if you needed to use ethtool to configure a NIC setting at boot time, you'd just stick the given command line into « [rc.local](http://www.google.fr/search?q=rc.local) », or perhaps another runlevel script, and forget about it.  The problem with this approach is (at least) twofold :

* Frankly, it's easy to forget about something like this, which makes future support / debugging of network issues more of a pain.
* Anything that automatically modifies the runlevel script (such as updates to the parent package) may destroy your local edits.

In order to deal with these issues, and to standardise the implementation of the ethtool-at-boot technique, the Red Hat (and, thus, Fedora) maintainers introduced an option for defining ethtool parameters on a per-interface basis via the standard « [sysconfig](http://www.comptechdoc.org/os/linux/howlinuxworks/linux_hlsysconfig.html) » directory system.  Now, this actually happened a number of years ago, but the implementation was poorly announced (and poorly documented at the time), and thus, even today a lot of users *and administrators* don't seem to know about it.
Now, there's a very good chance that you already know this, but just to refresh your memory : in the sysconfig directory, there is another directory called « [network-scripts](http://www.faqs.org/docs/securing/chap9sec90.html) », which in turn contains a series of files named « ifcfg-eth? », where « ? » is a device number.  Each network device has a configuration file associated with it ; for example, ifcfg-eth1 is the configuration file for the « eth1 » device.
In order to specify the ethtool options for a given network interface, simply edit the associated configuration file, and add a « ETHTOOL\_OPTS » line.  For example :

```bash
ETHTOOL_OPTS="autoneg off speed 100 duplex full"
```

Now, whenever the network service initialises that interface, ethtool will be run with the specified options.  Simple, easy, and best of all, *standardised*.  What could be better ?