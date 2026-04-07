+++
title = "load balancing, or, « how i learned to love a piranha »"
date = 2009-10-21
+++

Hello again, everybody !  Today i thought that we'd take a look at a fun and useful topic of interest to many system administrators : load balancing & redundancy.  Now, i know, it doesn't *sound* too exciting - but trust me, once you get your first mini-cluster set up, you'll never look at service management quite the same way again.  It's not even that tough to set up, and you can get a basic setup going in almost no time at all, thanks to some great open source software that can be found in more or less any modern repository.
First, as always, a little bit of theory.  The most basic web server setup (for example), looks something like figure 001, below :
[caption id="attachment\_133" align="alignnone" width="243" caption="Figure 001"]![Figure 001](http://www.dark.ca/wordpress/wp-content/uploads/2009/09/simple-web_server-no_lb.png "simple-web_server-no_lb")[/caption]
As you can see, this is a functional setup, but it does have (at least) two major drawbacks :

* A critical failure on the web server means the *service* (i.e. the content being served) disappears along with it.
* If the web server becomes overloaded, you may be forced to take the entire machine down to upgrade it (or just let your adoring public deal with a slow, unresponsive website, i suppose).

The solution to both of these problems forms the topic of this blog entry : *load balancing*.  The idea is straightforward enough : by adding more than one web server, we can ensure that our service continues to be available even when a machine fails, and we can also spread the love, er, load, across multiple machines, thus increasing our overall efficiency.  Nice !

### batman and round robin

Now, there are a couple of ways to go about this, one of which is called « Round Robin DNS » (or RRDNS), which is both very simple and moderately useful.  [DNS](http://en.wikipedia.org/wiki/Domain_Name_System), for those needing a refresher, is (in a nutshell) the way that human-readable hostnames get translated into machine-readable numbers.  Generally speaking, hostnames are tied to IP addresses in a *one-to-one* or *many-to-one* fashion, such that when you type in a hostname, you get a single number back.  For example :

```bash
$ host www.dark.ca
www.dark.ca has address 88.191.66.127
```

In other words, when you type *www.dark.ca* into your browser, you get one particular machine on the Internet (as indicated by the address); however, it is also possible to set up a *one-to-many* relationship - this is the basis or RRDNS.  A very common example is Google :

```bash
$ host www.google.com
www.google.com is an alias for www.l.google.com.
www.l.google.com has address 74.125.39.99
www.l.google.com has address 74.125.39.103
www.l.google.com has address 74.125.39.104
www.l.google.com has address 74.125.39.105
www.l.google.com has address 74.125.39.106
www.l.google.com has address 74.125.39.147
```

So what's going on here ?  In essence, the Google administrators have created a situation whereby typing in *www.google.com* into your browser will get you one of a whole group of possibilities.  In this way, each time you request some content from them, one of any number of machines will be responsible for delivering that service.  (Now, to be fair, the *reality* of what's going on at Google is likely far more complex, but the premise is identical.)  Your web browser will only get *one* answer back, which is more or less randomly provided by the DNS server, and that response is the machine you'll interact with.  As you can see, this (sort of) satisfies our problem of resource usage, and it (sort of) addresses the problem resource failure.  For those of you who are more visually inclined, please see figure 002 below :
[caption id="attachment\_137" align="alignnone" width="299" caption="Figure 002"]![Figure 002](http://www.dark.ca/wordpress/wp-content/uploads/2009/09/rrdns.png "rrdns")[/caption]
It's not perfect, but it *is* workable, and most of all, it's dead simple to set up - you just need to set your DNS configuration up and you're good to go (an exercise i leave to you, fair reader, as RRDNS is not really the focus of our discussion today).  Thus, while RRDNS is a simple method for implementing a rudimentary load balancing infrastructure, it still has notable failings :

* The load balancing isn't systematic at all - by pure chance, one machine could end up getting hammered while others do very little, for example.
* If a machine fails, there's a chance that the DNS response will contain the address of the downed machine.  In other words, the chances of you getting the downed machine are 1 in X, where X is the number of possible responses to the DNS query.  The odds get better (or worse, depending on how you look at it) as more machines fail.
* A slightly more obscure problem is that of response caching : as a method of optimisation, many DNS systems, as well as software that interacts with DNS, will *cache* (hold on to) hostname lookups for variable lengths of time.  This can invalidate the magic of RRDNS altogether...

### another attack vector

Another approach to the problem, and the one we'll be exploring in great depth in this article, is using a dedicated load balancing infrastructure, combining a handful of great open source tools and proven methodologies.  First, however, some more theory.
Our new approach to load balancing must propose both a solution to the original problems (critical failure & resource usage), as well as address and solve the drawbacks of RRDNS as noted above.  Really, what we want is an intelligent (or, at least, systematic) distribution of load across multiple machines, and a way to ensure that requests don't get sent to downed machines by accident.  It'd be nice if these functions were automated too, since the last thing an administrator wants to do is baby-sit racks of servers.  What we'd like, in other words, could be represented by replacing the phrase « RRNDS » in figure 002 above, with the word « magic ».  For now, let's imagine that this magic sits on a machine that we'll call « Load Balancer » (or LB, for short), and that this LB machine would have a similar conceptual relationship to the web servers as RRDNS does.  Consider figure 003 :
[caption id="attachment\_143" align="alignnone" width="299" caption="Figure 003"]![Figure 003](http://www.dark.ca/wordpress/wp-content/uploads/2009/09/simple-web_server-lb.png "simple-web_server-lb")[/caption]
This is a basic way of thinking about what's going to happen.  It looks a lot like figure 002, but there is a very important difference : instead of relying on the somewhat nebulous concept of DNS for our load balancing, we can now give that responsibility to a proper machine running and dedicated to the purpose.  As you can imagine, this is already a huge improvement, since this opens the door to all sorts of additional features and possibilities that simply aren't possible with straight DNS.  Another interesting aspect of this diagram is that, visually speaking, it would appear that the Internet cloud only « sees » one machine (the load balancer), even though there are a number of web servers behind it.  This concept of having a single point of entry lies at the very core of our strategy - both figuratively *and* *literally* - as we'll soon discover
In the here and now, however, we're still dealing with theory, and a solution based on « magic » is about as theoretical as it gets. Luckily for us though, magic is *exactly* what we're about to unleash - in the form of « [Linux Virtual Server](http://www.linuxvirtualserver.org/) », or « LVS » for short.  From their homepage :

The Linux Virtual Server is a highly scalable and highly available server built on a cluster of real servers, with the [load balancer](http://kb.linuxvirtualserver.org/wiki/Load_balancer) running on the Linux operating system. The architecture of the server cluster is fully transparent to end users, and the users interact as if it were a single high-performance virtual server. [...] The Linux Virtual Server as an advanced load balancing solution can be used to build highly scalable and highly available network services, such as scalable web, cache, mail, ftp, media and VoIP services.

The thing about LVS is that while it's not inherently complex, it is highly malleable, and this means you really do need to have a solid handle on exactly *what* you want to do, and *how* you want to do it, before you start playing around.  Put another way, there are a myriad of ways to use LVS, but you'll only use one of them at a time, and picking the right methodology is important.  The best way to do this is by building maps and really getting a solid feel for how the various components of the overall architecture relate to each other.  Once you've got a good mental idea of what things should look like, actually configuring LVS is about as straightforward as it gets (no, really!).

### let's complicate the issue further, for science !

Looking back to figure 003, we can see that our map includes the Internet, the Load Balancer, and some Web Servers.  This is a pretty typical sort of setup, and thus, we can approach it from a few different ways.  One of the decisions that needs to be made fairly early on, though, has more to do with topology and routing than LVS specifically : how, exactly, do the objects on the map relate to each other at a network level ?  As always, there can be lots of answers to this question - each with their advantages and disadvantages - but ultimately we must pick only one.  Since i value *simplicity* when it comes to technology, figure 004 describes a simple network topology :
[caption id="attachment\_147" align="alignnone" width="358" caption="figure 004"]![figure 004](http://www.dark.ca/wordpress/wp-content/uploads/2009/09/dr-network_map.png "dr-network_map")[/caption]
Now, for those of you out there who may have some experience with LVS, you can see exactly where this is headed - for everybody else, this might not be what you were expecting at all.  Let's take a look at some of the more obvious points :

* There are two load balancers.
* The web servers are on the same network segment as the LBs.
* Unlike the previous diagrams, the LBs do not appear to be « in between » the Internet and the web servers.

The first point is easy  : there are two LBs for reasons of redundancy, as a single LB represents a single point of failure.  In other words, if the LB stops working for whatever reason, all of your services behind it become functionally unavailable, thus, you really, *really* want to have another machine ready to go immediately following a failure.
A little bit more explanation is required to explain the second and third points - but the short answer is two words : « [Direct Routing](http://kb.linuxvirtualserver.org/wiki/LVS/DR) » (or DR for short).  From the LVS wiki :

Direct Routing [is] an IP load balancing technology implemented in LVS. It directly routes packets to backend server through rewriting MAC address of data frame with the MAC address of the selected backend server. It has the best scalability among all other methods because the overhead of rewriting MAC address is pretty low, but it requires that the load balancer and the backend servers (real servers) are in a physical network.

If that sounds heavy, don't worry - figure 005 explains it in easy visual form :
![figure 005](http://www.dark.ca/wordpress/wp-content/uploads/2009/09/dr-logic.png "dr-logic")
In a nutshell, requests get sent to the LB, which then passes it to the Web Server, who in turn responds directly to the client.  It's fast, efficient, scalable, and easy to set up, with the only caveat being that the LBs and the machines they're balancing *must* be *on the same network*.  As long as you're willing to accept that restriction, Direct Routing is an excellent choice - and it's the one we'll be exploring further today.

### a little less conversation, a little more action

So with that in mind, let's get started.  I'm going to be describing four machines in the following scenario.  All four are identical off-the-shelf servers running CentOS 5.2 - nothing fancy here.  The naming and numbering conventions are simple as well :
**[TABLE=2]**
You probably noticed the fifth item in this list, labelled « Virtual Web Server ».  This represents our *virtual*, or *clustered* service, and is not a real machine.  This will be explained in further detail later on - for now, let's go ahead and install the key software on **both** of the Load Balancer machines :

```bash
[root@A01 etc]# yum install ipvsadm piranha httpd
```

« ipvsadm » is, as you might have guessed, the administrative tool for « [IPVS](http://www.linuxvirtualserver.org/software/ipvs.html) », which is in turn an acronym for « Internet Protocol Virtual Server », which makes more sense when you say « IP-based Virtual Server » instead.  As the name implies, IPVS is implemented at the IP level (which is more generically known as Layer-3 of the [OSI model](http://en.wikipedia.org/wiki/OSI_model)), and is used to spread incoming connections to one IP address towards other IP addresses according to one of many pre-defined methods.  It's the tool that allows us to control our new load balancing infrastructure, and is the key software component around which this entire exercise revolves.  It is powerful, but sort of a pain to use, which brings us to the second item in the list : piranha.
Piranha is a web-based tool (hence httpd, above) for administering LVS, and is effectively a front-end for ipvsadm.  As installed in CentOS, however, the Piranha package contains not only the PHP pages that make up the interface, but also a handful of other tools of particular interest and usefulness that we'll take a look at as well.  For now, let's continue with some basic setup and configuration.
A quick word of warning : *before* starting « piranha-gui » (one of the services supplied by Piranha) up for the first time, it's important that both LBs have the *same time* set on them.  You've probably already got [NTP](http://en.wikipedia.org/wiki/NTP) installed and functioning, but if not, here's a hint :

```bash
[root@A01 ~]# yum -y install ntp && ntpdate pool.ntp.org && chkconfig ntpd on && service start ntpd
```

Moving right along, the next step is to define a login for the Piranha web interface :

```bash
[root@A01 ~]# /usr/sbin/piranha-passwd
```

You can define multiple logins if you like, but for now, one is certainly enough.  Now, unless you plan to run your load balanced infrastructure on a completely internal network, you'll probably want to set up some basic restrictions on who can access the interface.  Since the interface is served via an instance of [Apache HTTPd](http://httpd.apache.org/), all we have to do is set up a normal « [.htaccess](http://httpd.apache.org/docs/2.0/howto/htaccess.html) » file.  Now, a full breakdown of .htaccess (and, in particular, [mod\_access](http://httpd.apache.org/docs/2.0/mod/mod_access.html)) is outside of the scope of this document, but the simple jist is as follows :

```bash
[root@A01 ~]# cat /etc/sysconfig/ha/web/secure/.htaccess
Order deny,allow
Deny from all              # by default, deny from everybody
Allow from 192.168.0       # requests from this network are allowed
```

With those items out of the way, we can now activate piranha-gui :

```bash
[root@A01 ~]# chkconfig piranha-gui on && service piranha-gui start
```

Congratulations !  The interface is now running on port 3636, and can be accessed via your browser of choice - in the case of our example, it'd be « http://A01:3636/ ».  The username for the web login is « piranha », and the password is the one we set above.  Now that we're logged in, let's take a look at the interface in greater depth.

### look out - piranhas !

The first screen - known as the « Control » page - is a summary of the current state of affairs.  Since nothing is configured or even active, there isn't a lot to see right now.  Moving on to the « Global Settings » tab, we have our first opportunity to start putting some settings into place :

* Primary server public IP : Put the IP address of the « primary » LB.  In this example, we'll put the IP of A01.
* Private server public IP : If we *weren't* using direct routing, this field would need a value.  In our example, therefore, it should be empty.
* Use network type : Direct Routing (of course!)

On to the « Redundancy » tab :

* Redundant server public IP : Put the IP address of the « secondary » LB.  In this example, we'll put the IP of A02.
* Syncdaemon : Optional and useful - but know that it requires additional configuration in order to make it work.
  + This feature (which is *relatively* new to LVS) ensures that the state information (i.e. connections, etc..) are shared with the secondary in the event that a failover occurs.  For more information, please see [this page](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.server_state_sync_demon.html) from the LVS Howto.
  + It is *not* necessary, strictly speaking, so we can just leave it unchecked for now.

Under the « Virtual Servers » tab, let's go ahead and click « Add », then select the new unconfigured entry and hit « Edit » :

* Name : This is an arbitrary identifier for a given clustered service.  For our example, we'd put « WWW ».
* Application Port : The port number for the service - HTTP runs on port 80, for example.
* Protocol : TCP or UDP - this is normally TCP.
* Virtual IP Address : This is the IP address of the *virtual* service (VIP), which you may recall from the table above.  This is the IP address that clients will send requests to, regardless of the *real* IP addresses (RIP) of the *real* servers which are responsible for the service.  In our example, we'd put 192.168.0.40 .
  + Each service that you wish to cluster needs a *unique* « address : port » pairing.  For example, 192.168.0.40:80 could be a web service, and 192.168.0.40:25 would likely be a mail service, but if you wanted to run *another, separate* web service, you'd need to assign a different virtual IP.
* Virtual IP Network Mask : Normally this is 255.255.255.255, indicating a *single* IP address (the Virtual IP Address above).
  + You can actually cluster subnets, but this is outside of the scope of this tutorial.
* Device : The Virtual IP address needs to be assigned to a « [virtual network interface](http://www.google.fr/search?q=virtual+network+interface) », which can be named more or less anything, but generally follows the format « ethN:X », where N is the physical device, and X is an incremental numeric identifier.  For example, if your physical interface is « eth0 », and this is the *first* virtual interface, then it would be named « eth0:1 ».
  + If and when you set up multiple virtual interfaces, it is important to **not** mix these up.  Piranha has no facility for sanity checking these identifiers, so you may wish to track them yourself in a [Google document](http://docs.google.com/) or something.
* Scheduling : There are a number of options here, and some are *very* different from one another.  For the purposes of this exercise, we'll pick a simple, yet relatively effective scheduler called « Least-Connections ».
  + This does exactly what it sounds like : when a new request is made to the virtual service, the LB will check to see how many connections are open to each of the real servers in the cluster, and then route the connection to the machine with the *least* connections.  Congrats, you've now got load balancing !

Finally, let's add some real servers into the cluster.  From the « Edit » screen we're already in, click on the « Real Server » sub-tab.

* Name : This is the hostname of the real server.  In our example, we'd put B01.
* Address : The IP address of the real server.  In our example, for B01, we'd put 192.168.0.38 .
* Port : Generally speaking this can be left empty, as it will inherit the Port value defined in the virtual service (in this case, 80).
  + A value would be required here if your real server is running the service on a *different* port than that specified in the virtual service ; if your real server is running a web service on port 8080 instead of 80, for example.
* Weight : Despite the name, this value is used in various different ways depending on which Scheduler you selected for the virtual service.  In our example, however, this field is irrelevant, and can be left empty.

You can apply and add as many real servers as you like, one at a time, in this fashion.  Go ahead and set up B02 (or whatever your equivalent is) now.
If you're wondering when the *secondary* LB is going to be configured, well, wonder no longer : the future is now.  Luckily, this step is very, very easy.  From the secondary :

```bash
[root@A02 ~]# scp root@A01:/etc/sysconfig/ha/lvs.conf /etc/sysconfig/ha/
```

### now is a good time to grab a beer

Phew !  That was a lot of work.  After consuming a suitable refreshment, let's move on to the final few steps.  Earlier i mentioned that there were some other items that we'd need to learn about besides the Piranha interface - « [Pulse](http://www.redhat.com/support/wpapers/piranha/x32.html) » is one such item.  Pulse, as a tool, is in the same family as some other tools you may have heard of, such as « [Heartbeat](http://www.linux-ha.org/) », « [Keepalived](http://www.keepalived.org/) », or « [OpenAIS](http://www.openais.org/doku.php) ».
The basic idea of all of these tools is simple : to provide a « [failover](http://en.wikipedia.org/wiki/Failover) » facility between a group of two or more machines.  In our example, our primary LB is the one that is normally active, but in the case that it fails for some reason, we'd like our secondary to click in and take over the responsibilities of the unavailable primary - this is what Pulse does.  Each of the load balancers runs an instance of « pulse » (the executable, not the package), which behaves in this fashion :

* Each LB sends out a broadcast packet (a pulse, as it were) stating that they are alive.  As long as the active LB (commonly the primary) continues to announce itself, everybody is happy and nothing changes.
* If, however, the inactive LB (commonly the secondary) server notices that it hasn't seen any pulses from the active LB lately, it assumes that the active LB has failed.
* The secondary, formerly inactive LB, then becomes active.  This state is maintained until such a time as the primary starts announcing itself again, at which point the secondary demotes itself back to inactivity.

The difference between the active and the inactive server is actually very simple : the active server is the one with the virtual addresses assigned to it (remember those, from the Virtual Servers tab in Piranha?).
Let's go ahead of start it up (on the primary LB first, then on the secondary) :

```bash
[root@A01 ~]# chkconfig --add pulse
[root@A01 ~]# service pulse start
```

### an internet ballgame drama - in 5 parts

You may have noticed that we haven't even touched the « real » servers (i.e. the web servers) yet.  Now is the time.  As it so happens, there's only one major step that relates to the real servers, but it's a very, very important one : defining VIPs, and then ensuring that the web servers are OK with the routing voodoo that we're using to make this whole load balancing infrastructure work.  The solution is simple, but the *reason* for the solution may not be immediately obvious - for that, we need to take a look at the IP layer of each packet (neat!).  First, though, let's run through a series of little stories :

* Alice has a ball that she'd like to pass to Bob, so she tosses it his way.
* Bob catches the ball, sees that it's from Alice, and throws it back at her.  What great fun !

Now imagine that Alice and Bob are hanging out with a few hundred million of their closest friends - but they still want to play ball.

* Alice writes Bob's name on the ball, who then passes it to somebody else, and so forth.
* Eventually the ball gets passed to Bob.  Unfortunately for Bob, he has no idea where it came from, so he can't send it back.

The solution is obvious :

* Alice writes « From : Alice, To : Bob » on the ball, the passes it along.
* Bob gets the ball, and switches the names around so that it says « From : Bob, To : Alice », and sends it back.

OK, so, those were some nice stories, but how do they apply to our Load Balancing setup ?  As it turns out, all we need to do is throw in [some tubes](http://www.google.fr/search?q=the+internet+is+a+series+of+tubes), and we've described one of the basic functions of the Internet Protocol - that the source and destination IP addresses of a given packet are part of the IP layer of said packet.  Let's complicate it by one more level :

* Alice prepares the ball as above, and send it flying.
* Bob gets the ball, who's been avoiding Alice since things got weird at the bar last week-end, passes it along to Charles.
* Charles - who's had a not-so-secret crush on Alice since high school - happily writes « From : Charles, To : Alice », and tosses it away.
* Alice receives the ball, but much to her surprise, it's from Charles, and not Bob as she expected. *Awkward !*

With that last story in mind, let's take another look at figure 005 above (go ahead, i'll wait).  Notice anything ?  That's right - the original source sends their packet off, but then receives a response from a *different* machine than they expected.  This *does not work* - it violates some basic rules about how communications are supposed to function on the Internet.   For the thrilling conclusion - and a solution to the problem - let's return to our drama :

* As it turns out, Bob is a player : he gets so many balls from so many women that he needs to keep track of them all in a little notebook.
* When Bob gets Alice's ball he passes it to Charles, then he records where it came from and who he gave it to in his notebook
* Charles - in an attempt to get into Bob's circle of friends - agrees to write « From : Bob, To : Alice » on the ball, then sends it back.
* Alice - expecting a ball from Bob - is happy to receive her Bob-signed spheroid.
* Bob then gets another ball from Denise, passes it to Edward, and records this relationship as well.
* Edward - a sycophant if ever there was - prepares the ball in the same fashion as Charles, and fires it back.

Of course, the more balls Bob has to deal with, the more helpers he can use to spread the work around.  Now, as you've no doubt pieced together, Alice and Denise are any given sources on the Internet, Bob is our LB, and Charles & Edward are the web servers.  Now, instead of writing people's names on balls, we should now make the mental leap to IP addresses in packets.  With our tables of hostnames and addresses in mind, let's consider the following example :

* The source sends a request for a web page.
  + The source IP is « 10.1.2.3 », and the destination IP is « 192.168.0.40 » (the VIP for WWW).
* The packet is sent to A01, which is currently active, and thus has the VIP for WWW assigned to it.
* A01 then forwards the packet to B02 (by chance), which crafts a response packet.
  + The RIP for B02 is « 192.168.0.39 », but instead of using that, the source IP is set to « 192.168.0.40 », and the destination is « 10.1.2.3 ».
* The source, expecting a response from « .40 », indeed receives a packet that appears to be from WWW.  Done and done.

The theory is sound, but how can we implement this in practice ?  As i said - it's simple !  We simply add a *dummy interface* to each of the web servers that has the same address as the VIP, which will allow the web servers to interact with packets properly.  This is best done by creating a simple sysconfig entry on each of the web servers for the required dummy interface, as follows :

```bash
[root@B01 ~]# vim /etc/sysconfig/network-scripts/ifcfg-lo:0
# for VIP
DEVICE=lo:0
IPADDR=192.168.0.40
NETMASK=255.255.255.255
BROADCAST=192.168.0.40
ONBOOT=yes
NAME=vip0
```

[root@B01 ~]# vim /etc/sysconfig/network-scripts/ifcfg-lo:0

# for VIP

DEVICE=lo:0

IPADDR=192.168.0.34

NETMASK=255.255.255.255

BROADCAST=192.168.0.34

ONBOOT=yes

NAM**all together now**

The « lo » indicates that it's a « [Loopback address](http://en.wikipedia.org/wiki/Loopback#Network_equipment) », which is best described by Wikipedia :

Such an interface is assigned an address that can be accessed from management equipment over a network but is not assigned to any of the real interfaces on the device. This loopback address is also used for management datagrams, such as alarms, originating from the equipment. The property that makes this virtual interface special is that applications that use it will send or receive traffic using the address assigned to the virtual interface as opposed to the address on the physical interface through which the traffic passes.

In other words, it's a fake IP that the machine can use to make packets anyways.  Now, there is a known scenario in which a machine with a given loopback address will, in this particular situation, cause confusion on the network about which interface actually « owns » a given address.  It has to do with ARP, and interested readers are encouraged to Google for « [LVS ARP problem](http://www.google.fr/search?q=lvs+arp+problem) » for more technical details - for now, let's just get right to the solution.  On each of the real servers, we'll need to edit « sysctl.conf » :

```bash
[root@B01 ~]# vim /etc/sysctl.conf
```

```bash
# this file already has stuff in it, so put this at the bottom
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
```

Now, restart [sysctl](http://en.wikipedia.org/wiki/Sysctl) :

```bash
[root@B01 ~]# sysctl -p
```

That's it - problem solved.

### all together now !

At this point we've now explored each key item that is necessary to make this whole front-end infrastructure work, but it is perhaps not quite clear how it all works together.  So, let's take a step back for a moment and review :

* There are four servers : two are load balancers, and two are web servers.
  + Of the two load balancers, only one is active at any given time ; the other is a backup.
* Every DNS entry for the sites on the web servers points to *one* actual IP address.
  + This IP address is called the « Virtual IP ».
  + The VIP is claimed by the *active* load balancer, meaning that when a request is made for a given website, it goes to the active LB.
* The LB then re-directs the request to an actual web server.
  + The re-direction can be random, or based on varying levels of logical decision making.
  + The web server will respond *directly* - the LB is not a proxy.

Great !  Now, what software runs where, and why ?

* The load balancers use LVS in order to manage the relationship between VIPs and RIPs.
* Pulse is used between the LBs in order to determine who is alive, and which one is active.
* An optional (but useful) web interface to both LVS and Pulse comes in the form of Piranha, which runs on a dedicated instance of Apache HTTPd on port 3636.

And that, my friends, is that !  If you have any questions, feel free to comment below (remember to subscribe to the RSS feed for responses).  Happy balancing !

#### oh, p.s., one last thing...

In case you're wondering how to keep your LVS configuration file synchronised across both of the load balancers, one way to do it would be with a network-aware filesystem - [POHMELFS](http://www.dark.ca/2009/06/17/pohmelfs-nfs-future/), for example. ;-)