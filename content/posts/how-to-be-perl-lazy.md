+++
title = "how to be properly lazy, with perl !"
date = 2009-06-12
+++

One of the wonderful things about Perl is that it enables the busy System Administrator to be lazy - and that's a good thing ! Of course, i don't mean lazy as in unmotivated, or possesed of a poor work ethic, i mean it in the sense that Perl lets us do as little work as possible in a wide variety of situations. Let's examine this idea, shall we ?
In the computer world, one often finds themselves doing the same sorts of things over and over again, such as adding a new user to the network, or verifying that the backups executed properly. Usually, these are relatively simple processes which are less about problem solving, and more about following the same set of steps over and over until the desired goal is attained. It is in these situations that the (properly) lazy admin identifies a way to automate as much as possible these processes, so that he or she can get back to more brain-intensive work (this has the net effect of improving overall efficiency and value - see how laziness pays off in the end ? :) )
There are, of course, as many scripting and programming languages as there are grains of sand on a beach, but despite the many competitors and alternatives out there, Perl remains the language of choice for many Linux admins around the world. This is in no small part due to Perl's ability to manipulate data in a rapid, logical, and easily deployable manner - the most obvious example of this being the vaunted « Perl One-Liner ».

### example !

There comes a time in every admin's life when they must take a bunch of text files, and systematically swap some of the text within with new data - commonly known as searching and replacing.  You could certainly do this by hand using an editor or by using a relatively straightforward C program if you were so inclined.  But there is another way - a better, smarter, *lazier* way : the Perl search & replace one-liner !  Let's take a look at the code, then break down each component.

```bash
$ perl -p -i -e 's/oldandbusted/newhotness/' *.txt
```

That's it, you're done - take a lap and hit the showers.  So, what exactly just happened there ?  We employed a classic and very common usage method in command-line Perl which can easily be remembered as « pie » :

* « -p » : In a nutshell, this tells Perl to loop through each line of input, then perform the desired action (in this case, the search & replace) against each of those lines.
* « -i » : This instructs Perl to actually edit the input files directly (or « in place »), instead of just displaying the changes on the screen.
* « -e » : This describes exactly one line of code - in this case, the search and replace regular expression...
* « 's/old/new/' » : This is the regular expression (or « regex ») which Perl will use to perform the search & replace.  (What's a regex ?  [Wikipedia has the answers you seek](http://en.wikipedia.org/wiki/Regular_expression) !)
* « \*.txt » : The target filename - in this case, a simple glob.  (What's a glob ?  [Wikipedia has the answer](http://en.wikipedia.org/wiki/Glob_(programming)) !)

The key to this whole operation was the fourth bullet point - the regex.  Don't worry if your regex-fu is not yet strong - this is just an example, and it could have been anything - the point is that Perl can be used to rapidly execute regular expressions on data in simple, easy to execute ways, such as the search & replace one-liner above.  This sort of thing comes in handy on a *daily basis*, and thus, the perl one-liner is a powerful tool in the System Administrator's toolbox.
For more one-liners, use the Google : <http://www.google.fr/search?q=perl+one-liners>