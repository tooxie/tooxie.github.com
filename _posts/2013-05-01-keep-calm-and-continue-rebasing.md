---
layout: post
title: Keep calm and continue rebasing
---

<a href="https://news.ycombinator.com/item?id=5643342"><img src="/static/hn.gif" alt="Published to Hacker News"></a>

![Keep calm and continue rebasing](http://sd.keepcalm-o-matic.co.uk/i/keep-calm-and-continue-rebasing.png)

Quite recently an article calling to *stay away from rebase* was published,
in which the author states that you should absolutely never rebase. Well...
that's bullshit.


<h2>
    TL;DR
</h2>

* Work on feature branches and rebase the shit out of it.
* `git pull --rebase` has some not-so-nice side effects.
* Never rebase master. Ever.


<h2>
    Continue rebasing
</h2>

Branches are private playgrounds where you experiment, push broken code and try
out different ideas; when you are happy with the result, you rebase, cleanup
history and issue a PR.

Sometimes you just commit incomplete work to continue it in a different
computer; at work, at home, at the university. Sometimes you commit broken code
for someone else to review and find the bug. Many valid reasons to just commit
crap code.

Keep committing broken code to your feature branches, keep ruthlessly rebasing
feature branches. But be strict about one thing, though: Once it gets merged
into master you don't rebase anymore. **Ever.**

![One does not simply rebase master](http://i.qkme.me/3u6tyi.jpg)

He makes a good point, though, when he says that you shouldn't
`git pull --rebase` because your commits will preserve timestamps and lose
the chronological order in which they were created. That's true, could be an
issue, and it's up to you to decide if you prefer a clean history or a
chronological one.

Although, someone comments in the original article:

> The so-called "timestamp problem" is a case of confusing UI. All commits,
> including rebased commits, have a correct and chronological commit
> timestamp (unless you manually change yours for some reason), but the
> author timestamp, which is preserved by rebase, tends to be shown more
> prominently. So that's not an inherent problem with rebase and could have
> been avoided with foreknowledge (but is imo a problem with the UI, not the
> users).

So, before you `--abort` every rebase you had in progress, just keep calm.
And here is one last meme, just to make it clear:

![You're gonna have a bad time](http://i.qkme.me/3u70t9.jpg)

<!-- vim:filetype=markdown
-->
