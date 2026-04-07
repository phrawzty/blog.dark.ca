+++
title = "force disk geometry with sfdisk"
date = 2009-06-22
+++

Hello again !  This is a quick and dirty update which covers a handy little trick when dealing with writeable removable media - especially USB drives, compact flash cards, and the like.
I end up using a *lot* of USB keys in my environment for a variety of reasons, not the least of which is as handy portable Linux drives that can be stuck into any workstation and booted from directly.  They're like LiveCDs, except since i can write to them, any changes that are made during a session don't disappear when the machine reboots (nice).  As an aside, if that sounds interesting to you, i suggest checking out the [Fedora LiveCD on USB Howto](http://fedoraproject.org/wiki/FedoraLiveCD/USBHowTo).
USB keys are so ubiquitous now that we buy in bulk, meaning we'll get a bunch of identical units at one time.  Once in a while (though more often than i'd like), one of the keys will end up having a detected geometry which is different from the others.  This isn't normally a big deal, but it can cause slight variations in the apparent available space to create a partition.  This ends up being a problem if i'm looking to clone data from one key to another using a disk imaging tool such as « [Partimage](http://en.wikipedia.org/wiki/Partimage) » (another tool that gets a lot of play around here).
The solution is fantastically simple, but perhaps not immediately obvious, as it requires the use of a tool that - for the most part - never gets touched by the average user (or admin !) : « [sfdisk](http://www.google.fr/search?q=sfdisk) ».  Sfdisk is a partition table manipulator that allows us to do a number of advanced (read: *dangerous*) operations to disks.  Since the common day-to-day operations one might perform on a disk, such as creating or modifying partition assignments, are covered by the more common « [fdisk](http://en.wikipedia.org/wiki/Fdisk) » (or even « [cfdisk](http://en.wikipedia.org/wiki/Cfdisk) »), sfdisk is rarely called upon outside of bizarre or extreme situations.
Altering geometry is one such situation.

### change is good

The first thing we need to do is determine what the *correct* geometry is.  This is obtained easily enough by running an fdisk report against a known-good key (sdc, in this case) :

```bash
[root@host_166 ~]# fdisk -l /dev/sdc

Disk /dev/sdc: 4001 MB, 4001366016 bytes
19 heads, 19 sectors/track, 21648 cylinders
Units = cylinders of 361 * 512 = 184832 bytes
Disk identifier: 0xf1bcd225

 Device Boot      Start         End      Blocks   Id  System
/dev/sdc1   *           1       21648     3907454+  83  Linux
```

Alternatively, we could ask sfdisk :

```bash
[root@host_166 ~]# sfdisk -g /dev/sdc
/dev/sdc: 21648 cylinders, 19 heads, 19 sectors/track
```

Now that we have the correct geometry, we can get sfdisk to alter that of the naughty key (sdb, in this case).  As you can likely guess, -C is the cylinders, -H is the heads, and -S in the sectors (per track) :

```bash
[root@host_166 ~]# sfdisk -C 21648 -H 19 -S 19 /dev/sdb
```

Depending on your particular version of sfdisk and distro, this may trigger an interactive process which will ask you to create the desired partitions on the key.  Assuming you just want one big Linux partition, you can hit « enter » and accept every default until it's done.
And that's that - one key brought rapidly in line with the others.
Cheers !