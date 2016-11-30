---
layout: post
title:  "Upgrading pip: a shell quirk"
date:   2016-11-29 16:55:34
---

I stumbled across a confusing issue the other day when trying to upgrade pip.

{% highlight bash %}
# I had installed pip with a deb package
$ sudo apt-get install python-pip

# Ubuntu 14.04 installs a pretty old version of pip
$ pip --version
pip 1.5.4 from /usr/lib/python2.7/dist-packages (python 2.7)

# So I decided to upgrade it
$ sudo pip install --upgrade pip

# But it didn't appear to work
$ pip --version
pip 1.5.4 from /usr/lib/python2.7/dist-packages (python 2.7)

# I double-checked which verison of pip was being used
$ which pip
/usr/local/bin/pip

# It looks like the new version of pip is being picked up,
#   so why is it still reporting the old version?
# Just for kicks, I tried calling the new binary directly:
$ /usr/local/bin/pip --version
pip 9.0.1 from /usr/local/lib/python2.7/dist-packages (python 2.7)
{% endhighlight %}

Huh!? Why would `pip --version` and ``` `which pip` --version``` return different things?

As it turns out, there was a linux shell feature working against me.
The shell maintains a hash table for the locations of executables so they don't have to search through all files on each path in `PATH` over and over again for the same executable.

{% highlight bash %}
# The hash-table remembers the previous location of pip
$ hash
hits	command
   1	/usr/bin/pip

# Reset the value for pip in the hash-table
$ hash pip

# And voila, we get the new version of pip!
$ pip --version
pip 9.0.1 from /usr/local/lib/python2.7/dist-packages (python 2.7)
{% endhighlight %}

For reference, here is the relevant section of the dash manpage (which I find more legible than the corresponding section of the bash manpage):

<pre>
$ man dash
...
     hash -rv command ...
            The shell maintains a hash table which remembers the locations of
            commands.  With no arguments whatsoever, the hash command prints
            out the contents of this table.  Entries which have not been
            looked at since the last cd command are marked with an asterisk;
            it is possible for these entries to be invalid.

            With arguments, the hash command removes the specified commands
            from the hash table (unless they are functions) and then locates
            them.  With the -v option, hash prints the locations of the com‐
            mands as it finds them.  The -r option causes the hash command to
            delete all the entries in the hash table except for functions.
</pre>
