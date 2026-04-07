+++
title = "CPAN RPMs in RHEL / CentOS : generation, conflict, and solutions"
date = 2010-04-08
+++

Hello all !  Today we're going to take a look at a somewhat obscure problem that - once encountered - can cause nothing but headaches for a system administrator.  The problem relates to conflicts in CPAN RPM packages, and what can be done to work around the issue.  If you've made it this far, i'm going to assume a couple of things : you're comfortable with RPMs and repositories, have worked with a .spec file before, and you know what Perl modules are.  Good ?  Ok, let's go.
*Edit : About a week after i posted this article, the pastebin i uploaded the examples to disappeared.  Maybe it will come back - i don't know - but if not, sorry for the broken links...*
[CPAN](http://www.cpan.org/) is an enormous collection of Perl modules.  If you've ever written a Perl script, there's a good chance you've used a module that - at one point or another - came from this archive.  One of the really neat features of CPAN is the [interactive manner](http://search.cpan.org/~jhi/perl-5.8.0/lib/CPAN.pm#Interactive_Mode) in which modules can be downloaded and installed from the archive using Perl right from the command line (frankly, if you're reading this post, there's a good chance you've used this feature, too).  This is a fairly common way to install new modules and add functionality to your system, especially if you're coding for local use (i.e. on your personal box).
It's useful, but it's not perfect, and one of the key areas where it starts to fail is scalability : if you've got a bunch of machines, and you need to SSH into each one to interactively install a CPAN module or two, it's going to be a hassle.  Likewise, CPAN doesn't often find its way into the hearts and minds of enterprise Red Hat or CentOS environments, where the official policy is often to install software via RPM only (for support, administration, and sanity reasons, this is often the case).
Luckily, some of the most commonly used CPAN modules exist as RPMs in the default repositories.  Some, but not all (and not even « many ») - for this, there are other repositories available.  Some examples :

* [EPEL](http://fedoraproject.org/wiki/EPEL)
* [Dag](http://dag.wieers.com/rpm/)
* [RPMForge](https://rpmrepo.org/RPMforge/)
* [Magnum Solutions](http://rpm.mag-sol.com/)

That last one - Magnum - is particularly interesting given the subject of our post today.  From their info page :
> At Magnum we have a firm rule that all CPAN modules on our machines are installed from RPMs. The Fedora and Centos projects build RPMs for many CPAN modules, but there are always ones missing and the ones that are available often lag behind the most up to date versions.  For that reason, we build a lot of RPMs of CPAN modules. And we don't want to keep that work to ourselves, so on these pages we make them available for anyone to download.

Their RPMs are generated automagically using a great tool called « cpanspec », which does exactly what you think it does : given a CPAN tarball, it will generate a .spec file suitable for building an installable RPM.  It is available in the standard repositories, and can be installed easily via YUM as normal, so go ahead and do that now.  Ok, example time : say you needed HTML::Laundry, but after a quick peek through your repositories, it becomes readily apparent that an RPM is not available.  Thanks to cpanspec, all is not lost :

```bash
[build@host-119 ~]$ wget http://search.cpan.org/CPAN/authors/id/S/ST/STEVECOOK/HTML-Laundry-0.0103.tar.gz
[build@host-119 ~]$ cpanspec --packager "build <build@domain.ext>" HTML-Laundry-0.0103.tar.gz
```

We just downloaded the tarball right from the CPAN website, and ran cpanspec against it.  The « --packager » argument simple defines the person who's generating the .spec, and doesn't necessarily have to be anything accurate.  Go ahead and try it for yourself.  Now take a look at the resulting .spec file (or on the a pastebin [here](http://pastebin.ca/1858756)).  As you can see, it fills in all the fields, including the critical (and often tricky-to-determine) « BuildRequires » and « Requires » items.  Frankly, it's solid gold, and it has made the lives of CentOS / RHEL admins all over the world much easier.
That said, it's not perfect, and there are times when you might run into problems.  Actually, you may run into two problems in particular.  The first is conflicts over ownership, which arises when multiple RPMs claim to be responsible for the same file (or files, or directories, or features, or whatever).  The second is more nefarious : an RPM that writes files to the system without declaring ownership for them - a condition often referred to as « clobbering ».  The former is irritating, but at least it's not destructive, unlike the latter, which can cause all manner of headaches.  To illustrate these two problems, let's take a look at another example (this one being decidedly more real-world than that of Laundry above) : [CGI.pm](http://search.cpan.org/dist/CGI.pm/lib/CGI.pm).
The [.spec file](http://pastebin.ca/1858885) that is generated from this tarball is functional and correct, and we can build an installable RPM out of it, so at first all appears well.  Again, go ahead and try for yourself - i'll wait.  You may wish to capture the build output for review - otherwise, check the [pastebin](http://pastebin.ca/1858890).  I'd like to draw your attention to the « Installing » lines.  By trimming the « Installing /var/tmp/perl-CGI.pm.3.49-1-root-root » element from each of those lines, we can see the actual paths and files that this RPM will install to.  Examples :

```bash
/usr/lib/perl5/vendor_perl/5.8.8/CGI.pm
/usr/lib/perl5/vendor_perl/5.8.8/CGI/Cookie.pm
/usr/lib/perl5/vendor_perl/5.8.8/CGI/Util.pm
/usr/share/man/man3/CGI.3pm
/usr/share/man/man3/CGI::Pretty.3pm
/usr/share/man/man3/CGI::Cookie.3pm
```

At first glance this looks perfectly acceptable.  But look what happens when we try to install the resulting RPM (clipped for brevity) :

```bash
[root@host-119 build]# rpm -iv /usr/src/redhat/RPMS/noarch/perl-CGI.pm-3.49-1.noarch.rpm
Preparing packages for installation...
file /usr/share/man/man3/CGI.3pm.gz from install of perl-CGI.pm-3.49-1.noarch conflicts with file from package perl-5.8.8-27.el5.x86_64
file /usr/share/man/man3/CGI::Cookie.3pm.gz from install of perl-CGI.pm-3.49-1.noarch conflicts with file from package perl-5.8.8-27.el5.x86_64
file /usr/share/man/man3/CGI::Pretty.3pm.gz from install of perl-CGI.pm-3.49-1.noarch conflicts with file from package perl-5.8.8-27.el5.x86_64
```

As it turns out, the Perl package that comes with RHEL / CentOS already contains CGI.pm.  This is normal, since it's so popular, and is included as a convenience.  Thus, RPM - in an attempt to preserve the coherence of the package management system - refuses to install overtop of the existing owned files.  This is a fine illustration of the first of the two problems previously noted : conflicts over ownership.  As i mentioned above, it's aggravating, but it's not a bug - it's a feature, and it's doing exactly what it's designed to do.  Irritating, but not ultimately dire.
If you look carefully, though, it's also an illustration of the second problem.  Note the list of files that are conflicting.  Look back to the list of files that the package contains - notice anything missing from the conflicts list ?  That's right - the actual module files (\*.pm) are not showing conflicts, which means they'd get overwritten without complaint by RPM.  You might be thinking « who cares ? that's what i want » right now, but trust me, it's not what you want.  Imagine this CGI package, with this version of CGI.pm gets installed, and then later you upgrade the Perl package - your CGI.pm files will get overwritten by the Perl package, because as far as RPM is concerned, Perl owns those files.  All of a sudden, things break because you had scripts that relied on your particular version, but since you just upgraded Perl, you think (quite naturally) that the problem could be anywhere - where do you even start looking ?
Imagine the headache if there are multiple administrators, multiple servers, multiple data centres, and multiple clients paying multiple dollars.  No fun at all.
So how can we upgrade CGI.pm, using an RPM, without running into these problems ?  As is often the case, the answer is deceptively simple, but not immediately obvious.  Ultimately what we want to accomplish is twofold :

* Avoid the man conflicts.
* Ensure that the existing owned module files are not clobbered by our new package.

Concerning the man pages - and i'm going to be perfectly blunt here - the solution is to simply not install them, since, of course, they're already there.  As for avoiding a clobbering condition, this requires a little bit of investigation into how Perl modules and libraries are stored on an RHEL / CentOS machine.  Consider the following output :

```bash
[root@host-119 ~]# ls -d /usr/lib64/perl5/*
/usr/lib64/perl5/5.8.8  /usr/lib64/perl5/site_perl  /usr/lib64/perl5/vendor_perl
```

What's it all mean ?  Well, the « 5.8.8 » directory is the default directory as defined by the Perl architecture, and is system and platform-agnostic, which is to say that it's (supposed to be) the same on every system.  The « vendor\_perl » directory contains everything that specific to RHEL / CentOS (the « vendor » of the distribution).  As you may recall from the rpmbuild output above, this is where the RPM wants to install the modules (thus creating the clobbering condition).
There's a third directory there, promisingly named « site\_perl » ; as the name implies, this is where site-specific files are stored, which is to say items that are neither part of the default Perl architecture, nor part of the RHEL / CentOS distribution.  As you've no doubt guessed by now, site\_perl is where we're going to put our new modules.
Luckily for us, the only thing that needs to be changed is the .spec file - and we even get a headstart, since cpanspec does most of the heavy lifting for us.  Examining the [.spec file](http://pastebin.ca/1858885) once more, we see the following lines of note (again, cut for brevity) :

```bash
%build
%{__perl} Makefile.PL INSTALLDIRS=vendor
%files
%{perl_vendorlib}/*
```

These indicate that the target installation directory is that of the vendor, which is normally the case, and thus the default setting.  Since we want to install to the site directory, we make the following changes :

```bash
%build
%{__perl} Makefile.PL INSTALLDIRS=site
%files
%{perl_sitelib}/*
```

That solves our clobbering problem quite nicely, but what about the man files ?  As i mentioned above, the idea is to simply avoid installing them altogether, but since they're generated automatically during the build process, how can we exclude them ?  What i'm about to present is a bit of a hack, but it's absolutely effective, and ultimately quite clean : we delete them after they've been generated, and then don't declare them in the file list.  Some items are already being potentially deleted by default, so let's go ahead and add our own line into the mix :

```bash
find $RPM_BUILD_ROOT -depth -type d -exec rmdir {} 2>/dev/null ;
# destroy manified man, man.
find $RPM_BUILD_ROOT -type f -name '*.3pm' -exec rm -f {} ;
```

This will look for all of the « manified » man files and just remove from the build tree.  All that's left now is to remove them from the file list.  This is as simple as deleting (or commenting out) their sole declaration :

```bash
#%{_mandir}/man3/*
```

Another option is to simply install use the « --excludedocs » argument when installing the RPM.  I opted to remove the docs altogether in order to ensure that the package can be installed without errors by anyone else without needed to know about the argument requirement ahead of time (and to facilitate automated rollouts).
What you'll end up with is a .spec file that [looks like this](http://pastebin.ca/1858951).  Go ahead and build your RPM - it'll install without conflicts and without danger.  This is a technique that can be used for other CPAN packages as well, so go ahead and install everything you've always wanted.