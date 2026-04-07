+++
title = "getting Dia to give you a pdf"
date = 2009-07-16
+++

Hello again !  Today's quick tip concerns a software package called [Dia](http://en.wikipedia.org/wiki/Dia_(software)), which is an open source tool (available for both Windows and Linux, as it goes) used to make diagrams, flowcharts, network maps, and so forth.  It has its own file format (.dia), which is (obviously?) useful for saving the projects you're working on, but less useful if you need to give the diagram to anybody else, either in print or electronic form.
Dia can export to a variety of formats including [SVG](http://en.wikipedia.org/wiki/Scalable_Vector_Graphics), [PNG](http://en.wikipedia.org/wiki/Portable_Network_Graphicshttp://en.wikipedia.org/wiki/Portable_Network_Graphics), and [EPS](http://en.wikipedia.org/wiki/Encapsulated_PostScript), but one export format that it lacks native support for is the venerable PDF, which has become a de facto standard for transmitting documents between diverse environments.  There are many advantages and interesting aspects of the PDF format, not the least of which being that what you see on your screen is what you get when it's printed.  It is unfortunate, then, that Dia won't spit out a PDF (even if you ask very nicely).
Of course, being that it's so easy to print directly to PDF (via [CUPS](http://en.wikipedia.org/wiki/CUPS), for example) these days, having native support for PDF may not, at first, seem all that useful.  Well, as it turns out, printing directly to PDF might not give you quite what you were looking for.  In practice, you *do* get a PDF, but what appeared to be a modestly-sized diagram in Dia will turn out to be a multi-page monster in (virtually) printed form.  As a general rule, this is not what you want.
In order to get a usable PDF we need to use an intermediate step between Dia and the final file.  The idea, quite simply, is to export the diagram as one of the supported formats, then convert that file into a PDF.  There are a number of options here, but for our purposes we'll save the diagram as an EPS file, then use a quick little command-line tool called « epstopdf » to perform the conversion.
There's a good chance that you don't have epstopdf on your machine.  If you're using Ubuntu, you *used* to be able to install it easily via the APT packager, but these days the little conversion tool comes as part of a larger suite of tools called « texlive-extra-utils ».  This suite is dependant on a number of other packages, so go ahead and install them all :

```bash
$ sudo apt-get install texlive-extra-tools
```

EDIT : In Ubuntu 10.04, the package is named « texlive-font-utils ».
Among many, many little items of interest, our target application will be installed.  To use it, simply feed it the name of the EPS file as an argument :

```bash
$ epstopdf somediagram.eps
```

It will automatically output a PDF file of the same name.  There you go - a nice, shiny PDF of your Dia diagram.
Enjoy !