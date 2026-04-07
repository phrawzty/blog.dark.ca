+++
title = "setting the from address in GNU mail"
date = 2009-07-03
+++

Today i'm going to address a question that comes up again and again on the various Linux help forums and mailing lists (in one form or another) :
« How do i set the From address when using mail ? »
Of course, « mail » in this case refers to that most basic of all Linux mail applications : GNU mail.  Commonly part of the default software on a Linux box, it's not used very often in an interactive capacity, but it still gets a *lot* of play in the System Administration world as a way to quickly fire off automated emails from scripts.  For example :

```bash
$ echo "This is the message" | mail -s "This is the subject" user@domain.tld
```

This would result in a message that's *from* whatever raw user@system that the local mailing software detects, such as « systemcheck@2.1.0.192.int.domain.tld » or some such thing.  Normally, whatever defaults (including the From address) that get used are good enough, but occasionally it can be handy to have a finer level of control ; for example, policy restrictions on your local mail relay, or just because the default From address looks bad and you want to make it more readable.
The solution, like so many others in the Linux world, is straightforward once you already know about it, but just obscure enough that it isn't obvious at first.  The man page for « mail » on Ubuntu (and, likely, other distros as well) has this tasty little morsel of information listed as the *very first* option :

```bash
-a, --append=HEADER: VALUE Append given header to the message being sent
```

« Headers » form part of every email message, and are used to store all sorts of information about the email, from such pedestrian items as « From: » and « To: », to more esoteric things like Spam analysis breakdowns and binary encoding methods.  For now, the important thing to realise is that the From: address is contained within the From header, and using the argument noted above, we can set it quite easily :

```bash
$ echo "The message" | mail -s "The subject" --append=FROM:sensible@domain.tld user@domain.tld
```

This would send the same message as above, but with the From: address set to « sensible@domain.tld ».  In fact, *any* header can be altered in this fashion, not just the From header - so if you ever have the need to change other headers, now you know at least one way. :)