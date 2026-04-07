+++
title = "Easily keep your local git repo up to date"
date = 2024-07-12
+++

*(This post originally appeared on [dev.to](https://dev.to/phrawzty/easily-keep-your-local-git-repo-up-to-date-l52))*



Here's how to keep your clone of a Git repository current with upstream quickly and efficiently. Just pop this snippet into the `.git/config` of your local checkout and you're good to go (some assembly required).



```bash
[remote "upstream"]
    url = <http url>
    fetch = +refs/heads/*:refs/remotes/upstream/*
[alias]
    refresh = "!git checkout main && git fetch upstream && git merge upstream/main main"
```



Run `git refresh` before you branch locally, and just like magic, you'll avoid merge conflicts and other irritating side effects. It's a bit dirty, but it'll do the job in most cases. 😉



Want to know more about what's going on here? Great!




---




First: *why?* A few of the open source projects I work on have very active repositories, and since they all favour trunk-based development, that means `main` gets updated frequently. Furthermore, they all prefer pull requests to be proposed from clones, as humans don't touch the primary repository directly.



This is a really common model, so nothing special; however I *always* forgot to rebase before branching, except when I didn't forget to rebase, because I'd forgotten *how* to rebase. ([Git is fun!](https://jvns.ca/blog/2024/01/26/inside-git/))



So instead of having to do a little dance every time, I figured: 💭 *let the computer do the work*. And while incantations are great, I'm cranky enough to want to know what it is the computer is doing on my behalf, so let's break it down:



```bash
[remote "upstream"]
    url = <http url>
    fetch = +refs/heads/*:refs/remotes/upstream/*
```



The `[remote]` block defines a *remote* repository, i.e. a repo that isn't on your computer. Here I've given it an arbitrary name (`upstream`). Arbitrary because it's only used in *this* configuration context, so go with your heart on this one.



Inside that block there's `url = <http url>`, which—you guessed it—defines the URL where the remote repository lives. This could be a link to [Codeberg](https://codeberg.org/), GitHub, GitLab, or whatever y'all got going on.



Next is `fetch`, which literally tells git how to *fetch* updates from that remote repository. That's the easy part. What do those other bits mean?



The `+` tells git to "force" update the references. If you're familiar with Git, this means forcing the references even if the result is a non-fast-forward update. Remember at the top of the post when I said "It's a bit dirty"? This is what I'm talking about. But here's the thing: it's *probably* what you want anyway. 😇



The `refs/heads/*` part just tells git to get all the branches from the remote repo. Again: it's *probably* what you want.



Finally, `:refs/remotes/upstream/*` specifies where those fetched branches are going to be stored locally. There's a whole different blog post to be written about *just this*, but the tl;dr is we want to keep the upstream branches separate from the local branches, because things get crazy graph-theory style if we don't. 😝



OK, now the part you've all been waiting for: the *actual command*!



```bash
[alias]
    refresh = "!git checkout main && git fetch upstream && git merge upstream/main main"
```



The `[alias]` block defines, well, an alias. It's basically a shortcut that lets us run arbitrary commands or sequences (including, notably, shell commands) so that we don't have to write them out by hand each time. In this case, the alias is `refresh`; the idea here being that we can type `git refresh` on the shell and it *runs the command sequence*, thus simplifying our lives somewhat. And some is better than none.



Also: remember at the top of the post when I said "It's a bit dirty"? This shell command is *actually* what I'm talking about. 🥸 Let's be legends…



The `!` right at the start of the sequence tells git to treat the whole thing as a shell command, so whatever comes next will just be executed like we typed it out.



First we make sure that we're on the `main` branch in our local repo by running `git checking main`. It's possible that your repo uses a different name for trunk ("master" was popular for a long time), but "main" is common.



Then we `git fetch upstream` to nab that fresh upstream data from the remote repository. Remember `upstream`? It's the remote repository we defined previously. Ah what fun we had.



Finally, we *merge* the changes from the `main` branch of `upstream` into the `main` branch of our local repository. That's literally how to read `git merge upstream/main main`. 🧠



And that's it! A couple of lines gets us a quick, reusable, fairly reliable method of keeping our local repo up to date.



Hope that helps! Let me know!




---




BONUS: You might also be wondering what those `&&` are for in between each of the commands. The Very Correct Definition is that `&&` is a "logical AND operator". What it's used for in this case is a way to chain a series of shell commands into a sequence, whereby the *next* command will only be called if the *current* one was successful. This way, we don't end up doing horrible things to our local repo ~~if~~ when things go wrong.