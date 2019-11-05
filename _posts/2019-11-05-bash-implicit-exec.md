---
layout: post
title: Implicit exec's in Bash
---

In a typical Bash script, each command is executed by calling fork and exec.

Here's what that looks like:
{% highlight bash %}
# We'll use strace to follow the fork+exec syscalls
# Here, we create an alias with some useful flags to make the output less verbose
$ alias STRACE='strace -fqq -e signal=none -e status=successful -e trace=execve'

# Typical program - two commands, each run in a separate process
# You can tell that fork() was called because of the "[pid XXX]" sections
$ STRACE bash -c 'sleep 1 ; hostname'
execve("/usr/bin/bash", ["bash", "-c", "sleep 1 ; hostname"], 0x7ffdc84f5770 /* 42 vars */) = 0
[pid 139443] execve("/usr/bin/sleep", ["sleep", "1"], 0x55e3139282f0 /* 42 vars */) = 0
[pid 139444] execve("/usr/bin/hostname", ["hostname"], 0x55e313928c70 /* 42 vars */) = 0
lyeager-dt
{% endhighlight %}

In your Bash script, you can choose to "exec" a command, which skips the fork and executes the program directly, without returning control back to Bash afterwards.

{% highlight bash %}
# Notice how the second command is missing the [pid] section
# This means that we skipped the fork and ran exec directly
$ STRACE bash -c 'sleep 1 ; exec hostname'
execve("/usr/bin/bash", ["bash", "-c", "sleep 1 ; exec hostname"], 0x7fff0a83ca08 /* 42 vars */) = 0
[pid 141756] execve("/usr/bin/sleep", ["sleep", "1"], 0x560642aba330 /* 42 vars */) = 0
execve("/usr/bin/hostname", ["hostname"], 0x560642abacb0 /* 41 vars */) = 0
lyeager-dt
{% endhighlight %}

Today, I noticed that when you give a single command to the "-c" Bash flag (e.g. bash -c hostname), there is an implicit "exec" inserted before your command.

{% highlight bash %}
# Notice how the hostname command is not run in a separate process
$ STRACE bash -c 'hostname'
execve("/usr/bin/bash", ["bash", "-c", "hostname"], 0x7ffe4265fd18 /* 42 vars */) = 0
execve("/usr/bin/hostname", ["hostname"], 0x557d7410a330 /* 42 vars */) = 0
lyeager-dt

# Here, the exec is explicit. The output is the same.
$ STRACE bash -c 'exec hostname'
execve("/usr/bin/bash", ["bash", "-c", "exec hostname"], 0x7fff2499fa28 /* 42 vars */) = 0
execve("/usr/bin/hostname", ["hostname"], 0x55e4f1424330 /* 42 vars */) = 0
lyeager-dt
{% endhighlight %}

As far as I can tell, this is entirely undocumented in bash(1).

But it actually enables something pretty cool.
This chain of bash+numactl+bash+numactl programs all completes without ever needing to fork.

{% highlight bash %}
# Notice how all of the subcommands are exec'd in the same process
$ STRACE bash -c 'numactl --membind=0 -- "$@"' -- bash -c 'numactl --show'
execve("/usr/bin/bash", ["bash", "-c", "numactl --membind=0 -- \"$@\"", "--", "bash", "-c", "numactl --show"], 0x7ffca329ed70 /* 42 vars */) = 0
execve("/usr/bin/numactl", ["numactl", "--membind=0", "--", "bash", "-c", "numactl --show"], 0x564ce6e6c300 /* 42 vars */) = 0
execve("/usr/bin/bash", ["bash", "-c", "numactl --show"], 0x7fff37bc6230 /* 42 vars */) = 0
execve("/usr/bin/numactl", ["numactl", "--show"], 0x55e870a982f0 /* 42 vars */) = 0
policy: bind
preferred node: 0
physcpubind: 0 1 2 3 4 5 6 7 8 9 10 11
cpubind: 0
nodebind: 0
membind: 0
{% endhighlight %}
