+++
title = "(complex) partitioning in kickstart"
date = 2009-08-03
+++

UPDATE: This article was written back in 2009. According to [a commenter below](https://www.dark.ca/2009/08/03/complex-partitioning-in-kickstart/#comment-200157), Busybox has been replaced by Bash in RHEL 6; perhaps Fedora as well?
Bonjour my geeky friends ! :)  As you are likely aware, it is now summer-time here in the northern hemisphere, and thus, i've been spending as much time *away* from the computer as possible.  That said, it's been a long time, i shouldn't have left you, without a strong beat to step to.
Now, if you're not familiar with kickstarting, it's basically just a way to automate the installation of an operating environment on a machine - think hands-free installation.  Anaconda is the OS installation tool used in Fedora, RedHat, and some other Linux OS's, and it can be used in a kickstart capacity.  For those of you looking for an intro, i heavily suggest reading over the [excellent documentation](http://fedoraproject.org/wiki/Anaconda/Kickstart) at the Fedora project website.  The kickstart configuration process could very easily be a couple of blog entries on its own (which i'll no doubt get around to in the future), but for now i want to touch on one particular aspect of it : *complex partition schemes*.

### how it is

The current method for declaring partitions is relatively powerful, in that all manner of basic partitions, LVM components, and even RAID devices can be specified - but where it fails is in the creating of the actual partitions on the disk itself.  The options that can be supplied to the partition keywords can make this clunky at best (and impossible at worst).
A basic example of a partitioning scheme that requires nothing outside of the available functions :

```bash
DEVICE                 MOUNTPOINT               SIZE
/dev/sda               (total)                  500,000 MB
/dev/sda1              /boot/                       128 MB
/dev/sda2              /                         20,000 MB
/dev/sda3              /var/log/                 20,000 MB
/dev/sda5              /home/                   400,000 MB
/dev/sda6              /opt/                     51,680 MB
/dev/sda7              swap                       8,192 MB
```

Great, no problem - we can easily define that in the kickstart :

```bash
part  /boot     --asprimary  --size=128
part  /         --asprimary  --size=20000
part  /var/log  --asprimary  --size=20000
part  /home                  --size=400000
part  /opt                   --size=51680
part  swap                   --size=8192
```

But what happens if we want to use this same kickstart on another machine (or, indeed, *many* other machines) that *don't* have the same disk size ?  One of the options that can be used with the « part » keyword is « --grow », which tells Anaconda to create as large a partition as possible.  This can be used along with « --maxsize= », which does exactly what you think it does.
Continuing with the example, we can modify the « /home » partition to be of a variable size, which should do us nicely on disks which may be smaller or larger than our original 500GB unit.

```bash
part  /home  --size=1024  --grow
```

Here we've stated that we'd like the partition to be *at least* a gig, but that it should otherwise be as large as possible given the constraints of both the other partitions, as well as the total space available on the device.  But what if you *also* want « /opt » to be variable in size ?  One way would be to grow both of them :

```bash
part  /home  --size=1024  --grow
part  /opt   --size=1024  --grow
```

Now, what do you think that will do ? If you guessed « grow both of them to *half* the total available size each », you'd be correct.  Maybe this is what you wanted - but then again, maybe it wasn't.  Of course, we could always specify a maximum ceiling on how far /opt will grow :

```bash
part  /opt  --size=1024  --maxsize=200000  --grow
```

That works, but only at the *potential expense* of /home.  Consider what would happen if this was run against a 250GB disk ; the other (static) partitions would eat up some 48GB, /opt would grow to the maximum specified size of 200GB, and /home would be left with the remaining 2GB of available space.
If we were to add more partitions into the mix, the whole thing would become an imprecise mess rather quickly.  Furthermore, we haven't even begun to look at scenarios where there may (or may not) more than one disk, nor any fun tricks like automatically setting the swap size to be same as the actual amount of RAM (for example).  For these sorts of things we need a different approach.

### the magic of pre, the power of parted

The kickstart configuration contains a section called « [%pre](http://fedoraproject.org/wiki/Anaconda/Kickstart#Chapter_4._Pre-installation_Script) », which should be familiar to anybody who's dealt with RPM packaging.  Basically, the pre section contains text which will be parsed by the shell during the installation process - in other words, you can write a shell script here.  Fairly be thee warned, however, as the shell spawned by Anaconda is « [BusyBox](http://en.wikipedia.org/wiki/BusyBox) », *not* « [bash](http://en.wikipedia.org/wiki/Bash) », and it lacks some of the functionality that you might expect.  We can use the %pre section to our advantage in many ways - including partitioning.  Instead of using the built-in functions to set up the partitions, we can do it ourselves (in a manner of speaking) using « [parted](http://en.wikipedia.org/wiki/GNU_Parted) ».
Parted is, as you might expect, a tool for editing partition data.  Generally speaking it's an interactive tool, but one of the nifty features is the « scripted mode », wherein partitioning commands can be passed to Parted on the command-line and executed immediately without further intervention.  This is *very* handy in any sort of automated scenario, including during a kickstart.
We can use Parted to lay the groundwork for the basic example above, wherein /home is dynamically sized.  Initially this will appear inefficient, since we won't be doing anything that can't be accomplished by using the existing Kickstart functionality, but it provides an excellent base from which to do more interesting things.  What follows (until otherwise noted) are text blocks that can be inserted directly into the %pre section of the kickstart config :

```bash
# clear the MBR and partition table
dd if=/dev/zero of=/dev/sda bs=512 count=1
parted -s /dev/sda mklabel msdos
```

This ensures that the disk is clean, so that we don't run into any existing partition data that might cause trouble.  The « dd » command overwrites the first bit of the disk, so that any basic partition information is destroyed, then Parted is used to create a new disk label.

```bash
TOTAL=`parted -s /dev/sda unit mb print free | grep Free | awk '{print $3}' | cut -d "M" -f1`
```

That little line gives us the total size of the disk, and assigns to a variable named « TOTAL ».  There are other ways to obtain this value, but in keeping with the spirit of using Parted to solve our problems, this works.  In this instance, « [awk](http://en.wikipedia.org/wiki/AWK) » and « [cut](http://www.gnu.org/software/coreutils/manual/html_node/cut-invocation.html) » are used to extract the string we're interested in.  Continuing on...

```bash
# calculate start points
let SWAP_START=$TOTAL-8192
let OPT_START=$SWAP_START-51680
```

Here we determine the *starting position* for the swap and /opt partitions.  Since we know the total size, we can subtract 8GB from it, and that gives us where the swap partition *starts*.  Likewise, we can calculate the starting position of /opt based on the start point of swap (and so forth, were there other partitions to calculate).

```bash
# partitions IN ORDER
parted -s /dev/sda mkpart primary ext3 0 128
parted -s /dev/sda mkpart primary ext3 128 20128
parted -s /dev/sda mkpart primary ext3 20128 40256
parted -s /dev/sda mkpart extended 40256 $TOTAL
parted -s /dev/sda mkpart logical ext3 40256 $OPT_START
parted -s /dev/sda mkpart logical ext3 $OPT_START $SWAP_START
parted -s /dev/sda mkpart logical $SWAP_START $TOTAL
```

The variables we populated above are used here in order to create the partitions on the disk.  The syntax is very simple :

* « parted -s »  : run Parted in scripted (non-interactive) mode.
* « /dev/sda » : the device (later, we'll see how to determine this dynamically).
* « mkpart » : the action to take (make partition).
* « primary | extended | logical » : the [type of partition](http://en.wikipedia.org/wiki/Disk_partitioning#PC_BIOS_partition_types).
* « ext3 » : the type of filesystem (there are a number of possible options, but ext3 is pretty standard).
  + Notice that the « extended » and « swap » definitions do not contain a filesystem type - it is not necessary.
* « start# end# » : the start and end points, expressed in MB.

Finally, we must still declare the partitions in the usual way.  Take note that this does *not* occur in the %pre section - this goes in the normal portion of the configuration for defining partitions :

```bash
part  /boot     --onpart=/dev/sda1
part  /         --onpart=/dev/sda2
part  /var/log  --onpart=/dev/sda3
part  /home     --onpart=/dev/sda5
part  /opt      --onpart=/dev/sda6
part  swap      --onpart=/dev/sda7
```

As i mentioned when we began this section, yes, this is (so far) a remarkably inefficient way to set this particular basic configuration up.  But, again to re-iterate, this exercise is about putting the groundwork in place for much more interesting applications of the technique.

### mo' drives, mo' better

Perhaps some of your machines have more than one drive, and some don't.  These sorts of things can be determined, and then reacted upon dynamically using the described technique.  Back to the %pre section :

```bash
# Determine number of drives (one or two in this case)
set $(list-harddrives)
let numd=$#/2
d1=$1
d2=$3
```

In this case, we're using a built-in function called « list-harddrives » to help us determine which drive or drives are present, and then assign their device identifiers to variables.  In other words, if you have an « sda » and an « sdb », those identifiers will be assigned to « $d1 » and « $d2 », and if you just have an sda, then $d2 will be empty.
This gives us some interesting new options ; for example, if we wanted to put /home on to the second drive, we could write up some simple logic to make that happen :

```bash
# if $d2 has a value, it's that of the second device.
if [ ! -z $d2 ]
then
  HOMEDEVICE=$d2
else
  HOMEDEVICE=$d1
fi

# snip...
part  /home  --size=1024  --ondisk=/dev/$HOMEDEVICE  --grow
```

That, of course, assumes that the other partitions are defined, and that /home is the only entity which should be grown dynamically - but you get the idea.  There's nothing stopping us from writing a normal shell script that could determine the number of drives, their total size, and where the partition start points should be based on that information.  In fact, let's examine this idea a little further.

### the size, she is dynamic !

Instead of trying to wrangle the partition sizes together with the default options, we can get as complex (or as simple) as we like with a few if statements, and some basic maths.  Thinking about our layout then, we can express something like the following quite easily :

* If there is one drive that is at least 500 GB in size, then /opt should be 200 GB, and /home should consume the rest.
* If there is one drive is less than 500 GB, but more than 250 GB, then /opt and /home should each take half.
* If there is one drive that is less than 250 GB, then /home should take two-thirds, and /opt gets the rest.

```bash
# $TOTAL from above...
if [ $TOTAL -ge 512000 ]
then
  let OPT_START=$SWAP_START-204800
elif [ $TOTAL -lt 512000 ] && [ $TOTAL -ge 256000 ]
then
  # get the dynamic space total, which is between where /var/log ends, and swap begins
  let DYN_TOTAL=$SWAP_START-40256
  let OPT_START=$DYN_TOTAL/2
elif [ $TOTAL -lt 256000 ]
then
  let DYN_TOTAL=$SWAP_START-40256
  let OPT_START=$DYN_TOTAL/3
  let OPT_START=$OPT_START+$OPT_START
fi
```

Now, instead of having to create three different kickstart files, each describing a different scenario, we've covered it with one - nice !

### other possibilities

At the end of the day, the possilibities are nearly endless, with the only restriction being that whatever you'd like to do has to be do-able in BusyBox - which, at this level, provides *a lot* great functionality.
Stay tuned for more entries related to kickstarting, PXE-based installations, and so forth, all to come here on dan's linux blog.  Cheers !