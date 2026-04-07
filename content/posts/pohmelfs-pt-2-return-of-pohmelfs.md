+++
title = "pohmelfs pt. 2, return of pohmelfs !"
date = 2009-07-06
+++

Hello again fair readers.  Today i'm going to re-visit POHMELFS, which i introduced in an [earlier blog post](http://www.dark.ca/2009/06/17/pohmelfs-nfs-future/).  I received a comment on that post which basically asked for more information on some of the more interesting (read : advanced) features of POHMELFS, such as distributed storage, and the like.  Well, today is the day !  If you need a refresher, be sure to skim over my previous post, as we're going to dive in now right where i left off last time.

### patch for the win

One of the reasons that there was a bit of a delay between my last POHMELFS post and this one was because i hit a bug.  Given that we're working with staging-level code here, that's to be expected - luckily, thanks to some quick work by [Evgeniy Polyakov on the POHMLEFS mailing list](http://www.ioremap.net/pipermail/pohmelfs/2009-June/000087.html), there is still hope - hope in the form of a tasty little patch.

```bash
diff --git a/drivers/staging/pohmelfs/trans.c b/drivers/staging/pohmelfs/trans.c
index eab7868..bf7b09a 100644
--- a/drivers/staging/pohmelfs/trans.c
+++ b/drivers/staging/pohmelfs/trans.c
@@ -467,7 +467,8 @@ int netfs_trans_finish_send(struct netfs_trans *t, struct pohmelfs_sb *psb)
 				continue;
 		}

-		if (psb->active_state && (psb->active_state->state.ctl.prio >= st->ctl.prio))
+		if (psb->active_state && (psb->active_state->state.ctl.prio >= st->ctl.prio) &&
+				(t->flags & NETFS_TRANS_SINGLE_DST))
 			st = &psb->active_state->state;

 		err = netfs_trans_push(t, st);
```

Basically, this patch fixes a minor, but ultimately crippling bug related to writing to multiple servers.  The details are not important - what's important is that we apply the patch and keep the dream alive.  First, you'll need to copy and paste that block of code into a text file on one of the systems (in « ~/pohmel.diff, for example »).  Then, in order to apply the patch, we'll need to use a standard tool called (appropriately) « patch » :

```bash
[root@host_75 ~]# cd /usr/src/linux
[root@host_75 ~]# patch -p 1 < ~/pohmel.diff
patching file drivers/staging/pohmelfs/trans.c
```

Now, just as we did last time, we must play the kernel and module compilation and installation game (fun!).  If you need a refresher on how to do this, just go back to my [previous post](http://www.dark.ca/2009/06/17/pohmelfs-nfs-future/).  Note that this time around, the whole process will be *much faster*, since only the POHMELFS components need to be recompiled - everything else will stay the same.  As a result, you can skip the part where you archive the entire kernel tree and copy it over - instead, just patch and recompile on each server and the client.  It's your call.
Once that's out of the way we'll reboot, and then it's off to the races.

### a new challenger appears !

It's now time to add a third machine into the mix (« host\_147 » in this case).  Using this new box, we'll create a simple sort of setup which is, in fact, quite representative of how things might work in the real world : two storage servers and a client.  As you no doubt recall, one of the neat features of POHMELFS is that it can be employed in a parallel fashion, meaning that a file which appears to the client to be in one place, is actaully located in more than one storage medium.  A general way of describing these ideas is by using the terms « logical » and « physical » ; the logical medium is the filesystem that the client sees, and the physical medium is the actual hard drive upon which the data is stored.
In this case, host\_75 and host\_166 will be the servers, each containing one copy of the data on their respective physical mediums (i.e. hard drives), and host\_147 will be our client, which will access the data via the logical medium (i.e. the POHMELFS export).  The new machine was set up in the same way as host\_166 was, so we'll skip over that, and get right to the good stuff.
A new directory should be created on each of the machines : « /opt/pohtest ».  This will serve as the export directory on the servers, and the mount directory on the client - don't put any data in it yet, though.

### server config

On the servers, we'll initiate the server daemon.  Unlike our first test, where we just let the defaults ride, this time around we'll configure things a bit more intelligently :

```bash
[root@host_75 (and host_166) ~]# fserver -r /opt/pohtest -l /var/log/pohmelfs.log -d
```

In the above example, « -r » defines the directory to export, « -l » is where to output the logs to, and « -d » puts the process into the background, instead of on our console as before.  This is normally how things would work, so it's good to get used to it now.  Now, we can follow the log files on each machine by using « tail » :

```bash
[root@host_75 (and host_166) ~]# tail -f /var/log/pohmelfs.log
Server is now listening at 0.0.0.0:1025.
```

### client config

With the servers up and ready to go, we can now turn our attention on the client.  Don't forget to load the pohmelfs module first !

```bash
[root@host_147 ~]# modprobe pohmelfs
[root@host_147 ~]# cfg -A add -a 192.168.0.75 -p 1025 -i 1
```

Now we mount.  It's important that we mount *before* we attempt to add the second server into the mix - trying to do it ahead of time will only result in terrible, crippling failure.

```bash
[root@host_147 ~]# mount -t pohmel -o idx=1 none /opt/pohtest/
```

No output means it worked (as usual), so let's verify :

```bash
[root@host_147 ~]# df | grep poh
none                 154590376  10018492 144571884   7% /opt/pohtest
```

Great, now let's add the other server :

```bash
[root@host_147 opt]# cfg -A add -a 192.168.0.166 -p 1025 -i 1
```

Now we must wait *at least 5 seconds* for the synchronisation to occur.  In reality it's shorter than that, but 5 seconds is an easy number to remember, and it's safe.  So far this looks exactly the same as before, but there's a bit of a conceptual twist - as you can see, both of those new add statements have the *same index* (as denoted by the -i).  This means that they're grouped together as part of the same logical medium.  We can check on this by using the « show » action :

```bash
[root@host_147 ~]# cfg -A show -i 1
Config Index = 1
Family    Server IP                                            Port     
AF_INET   192.168.0.75                                         1025
AF_INET   192.168.0.166                                        1025
```

Everything seems on the up and up so far, so we can go ahead and try our first mount.  A series of options will be passed to the mount line, notably « idx=1 », which means index 1 (as seen above) - this is *very* important to specify, as without it, POHMELFS won't be able to determine which logical group you're talking about.
And if we take a look at the log output on the servers, we'll see that the client connection has been accepted.  Both of the logs should show the accepted line, but with different port numbers (the trailing digits at the end) :

```bash
Accepted client 192.168.0.147:48277.
```

There are other diagnostics we can run to take a look at what we've got running.  At this stage they won't tell us anything we don't already know, but it will give us some practice with the tools and data, so that when the time comes to debug problems down the road, we'll be ready.
For example, POHMELFS will write some handy information to « mountstats », which is exactly what it sounds like :

```bash
[root@host_147 ~]# cat /proc/1/mountstats
   ...
device none mounted on /opt/pohtest with fstype pohmel
idx addr(:port) socket_type protocol active priority permissions
1 192.168.0.75:1025 1 6 1 0 3
1 192.168.0.166:1025 1 6 1 0 3
```

It's not lined up very nicely, but the interesting column right now is « active », which lists « 1 » in both cases, meaning the connections are open.  The « permissions » column lists « 3 » for both nodes which, in this case, means that they're both available for reading and writing (as opposed to being read or write-only, which are also valid options).

### but will it blend ?

Accepting the connection is one thing - successfully reading and writing files is entirely another.  Let's do some tests ; first we'll use the client to create an empty file in mount :

```bash
[root@host_147 ~]# cd /opt/pohtest/
[root@host_147 pohtest]# touch FILE
[root@host_147 pohtest]# ls
FILE
```

Great, now let's take a look at our servers :

```bash
[root@host_166 pohtest]# ls -l
total 0
-rw-r--r-- 1 root root 0 2009-07-06 16:58 FILE
[root@host_166 ~]#
```

And the other :

```bash
[root@host_75 ~]# ls -l /opt/pohtest/
total 0
-rw-r--r-- 1 root root 0 2009-07-06 16:46 FILE
[root@host_75 ~]#
```

Now, during my limited tests, i noticed a small lag time between my manipulations on the client, and when those actions were reflected on the servers.  At this stage of the game i'm not sure whether that's normal or not, or exactly what's causing it - so don't be alarmed if you see a small lag as well.  I'll be sure to post further updates on this point once i've got more information.
**Update** : As per Evgeniy on the mailing list :

```bash
This delay is not a bug, but feature - POHMELFS has local cache on
clients and  data written on client is stored in that cache first and
then flushed to the server when client is under memory pressure or when
another one requests updated but not yet flushed data.

To force client to flush the data one can 'sync' on client or use
'flush' utility on the server. The latter will invalidate data on the
client (which implies it to be flushed to the server first), so server
update will become visible next time client reads that data.
```

### how not to do it

Let's do another little test.  On one of the servers, we'll perform a manipulation in the POHMELFS export directory :

```bash
[root@host_75 ~]# touch /opt/pohtest/host75file
[root@host_75 ~]# ls -l /opt/pohtest/
total 4
-rw-r--r-- 1 root root 5 2009-07-06 16:46 FILE
-rw-r--r-- 1 root root 0 2009-07-06 16:57 host75file
[root@host_75 ~]#
```

Great, but if we take a look at the other server :

```bash
[root@host_166 ~]# ls -l /opt/pohtest/
total 4
-rw-r--r-- 1 root root 5 2009-07-06 16:59 FILE
```

And the client :

```bash
[root@host_147 ~]# ls -l /opt/pohtest/
total 0
-rw-r--r-- 1 root root 5 2009-07-06 20:47 FILE
```

We notice that it's not there.  Why ?  Unfortunately, like so much bureaucracy, we didn't go through the proper channels.  Recall that our client has certain software running on it that allows it to speak to both servers, and that the mountpoint uses that software to ensure consistency between across the shared filesystem.  In the example above, we wrote directly to the underlying filesystem of the server - completely avoiding said software - and thus POHMELFS had no way of knowing that a manipulation had occured.
In short - if you want to keep things consistent, you *must* interact via a client.  But what if we want our servers to be able to interact with the data as well ?  Well, there's nothing stopping us from setting up client processes on our servers, too.  This, however, will have to wait for the next instalment.
See you on the intertubes !