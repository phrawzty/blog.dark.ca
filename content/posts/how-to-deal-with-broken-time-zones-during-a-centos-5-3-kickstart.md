+++
title = "how to deal with broken time zones during a CentOS 5.3 kickstart"
date = 2009-12-01
+++

Hello again fair readers !  Today's quick tip concerns the problem with missing time zones when deploying CentOS 5.3 (and some of the more recent Fedoras) in a kickstart environment.  It's a [known problem](https://bugzilla.redhat.com/show_bug.cgi?id=481617), and unfortunately, since the source of the problem (an incomplete time zone data file) lies deep in the heart of the kickstart environment, fixing it directly is a distinct pain in the buttock region.
There is, however, a workaround - and it's not even that messy !  The first step is to use a region that *does* exist, such as « Europe/Paris », which will satisfy the installer - then set the time zone to what you *actually* want after the fact in the « %post » section.  So, in the top section of the kickstart file, we'll put :

```bash
# set temporarily to avoid time zone bug during install
timezone --utc Europe/Paris
```

The « --utc » switch simply states that the *system* clock is in UTC, which is pretty standard these days, but ultimately optional.  Next, in the %post section towards the end, we'll shoe horn our little hack fix into place :

```bash
# fix faulty time zone setting
mv /etc/sysconfig/clock /etc/sysconfig/clock.BAD
sed 's@^ZONE="Europe/Paris"@ZONE="Etc/UTC"@' /etc/sysconfig/clock.BAD > /etc/sysconfig/clock
/usr/sbin/tzdata-update
```

So, what's going on there ?  Let's break it down :

* In the first line, we're just backing up the original configuration file, to use in the next line...
* The second line is the important one - this is the actual manipulation which will fix the faulty time zone, setting it to whatever we want.  In this example « Etc/UTC » is used, but you can pick whatever is appropriate.
  + The tool being used here is « sed », a non-interactive editor which dates back to the 1970's, and which is still used by system administrators around the world every day.
  + The command we're issuing to sed is between the single quotes - astute readers will notice that it's a [regular expression](http://www.google.com/search?q=regular+expression+tutorial), but with @'s instead of the more usual /'s.  In it, we simply state that the instance of « ZONE="Europe/Paris" » is to be replaced with « ZONE="Etc/UTC" ».
  + This change is to be made against the backup file, and outputted to the actual config.
* Finally, we run « tzdata-update » which, as you've no doubt guessed, updates the time zone data system-wide, based (in part) on the newly-corrected clock config.

And that, as they say, is that.  Happy kickstarting, friends, and i'll see you next time !