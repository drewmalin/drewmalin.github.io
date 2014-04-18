---
layout: post
title: "[Bash] Incorporating Git into the Command Line"
date: 2014-04-17 20:00:00
category: development
---

I was recently wondering about how plugins like oh-my-zsh incorporate the status of the currently-checkout out git branch into the command line prompt. Turns out you can do the same very easily with bash! 

First, some basic info on modifying the command line prompt in the first place. The following options (taken from the bash man page) show the various information that can be added to the prompt:

{% highlight text %}
\a     an ASCII bell character (07)
\d     the date in "Weekday Month Date" format (e.g., "Tue May 26")
\D{format} the format is passed to strftime(3) and the result is inserted into the prompt string;
       an empty format results in a locale-specific time representation.  The braces are required
\e     an ASCII escape character (033)
\h     the hostname up to the first `.'
\H     the hostname
\j     the number of jobs currently managed by the shell
\l     the basename of the shell's terminal device name
\n     newline
\r     carriage return
\s     the name of the shell, the basename of $0 (the portion following the final slash)
\t     the current time in 24-hour HH:MM:SS format
\T     the current time in 12-hour HH:MM:SS format
\@     the current time in 12-hour am/pm format
\A     the current time in 24-hour HH:MM format
\u     the username of the current user
\v     the version of bash (e.g., 2.00)
\V     the release of bash, version + patch level (e.g., 2.00.0)
\w     the current working directory, with $HOME abbreviated with a tilde (uses the value of the
       PROMPT_DIRTRIM variable)
\W     the basename of the current working directory, with $HOME abbreviated with a tilde
\!     the history number of this command
\#     the command number of this command
\$     if the effective UID is 0, a #, otherwise a $
\nnn   the character corresponding to the octal number nnn
\\     a backslash
\[     begin a sequence of non-printing characters, which could be used to embed a terminal
       control sequence into the prompt
\]     end a sequence of non-printing characters
{% endhighlight %}

A very contrived example of applying this to the command line prompt is as follows (entered at the bottom of ~/.bashrc):

{% highlight bash %}
export PS1="\u - \s\v \w \t $ "
{% endhighlight %}

This results in something like the following:
![screenshot1]({{ site.url }}/assets/screenshot1.png)

We can additionally add a bit of color to the prompt by incorporating a bit of additional information:

{% highlight bash %}
GREEN="\[\033[0;32m\]"
YELLOW="\[\033[0;33m\]"
BLUE="\[\033[1;34m\]"
RESET="\[\033[00m\]"
export PS1="$GREEN \u - $YELLOW \s\v \w $BLUE \t $ $RESET"
{% endhighlight %}

Which results in a nicer result (note that the RESET color ensures that any entered command does not become colorized):

![screenshot2]({{ site.url }}/assets/screenshot2.png)

And now to add some git info. Git makes this very easy by providing a handle into the status of the currently checked out branch (if one exists) with *\_\_git\_ps1*. Additionally, the optional boolean *GIT\_PS1\_SHOWDIRTYSTATE*, if set, will update the git information if the current branch has uncommitted changes.

After incorporating these options (and updating the prompt to look a little nicer) the bottom of ~/.bashrc now looks something like:

{% highlight bash %}
GREEN="\[\033[0;32m\]"
YELLOW="\[\033[0;33m\]"
BLUE="\[\033[1;34m\]"
RESET="\[\033[00m\]"

export GIT_PS1_SHOWDIRTYSTATE=1
export PS1="$BLUE\u $YELLOW\w$GREEN"'$(__git_ps1)'"$YELLOW \$$RESET "
{% endhighlight %}
And the command line itself now shows the appropriate git branch and status, if applicable:

![screenshot3]({{ sire.url }}/assets/screenshot3.png)

