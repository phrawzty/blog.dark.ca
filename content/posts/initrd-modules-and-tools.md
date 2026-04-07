+++
title = "initrd, modules, and tools"
date = 2009-06-10
+++

Today i thought i'd discuss initrd, how modules live within it, and an give a generic example of how it can be modified on a Fedora-based system.  Along the way, i'll provide supplementary information about gzip, tar, and cpio.  Let's begin !

### initrd

The term « initrd » is short for « initial ramdisk ».  It refers to a file of the same name used by Linux systems during the boot process in order to load certain modules (drivers, and the like) prior to the full system being brought online.  An example usage of initrd would be to load a module for a RAID card, so that the actual filesystem - and all of the delicious data contained within - can be accessed properly.  As with most everything, the related [Wikipedia entry](http://en.wikipedia.org/wiki/Initrd) has more general information on the subject.

There are (perhaps unfortunately) a number of ways in which initrds can be constructed and implemented.  Generally speaking, the actual file is a compressed archive, which may contain further archives, filesystems, configuration files, and so forth.  Over the years, different distributions have handled the initrd in different ways, such that one distro's method of constructing an initrd may be totally incompatible with that of another.  In fact, even within a single distribution, different releases may alter the internals of the initrd - Fedora being a recent popular example of this.

### Fedora

In the earlier days of Fedora, the initrd was a compressed gzip archive that contained a « squashfs » file, which was itself a compressed filesystem.  If you wanted to modify the contents of the initrd, all you had to do was un-gzip the initial file, then mount the squashed filesystem within. If some or all of these terms are fuzzy for you, don't despair - i'll explain everything as we go along.  The important thing you need to know now is that in Fedora 8, the nature of the initrd changed significantly.

In order to work with the current Fedora initrd, you need to be familiar with a few terms, their related technologies, and their usage...

### ramdisk

A « ramdisk » is, as the name implies, a sort of filesystem which has been loaded into RAM.  Although it does not physically exist, it appears for all intents and purposes to be a normal disk, which has the advantage of being seamless to the user (and to the system).  Ramdisks are handy because, compared to physical disks, they are *fast*.  Within the context of the initrd, however, speed isn't a concern - it's the *versatility* of the ramdisk that is key.  As in the module example i mentioned initially, the various bits and peices of hardware in a given computer often require special drivers to make them function properly.  These drivers can be built directly into the kernel, but often, for reasons of ease of use and cross-compatibility, it's easier to put them into the initrd, and then load them via the *initial ramdisk* that gets built when the machine boots.

### modules, in general

The basic idea behind a « loadable kernel module » (or just module for short) is, again, to add support for hardware, special filesystems, and the like, which the kernel doesn't natively understand.  Modules can be loaded at any time, but for the most part, they are initialised at boot time, so that the kernel can interact with things like RAID, audio, or video cards, non-Linux filesystems (NTFS, for example), or even more esoteric items like robotic controllers and (the dreaded) software modems.

As a general rule, modules themselves are single files which are referenced via one or more configuration files.  We'll take a closer look at these ideas below.

### gzip (and tar, too !)

Gzip is, along with « tar », arguably the most ubiquitous archival format pairing in the Linux world.  Generally speaking, a gzip archive is a *single file* which has been compressed using the gzip binary.  Commonly, gzip is used to compress a « tar » archive, which normally does not use compression, but which can contain *any number* of files in a single archive.  Together, they are used to create « .tar.gz » or « .tgz » files, which are more or less the standard way to create compressed archives in Linux ; together, they're like « zip » or « rar » (or « arj », or « boo ») in the Windows world.  ([Wikipedia](http://en.wikipedia.org/wiki/Gzip))

Within the context of the Fedora initrd, however, tar is not used - from the pair, only gzip is important.

### cpio

It is a rare and glorious thing when ancient technology is updated and used constantly from day one.  Take the lightbulb, for example.  Or fire.  Or « cpio » .  It's basically like tar (mentioned above), in that it can be used to take any number of given files, and create a single archival file containing said files.  Like tar, compression isn't part of the deal, and gzip can be used to compress a cpio archive as normal.

While cpio has a long and storied history (it was first specified way back in *1977),* over the years, it has increasingly replaced by tar in many implementations.  Some time ago, Red Hat decided to use cpio (instead of tar) for their ultra-popular packaging format « RPM ».  This sparked a renewed interest in cpio in many communities, including, as you may have already guessed, the Fedora team - it's now employed regularly in the initrd (among other things).  ([Wikipedia](http://en.wikipedia.org/wiki/Cpio))

## Ok, lets go !

The first thing we'll need to do is copy the initrd to somewhere handy, like « /tmp », so that we can play with a copy .  *Never* play with the original - it should be preserved somewhere in case our modifications cause the file to become unusable.  The initrd will be named « initrd.img » - but what is it, exactly ?  Let's find out :

```bash
$ cp <original_initrd.img> /tmp/
$ cd /tmp
$ file initrd.img
initrd.img: gzip compressed data, from Unix, last modified: Fri Nov  2 15:54:12 2007, max compression
```

Ah, it's our old friend, gzip.  Since we know that gzip is a file compression tool, we know that initrd.img contains a file - let's extract it :

```bash
$ gzip -d initrd.img
gzip: initrd.img: unknown suffix -- ignored
```

Uh oh ! By default, gzip doesn't handle files with suffixes it doesn't understand (i.e. « .gz » ).  The easiest way to deal with this is simply to add the appropriate suffix, then try again :

```bash
$ mv initrd.img initrd.img.gz
$ gzip -d initrd.img.gz
```

This gives us a single file (as we expected) : initrd.img .  Though it has the same name as the file we started with, it's not the same data at all.  This initrd.img is larger than the original, and most importantly, it's a different *type* of file :

```bash
$ file initrd.img
initrd.img: ASCII cpio archive (SVR4 with no CRC)
```

Ah ha ! It's a cpio archive - the first of two we'll encounter during this process.  Note the trailing information, that it's an « SVR4 » archive, as this is important for the next step : unarchiving.

### Unpacking the initial cpio archive

As i mentioned above, cpio has a long and storied history, which is reflected in the myriad of arguments and options that can be provided during normal usage.  The command we'll be using at this stage will be :

```bash
$ cpio -i -d -H newc -F <file> --no-absolute-filenames
```

The arguments are, in order :

* -i : « copy-in » is more or less like saying « extract » in the parlance of cpio.
* -d : « make directories » allows cpio to re-create the directory structure in the archive, instead of just unpacking everything to the same place.
* -H newc : « format type » defines the particular format used by this cpio archive.  Over the course of cpio's history, numerous formats have been used for various reasons : « newc » is an SVR4-based format that allows for large files and filesystems.  Remember the archive type from above ?  This is why we needed to make note of it.
* -F <file> : « file filename » simply indicates the file we want to work with.
* --no-absolute-filenames : as the name suggests, this forces cpio to unpack the contents relative to where we executed the command.  Basically speaking, it prevents the possibility of cpio trying to write to our root filesystem, which could have disastrous results.

For more information about cpio, simply consult the [GNU cpio manual](http://www.gnu.org/software/cpio/manual/html_mono/cpio.html).

First we must make a directory in which to unpack the archive, then we can proceed :

```bash
$ mkdir /tmp/initrd
$ cd /tmp/initrd
$ cpio -i -d -H newc -F ../initrd.img --no-absolute-filenames
16442 blocks
```

The number of « blocks » refers to the amount of data that was unpacked - it is a measurement, like bytes.  This value may be different for you, as not all initrd's are created equal.

### the goods

A simple « ls » reveals the contents of the initrd.  Astute readers will notice that it looks very familiar - this is not a coincidence - it looks like the root of a normal filesystem :

```bash
$ ls -l
total 36
lrwxrwxrwx 1 dan dan    4 2009-06-10 16:37 bin -> sbin
drwxrwxr-x 2 dan dan 4096 2009-06-10 16:37 dev
drwxrwxr-x 3 dan dan 4096 2009-06-10 16:37 etc
lrwxrwxrwx 1 dan dan   10 2009-06-10 16:37 init -> /sbin/init
drwxrwxr-x 2 dan dan 4096 2009-06-10 16:37 modules
drwxrwxr-x 2 dan dan 4096 2009-06-10 16:37 proc
drwxrwxr-x 2 dan dan 4096 2009-06-10 16:37 sbin
drwxrwxr-x 2 dan dan 4096 2009-06-10 16:37 selinux
drwxrwxr-x 2 dan dan 4096 2009-06-10 16:37 sys
drwxrwxr-x 2 dan dan 4096 2009-06-10 16:37 tmp
drwxrwxr-x 6 dan dan 4096 2009-06-10 16:37 var
```

The directory we're interested in is « modules » :

```bash
$ cd modules
$ ls -l
total 5128
-rw-r--r-- 1 dan dan   12904 2009-06-10 16:37 module-info
-rw-r--r-- 1 dan dan  137235 2009-06-10 16:37 modules.alias
-rw-r--r-- 1 dan dan 4971479 2009-06-10 16:37 modules.cgz
-rw-r--r-- 1 dan dan   37966 2009-06-10 16:37 modules.dep
-rw-r--r-- 1 dan dan   60204 2009-06-10 16:37 pci.ids
```

Aah, delicious modules, at last ! But wait - what's that « .cgz » file ?

```bash
$ file modules.cgz
modules.cgz: gzip compressed data, from Unix, last modified: Fri Nov  2 15:54:07 2007, max compression
```

Another gzip - but the suffix gives us another hint.  Just as a .tgz is a gzip'd tar archive, a .cgz indicates a gzip'd cpio archive.  Of course, as before, we'll need to rename this file in order to work with it :

```bash
$ cp modules.cgz /tmp/modules.gz
$ cd /tmp
$ gzip -d modules.gz
$ file modules
modules: ASCII cpio archive (SVR4 with CRC)
```

As before, we must create a working directory into which we'll unpack the cpio archive.  Be sure to take note of the cpio arguments - there's been a change :

```bash
$ mkdir /tmp/work
$ cd /tmp/work
$ cpio -i -d -H crc -F ../modules --no-absolute-filenames
25396 blocks
```

How did i know to use a different format ? Look again at the file output for modules above, then compare it to the first cpio archive we dealt with.  The difference is subtle, but critical - the cpio manual (linked above) lists them all, just in case.  Ok, let's take a look at what we've got.

```bash
$ tree -d
.
`-- 2.6.23.1-42.fc8
    `-- i586
```

Here, the string « 2.6.23.1-42.fc8 » refers to the kernel version that the initrd is built for, and the « i586 » refers to the architecture.  Yours may be different (i randomly selected one that was hanging around on my test system), and that's OK, because the principles are still the same.  For now, you'll note that the i586 directory is full of « .ko » files - these are the module files themselves.

### adding the new module

The module that you want to add will be a .ko file as well.  You may have been supplied with this file directly, or perhaps you compiled it from source (which is outside of the scope of this document for today).  Regardless, as you may have guessed, you'll want to put it along with all the other modules :

```bash
$ cp /tmp/new_module/network_card.ko /tmp/work/2.6.23.1-42.fc8/i586/
```

With that in place, we'll re-pack cpio archive, so that it can be inserted back into the initrd.  Through the magic of re-direction, we'll accomplish this all in one swift command :

```bash
$ cd /tmp/work
$ find 2.6.23.1-42.fc8 | cpio -o -H crc | gzip > /tmp/modules.cgz
25590 blocks
```

Whoah ! Let's break that down :

* « find » was used to create a list of all the files with their relative directories.  Run « find <dir> » by itself to get a better idea of what this means.
* That list was then piped to cpio, where « -o » indicated that a new archive should be created (« copy-out » in the cpio parlance).
* That was then directly passed to gzip, which ultimately outputted the gzip'd cpio file « modules.cgz » .  Simple !

Now we can copy this shiny new modules.cgz into the initrd tree :

```bash
$ cp /tmp/modules.cgz /tmp/initrd/modules/
cp: overwrite `/tmp/initrd/modules/modules.cgz'? y
```

At this point, there's a very good chance that you'll need to modify a handful more items related to modules - in particular, the *other* files in the modules directory :

```bash
$ cd /tmp/initrd/modules
$ ls -l
total 5228
-rw-r--r-- 1 dan dan   12904 2009-06-10 16:37 module-info
-rw-r--r-- 1 dan dan  137235 2009-06-10 16:37 modules.alias
-rw-r--r-- 1 dan dan 5074769 2009-06-10 17:35 modules.cgz
-rw-r--r-- 1 dan dan   37966 2009-06-10 16:37 modules.dep
-rw-r--r-- 1 dan dan   60204 2009-06-10 16:37 pci.ids
```

There's our new modules.cgz, as you can see from the timestamp difference, along with the other items from before.  Those other items are all text files which contain various bits and pieces of information related to the modules contained in modules.cgz .  As this is a generic sort of post, i can't go into specifics about how each of these files would be modified for any *actual* module ; that said, a general overview of each of these items is most certainly in order :

* « module-info » : This file contains a an alphabetical list of each of the modules, with two lines per module that describe the type of device the module affects, and a verbose (i.e. human-readable) description of the module.  It is the most simplistic of the four, and the one most easily read by a human.  Example :

```bash
e1000
        eth
        "Intel(R) PRO/1000 Network Driver"
```

In this case, the module is named « e1000» , it affects Ethernet ( « eth » ) devices, and is an Intel gigabit NIC driver.

* « modules.alias » : Under normal usage, hardware will expose certain information to the kernel that helps the kernel to properly identify and assign said hardware.  This is done via a sort of identification string, which contains (in machine-readable format) such things as the bus used by the hardware, the vendor and device id's, and other such sundry items.  Normally, the system takes care of populating this file on its own, but in the case of the initrd, this doesn't happen.  As a result, it is highly likely that you'll need to insert the identification string for your hardware manually ; hopefully, your module came with some documentation to help with this.
  + If your module didn't come with the appropriate documentation, you could always try putting the hardware into an already-installed system, and then poking around for it manually.  More information is available from the [ArchLinux wiki](http://wiki.archlinux.org/index.php/ModaliasPrimer).
  + This file, since it's normally generated automatically, and used behind the scenes, is the most daunting of the four.  Once you get a handle on how the string is built, however, it stops being scary, and starts being merely irritating.  I'm not going to lie - this file is tough.  Google is your friend on this one !
  + Example :

```bash
alias pci:v00008086d0000108Bsv*sd*bc*sc*i* e1000
```

* « modules.dep » : This lists the « dependencies » for each module - effectively, it's a list of what modules the *other* modules depend on.  Sometimes modules are designed to operate completely independently (such as the above-noted e1000 module), while other times they build on each other, with each module providing certain functionalities used by the next.  This file builds those relationships.  Example :

```bash
msdos: fat
```

* « pci.ids » : Finally, can you guess what's listed in this file ?  If you guessed « PCI IDs », you're right !  This file is in the same genre as modules.alias, in that it contains a list of hardware identifiers that are used by the kernel in order to properly assign drivers and so forth.  Again, as with modules.alias, this data is not necessarily obvious, and your hardware documentation should list the appropriate identifiers.
  + In the event that your documentation doesn't have this information, there are two excellent sites that contain information on known PCI IDs : the [PCI ID Repository](http://pciids.sourceforge.net/), and the [PCI Vendor and Device List](http://www.pcidatabase.com/index.php).  Not exactly bedtime reading, i know, but they're both life-savers !
  + Example :

```bash
101e  American Megatrends Inc.
        1960  MegaRAID
        9010  MegaRAID 428 Ultra RAID Controller
        9060  MegaRAID 434 Ultra GT RAID Controller
```

Once these files have been properly updated, the only step that remains is re-packaging everything into a proper initrd.img !

### new initrd.img

Here in the final step we'll leverage the power of re-direction and create our initrd.img in one fell swoop :

```bash
$ cd /tmp/initrd
$ find . | cpio -o -H newc | gzip > /tmp/initrd.img
$ file /tmp/initrd.img
/tmp/initrd.img: gzip compressed data, from Unix, last modified: Wed Jun 10 19:39:51 2009
```

And there you have it, a shiny new initrd.img, ready for use in your PXE bootstrap, disk image, or wherever else such things are needed.
Aren't sure what a PXE bootstrap is ?  Not sure why you'd want to use a disk image ?  Stay tuned, fair reader, all will be revealed as we delve deeping into the mysterious land of the Linux System Administrator...