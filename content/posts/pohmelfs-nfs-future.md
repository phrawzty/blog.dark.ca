+++
title = "pohmelfs - the network filesystem of the future !"
date = 2009-06-17
+++

Hello again fair readers !  Today we're going to take a look at [POHMELFS](http://www.ioremap.net/projects/pohmelfs), which is a network file system that was just recently integrated into the Linux kernel.  This is an excellent exercise for three reasons : we'll learn about some great new concepts related to network file systems, we get to compile and install our own kernel, and we get to play with a great new platform that could, eventually, become the new standard for network file systems in the \*nix world.  Sound good ?  Let's go !
A network file system is, according to [Wikipedia](http://en.wikipedia.org/wiki/Network_file_system), « any computer file system that supports sharing of files, printers and other resources as persistent storage over a computer network. » It is important to note that there is also a specific protocol called « Network File System », which is exactly what it sounds like, and is highly popular in the \*nix world.  For the purposes of this article, unless otherwise stated every time i use « network file system » or « nfs », i'm referring to the concept, not the protocol.
Building on the idea of an nfs, there are other types of network-aware file systems which fall into similar categories, such as « [distributed file systems](http://en.wikipedia.org/wiki/Distributed_file_system) », « parallel file systems », « distributed parallel file systems », and so forth.  These terms are largely non-standard, and conventional wisdom tends to put all of these sub-categories into the one big nfs family.  POHMELFS, for example, is described by its creator as a « distributed parallel internet filesystem » - in fact, the name itself is an acronym for « Parallel Optimized Host Message Exchange Layered File System », which is, mercifully, pronounced simply as « poh-mel ».
The two interesting portions of the name are « parallel » and «optmized »...

### parallel

One of the interesting aspects of POHMELFS is that a given chunk of data can (and in many cases, *should*) reside in more than one distinct storage unit (different physical servers, for example).  The client can be made aware of this, which means that accesses to the data can happen in a « parallel » fashion, which has two major advantages :

* Seamless real-time replication of the data during write operations
* Faster overall data accesses during read operations

Imagine a scenario whereby you have two file servers : serverA and serverB, and any given number of clients (clientA, B, C, etc..).  In a classic non-parallel (and non-replicating) file system scenario, data would not be identical between the two file servers.  Clients would need to know which server had the data they wanted ahead of time, and if one of the servers went down, it would take all of its unique data with it.  Furthermore, if all of the clients wanted to read data from serverB at once, everybody would get bogged down overall, since there are a finite amount of resources available for everybody to use.
The same scenario with a parallel file system is much better indeed !  Firstly, data would be identical on the two file servers, meaning that if one were to go down, there would be no direct loss of data availability (already a *huge* improvement).  Secondly, and this is also huge, clients could spread their requests for data between the two servers, thus reducing the direct load on any one given server, and resulting in better overall resource usage (translation : better performance).

### optimized

The author claims that POHMELFS is designed from the ground-up with performance in mind.  This design principle has resulted in some amazing benchmarks which, frankly stated, make more or less every other network file system [look slow](http://www.ioremap.net/node/153) in [comparison](http://www.ioremap.net/node/134) ; some types of data access processes are merely rapid, whereas others are revolutionary.  Of course, speed isn't everything, and as you might expect, POHMELFS isn't quite as feature rich (yet?) as many of the other nfs options on the market.

## take it for a test drive

At the top of the article i mentioned that POHMELFS was recently introduced into the Linux kernel.  What i really meant was that as of kernel version 2.6.30, POHMELFS is located in the « [staging](http://kerneltrap.org/Linux/Introducing_the_Linux_Staging_Tree) » tree of the overall [source](http://en.wikipedia.org/wiki/Source_tree) collection.  This is already a little bit of a warning - code in the staging area is generally considered usable, but not ready to be merged into the kernel proper.  If this doesn't sound like your bag, it's time to bail out now, but for those of you in the mood for adventure, the staging tree is a great place to find interesting new features and functionalities.
In the test scenario we're about to run through, there are two servers : « host\_75 » and « host \_166 ».  Both of these machines are *very* basic installations of Fedora 8, which from a server perspective is not very different from any other Fedora prior or since - therefore, unless otherwise noted, these operations should be identical on any other Fedora box.

### kernel configuration

Long ago, in a faraway land where dragons battled wizards for supremacy over ancient battlements, the process of properly configuring, compiling, and installing a new Linux kernel was arcane knowledge - the stuff of legends.  In our modern, enlightened age, installing a new kernel on your machine is (relatively) simple !  Heck, you even get a *menu* these days...
The first step in compiling a new kernel is to make sure that your system has the necessary tools with which to work.  On a standard Fedora system, the fastest way is simply to install all of the development tools as a single mass operation.  This is overkill, really, but it's simple, and since we're just testing things out anyways, fast and simple are our primary criteria :

```bash
[root@host_75 ~]# yum groupinstall 'Development Tools'
```

The kernel menu i mentioned a few moments ago is going to require an additional package which is not part of the development tools group : « ncurses-devel ».

```bash
[root@host_75 ~]# yum install ncurses-devel
```

Next up, we need to download and unpack the kernel source package.  Your best bet is from [kernel.org](http://www.kernel.org/), which is reliable, rapid, and most importantly, secure :

```bash
[root@host_75 ~]# cd /usr/src
[root@host_75 src]# wget http://eu.kernel.org/pub/linux/kernel/v2.6/linux-2.6.30.tar.bz2
[root@host_75 src]# tar -xvjf linux-2.6.30.tar.bz2
[root@host_75 src]# ln -s linux-2.6.30 linux
```

Finally, we can take a look at the configuration menu.  You'll notice that we issue the command « make », which is a mechanism for compiling source code used across the \*nix world.  We'll see it again a little later on, but for now, understand that all we're doing here is «making » the configuration menu, not the kernel itself.

```bash
[root@host_75 src]# cd linux
[root@host_75 linux]# make menuconfig
```

Now there are a lot (*a LOT*) of possible options for configuring a kernel, and if you've got the time and the inclination, going through each one and reading the description can be very enlightening.  That said, what we're interested in is POHMELFS, and in order to enable it, we're going to have to explicitly tell the configuration that we're interested in staging-level code.
First, *enable* « Staging Drivers » :

```bash
Device Drivers ---> [*] Staging Drivers
```

Then *disable* « Exclude Staging drivers from being built ».  It's set up this way in order to prevent somebody from accidentally building anything from staging by accident :

```bash
Device Drivers ---> Staging Drivers ---> [ ] Exclude Staging drivers from being built
```

Next, enable POHMELFS as a *module* (if you'd like a refresher on modules, just check out any post on this blog with the « modules » tag) :

```bash
Device Drivers ---> Staging Drivers ---> <M> POHMELFS filesystem support
```

And, optionally, support for encryption :

```bash
Device Drivers ---> Staging Drivers ---> <M> POHMELFS filesystem support > [*] POHMELFS crypto support
```

Finally, you may wish to add a « local version string » - this is an identifier that you can customise to help you keep track of each kernel build.

```bash
General Setup ---> (-pohmelfs_test) Local version
```

Now we save and exit !

### build & install the kernel

From here, all that's left is to let it build - depending on your hardware, this can take a while.  Patience, grasshopper.

```bash
[root@host_75 linux]# make && make modules && make modules_install && make install
```

There are four distinct commands which, assuming the previous one is successful, will execute consecutively - that's what the double-ampersand (&&) does.

* make : Builds the actual kernel (this is the part that takes forever)
* make modules : Builds the modules (everything enabled as <M>, such as POHMELFS)
* make modules\_install : Copies the modules to their proper positions
* make install : Creates the initrd (which i discussed in a previous post), sets up the bootloader (which we'll take a look at), and so forth

Once the process is done, which is to say that all four items executed successfully, the last thing we need to check before we reboot is the bootloader - in this case, « [GRUB](http://en.wikipedia.org/wiki/GNU_GRUB) ».  The « make install » will add an entry for our new kernel into the GRUB configuration.  This is fairly automatic, but i like to check it, just to be safe.  The new entry should look something like this :

```bash
title Fedora (2.6.30-pohmelfs_test)
 root (hd0,0)
 kernel /vmlinuz-2.6.30-pohmelfs_test ro root=/dev/sda1
 initrd /initrd-2.6.30-pohmelfs_test.img
```

From here, we reboot, and when the GRUB menu appears, select our new POHMELFS item instead of the default entry.

### userspace tools

The actual POHMELFS executables - the « userspace tools », so called since they are used by the user, not by the system, are *not* included with the kernel.  This is normal.  Even though our kernel now supports POHMELFS, our system doesn't actually have any of the software which will interface with the kernel module yet.  This has to be downloaded, configured, and compiled in the same fashion as the kernel ; however, whereas the kernel was easily downloaded via HTTP, the POHMELFS source is only available via « [GIT](http://en.wikipedia.org/wiki/Git_(software)) ».
GIT, in a nutshell, is a platform for managing source code (like « [CVS](http://en.wikipedia.org/wiki/Concurrent_Versions_System) », « [Subversion](http://en.wikipedia.org/wiki/Subversion_(software)) », or « [VSS](http://en.wikipedia.org/wiki/Visual_SourceSafe) », just to name a few).  For our purposes today, we're only going to use *one* of its many features : copying the source code from the official POHMELFS site so that we can compile it ourselves.  If you don't already have GIT installed, now is the time !  Don't delay, act today !

```bash
[root@host_75 ~]# yum install git
```

Depending on your existing installation, this may cause a fairly large number of new packages to be downloaded and installed, so don't worry if the list looks huge.  With that out of the way, we can download the source - this is known as « cloning » a « project » in GIT terminology :

```bash
[root@host_75 ~]# git clone http://www.ioremap.net/git/pohmelfs-server.git
```

Preparation of the source and so forth for POHMELFS was, not too long ago, a bit of a pain in the yoohoo.  Now, the author has graciously included a tool which will take most of the pain out of the process - though there are still a few things to look out for :

```bash
[root@host_75 ~]# cd pohmelfs-server
[root@host_75 pohmelfs-server]# ./autogen.sh
```

This will take a little while, and will output a handful of lines as it goes along.  The next step is to is a standard « ./configure », which, if you've never compiled anything before, is just about the most standard possible way to pre-configure source for compilation in the \*nix world.  Normally, ./configure accepts a variety of options (take a look at « ./configure --help » for a taste), but for now, we're only interested in one :

```bash
[root@host_75 pohmelfs-server]# ./configure --with-kdir-path=/usr/src/linux/drivers/staging/pohmelfs
```

The supplied option tells ./configure where the POHMELFS kernel code is - in specific, where the « netfs.h » file is located.  This is important later on, so take note.  This process output tonnes of lines as it checks for the capabilities and desires of your environment, and customises the configuration as best it can for your machine.  Once it's done, we can go ahead and « make » :

```bash
[root@host_75 pohmelfs-server]# make
```

As of this writing, the above make may fail.  As it turns out, certain elements in the POHMELFS source, as they stand now, expect [OpenSSL](http://en.wikipedia.org/wiki/Openssl) to be installed on the system.  This is true even if you were clever and provided « --disable-openssl » to ./configure above (good thinking, though !).  We've got two options here : either modify the POHMELFS source in order to remove the references to things which do not exist, or simply install OpenSSL and be done with it.  If you've already got OpenSSL on your system, then no worries, you probably didn't even see this problem.
OpenSSL, briefly, is an open-source implementation of a series of encryption protocols and cryptographic algorithms which, among other things, allow for « secure » websites (i.e. via HTTP**S**), and other such things.  As such, it's a pretty standard thing to have on a machine (especially a server), and since it's so easy to install in our test scenario, we'll just go ahead and do that now :

```bash
[root@host_75 pohmelfs-server]# yum install openssl openssl-devel
```

Back to POHMELFS, it's time to reconfigure.  We'll enable openssl now, since we've got it anyways...

```bash
[root@host_75 pohmelfs-server]# ./configure --with-kdir-path=/usr/src/linux/drivers/staging/pohmelfs --enable-openssl
[root@host_75 pohmelfs-server]# make
```

As of this writing, the above make *will* fail (again, possibly).  In this instance, a required file can't be located : netfs.h .  Remember that one from above ?  Of course you do.  As it turns out, even though we explicitly specified the path to find this file, certain elements in the source (possibly auto-generated) expect it to be elsewhere.  As with OpenSSL, we have two options : alter the code ourselves, or just satisfy the requirement as painlessly as possible.  Well, you already know how we roll in these parts :

```bash
[root@host_75 pohmelfs-server]# mkdir /usr/src/linux/drivers/staging/pohmelfs/fs
[root@host_75 pohmelfs-server]# ln -s /usr/src/linux/drivers/staging/pohmelfs /usr/src/linux/drivers/staging/pohmelfs/fs/pohmelfs
```

What we've done here is create a link that points from where the source *wants* netfs.h to be, to where it actually is.  Hey, it's staging-level code, remember ?  No worries - this was an easy one anyways.  With that out of the way, we can make away !

```bash
[root@host_75 pohmelfs-server]# make
   ...
[root@host_75 pohmelfs-server]# make install
```

The make install will put the necessary binaries in the appropriate places on the system.  In particular :

```bash
[root@host_75 ~]# which fserver cfg
/usr/local/bin/fserver
/usr/local/bin/cfg
```

### and again !

That's one server down, one to go.  But, wait, that was a lot of work, and even more waiting, wasn't it ?  Doing it again would suck ; luckily, there are some shortcuts we can take.
If the hardware of both machines is more or less the same, there's a better-than-average chance that the same kernel you've already compiled will work on the other server - you can just port it over.  Now, there are very particular, very clean ways to go about this, and to those that like their test environments clean and tidy, i salute you ; we, however, know better.  Let's just pack up the source, copy it over, and deploy it all at once :

```bash
[root@host_75 ~] cd /usr/src/
[root@host_75 src] tar -cvzf src.tar.gz linux
   ...
[root@host_75 src] scp src.tar.gz root@host_166:/usr/src/
   ...
```

From here on in you'll want to keep an eye on the hostname being used - we're dealing with two machines now...

```bash
[root@host_166 ~] cd /usr/src/
[root@host_166 src] tar -xvzf src.tar.gz
[root@host_166 src] cd linux
[root@host_166 src] make modules_install && make install
```

Nice !  Reboot the second machine now, and *don't forget* to choose the new kernel from the GRUB menu.
Likewise, we don't need to install GIT on the second machine - we'll just do like we did with the kernel :

```bash
[root@host_75 ~]# tar -cvzf poh.tar.gz --exclude=.git pohmelfs-server/
[root@host_75 ~]# scp poh.tar.gz root@host_166:~/
```

Notice the « --exclude » line in the tar command ?  This is to prevent the GIT-specific stuff (which is substantial) from being archived, as it is not useful where we're going.

```bash
[root@host_166 ~]# tar -xvzf poh.tar.gz
   ...
[root@host_166 ~]# cd pohmelfs-server/
[root@host_166 pohmelfs-server]# make install
```

### testing time

The first thing we need to do is load the POHMELFS module, which was generated way back when we built the kernel :

```bash
[root@host_75 ~]# modprobe pohmelfs
[root@host_75 pohmelfs-server]# lsmod
Module                  Size  Used by
pohmelfs               59284  0
```

You will likely have a lot more items in this list - but one of them *must* be « pohmelfs ».  Before we start the server daemon, we'll have to decide which directory we want to « export », which is to say which one we'd like to share on the network.  For now, let's pick « /tmp », since it's simple (and it's the daemon default).  Let's put a file in there so that we can check to see if our share is properly working :

```bash
[root@host_75 ~]# touch /tmp/POHTEST.TXT
```

Next, we start the server daemon.  For our first test, we'll just launch the binary without any options - by default, the daemon will launch in a local console, export /tmp (as noted above), and bind the process to port 1025.  Eventually you may wish to change some or all of these defaults, but for now, we'll keep it simple :

```bash
[root@host_75 ~]# fserver
Server is now listening at 0.0.0.0:1025.
```

The most basic test possible at this point is to telnet from the second machine to the first, just to see if we can connect on the port :

```bash
[root@host_166 ~]# telnet host_75 1025
Trying 192.168.0.75...
Connected to 192.168.0.75.
Escape character is '^]'.
^]
telnet> quit
Connection closed.
```

This will create some output on the server console :

```bash
Accepted client 192.168.0.166:49744.
fserver_recv_data: size: 40, err: -104: Success [0].
Dropped thread 1 for client 192.168.0.166:49744.
Disconnected client 192.168.0.166:49744, operations served: 0.
Dropping worker: 3086465936.
```

Looks good !  Let's try a proper connection with the POHMELFS userspace tool : « cfg ».  The options we'll pass to it are the most basic possible set :

* « -A add » : Action is to add a new connection
* « -a <address> » : Connect to this server
* « -p <port> » : Connect on this port

```bash
[root@host_166 ~]# cfg -A add -a 192.168.0.75 -p 1025
Timed out polling for ack
main: err: -1.
```

Uh oh !  What happened ?  The error message tells us that the client « timed out » (or waited too long) to receive an acknowledgement of the connection from the server.  But why ?  The answer, though simple, is not immediately obvious : we forgot to load the POHMELFS module on the second machine.  No worries, go ahead and do that now, and we'll try again :

```bash
[root@host_166 ~]# modprobe pohmelfs
[root@host_166 ~]# cfg -A add -a 192.168.0.75 -p 1025
[root@host_166 ~]#
```

In a stroke of user-friendliness to last the ages, a successful cfg execution will produce *no* output.  No news is good news, i suppose...
Alright, now that we've got the server up, and we've prepared the client for a connection, we'll need to pick a spot to « mount » the remote share, then initiate the mount itself :

```bash
[root@host_166 ~]# mkdir pohtest
[root@host_166 ~]# mount -t pohmel -o idx=1 none pohtest/
mount: unknown filesystem type 'pohmel'
```

Curses, foiled again !  Well, *i* was, at least - your mileage may vary on this one.  If you get this error instead of a successful mount, the problem is very likely that the « pohmel » file system type isn't declared in all the proper places.  Check « proc » and the « filesystems » etc config :

```bash
[root@host_166 ~]# cat /proc/filesystems | grep poh
nodev    pohmel
```

```bash
[root@host_166 ~]# cat /etc/filesystems | grep poh
[root@host_166 ~]#
```

Ah ha !  Let's rectify that little oversight and try again :

```bash
[root@host_166 ~]# echo "nodev pohmel" >> /etc/filesystems
[root@host_166 ~]# mount -t pohmel -o idx=1 none pohtest/
```

No output ?  Great success !  Let's verify :

```bash
[root@host_166 ~]# df | grep poh
none                 154590376  10007348 144583028   7% /root/pohtest
```

The server console also confirms the connection :

```bash
fserver_root_capabilities: avail: 148053020672, used: 10247524352, export: 0, inodes: 39911424, flags: 2.
Accepted client 192.168.0.166:37617.
```

And, finally, we can see our test file :

```bash
[root@host_166 ~]# cd pohtest
[root@host_166 pohtest]# ls -l
total 0
-rw-r--r-- 1 root root 0 2009-06-16 15:39 POHTEST.TXT
```

Closing the connection cleanly is as simple as umounting :

```bash
[root@host_166 ~]# umount pohtest
```

This is confirmed on the server console :

```bash
Dropped thread 1 for client 192.168.0.166:58955.
Disconnected client 192.168.0.166:58955, operations served: 1.
Dropping worker: 3086400400.
```

### that's a wrap, for now

Now i know what you're thinking : where's the parallel storage ?  Where's the real-time mirroring and all that fun stuff ?  It's coming.  For now, we're just getting our feet wet with technology.  As time and testing permits, i'll post more about POHMELFS - so stay tuned !
**UPDATE** : Check out the next instalment in the series ! <http://www.dark.ca/2009/07/06/pohmelfs-pt-2-return-of-pohmelfs/>
Last but not least, be sure to check out « fserver -h » for such useful features as « fork to background » and « logfile » - both of which, i guaruntee, you'll want to look into if you intend to play around anymore with POHMELFS.
Finally, remember that while the code is very mature for *staging*, it's still considered highly experimental.  Good luck, and happy hacking !