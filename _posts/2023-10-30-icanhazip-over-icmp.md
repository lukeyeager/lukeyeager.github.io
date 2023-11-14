---
layout: post
title: '"icanhazip" over ICMP'
date: 2023-10-30 12:52 -0500
---

# Background

There are several public services which can tell you the public IP address of your computer. Some examples:
* [https://1.1.1.1/cdn-cgi/trace](https://1.1.1.1/cdn-cgi/trace)
* [https://ifconfig.me/](https://ifconfig.me/)
* [https://icanhazip.com/](https://icanhazip.com/)
* [https://www.whatismyip.com/](https://www.whatismyip.com/)

Why do these services exist? Because of [NATs](https://en.wikipedia.org/wiki/Network_address_translation).
You can get your **local** IP address easily (via `ip address` for Linux, or `ipconfig` for Windows).
But if the result is in [one of the private blocks](https://datatracker.ietf.org/doc/html/rfc1918#section-3) (e.g. 192.168.1.2), then you're probably behind a NAT (possibly provided by your home's router) and you need an external service to tell you your public IP.

Implementing the service over HTTP (like the services listed above) as easy as deploying the following nginx configuration:
```nginx
http {
    # ...
    server {
        # ...
        location /getmyip {
            echo "Your IP address is $remote_addr";
        }
    }
}
```

But where's the fun in that? Let's see if we can implement it over ICMP instead, and make it work with a standard tool like `ping` or `mtr`!

Inspired by Louis Poinsignon, who [distributed his CV over IPv6 traceroute](https://news.ycombinator.com/item?id=32609588).

# Version 1: ping
My first thought was, "maybe I can hack `ping` to send a custom response with the public IP embedded in there somewhere?"

Some searching led me to [libnetfilter_log](https://www.netfilter.org/projects/libnetfilter_log/index.html), which makes it possible to send ping responses from userspace.
I liked this approach because writing C in userspace seemed much less scary than writing eBPF.

Starting from [nfulnl_test.c](https://www.netfilter.org/projects/libnetfilter_log/doxygen/html/nfulnl__test_8c_source.html), I was eventually able to modify it to send a valid ECHOREPLY.
At this point, I realized that my primary idea was based on sending the ECHOREPLY from a [partially?] spoofed IP address, and I realized that this wasn't going to work with `libnetfilter_log` for a simple reason: you can't specify the source address from userspace - that's handled in the kernel.
But I was deep enough into the details of `libnetfilter_log` and `ping` that I wanted to come up with something before moving onto another approach.

I was able to modify the ICMP data payload to embed the source IP, but the client could only see the answer if they inspected the packet with tcpdump/wireshark/ngrep/etc.
Not good enough.
Luckily, I noticed while messing around with the payload that the round-trip time (rtt) sometimes printed nonsense values.
Intrigued, I dug into the source code and realized that (in one implementation, at least), [ping encodes the start timestamp into the payload](https://github.com/iputils/iputils/blob/20221126/ping/ping.c#L1530-L1532).
Armed with that dangerous knowledge, here is the first solution I came up with:
```console
$ ip -o address show eth0
18: eth0    inet 192.168.98.7/24 brd 192.168.98.255 scope global eth0\       valid_lft forever preferred_lft forever
                 ^^^^^^^^^^^^
$ ping -c1 192.168.98.1
PING 192.168.98.1 (192.168.98.1) 56(84) bytes of data.
64 bytes from 192.168.98.1: icmp_seq=1 ttl=64 time=1921680980071020 ms

--- 192.168.98.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1921680980071019.925/1921680980071020.032/1921680980071019.925/-5411360113588140.-32 ms
                       ^^^^^^^^^^^^
```
If you squint a bit, you'll see that the source IP (192.168.98.7) is embedded into the RTT.

Full source code: [https://github.com/lukeyeager/whatsmyip-icmp/tree/main/ping-ttl](https://github.com/lukeyeager/whatsmyip-icmp/tree/main/ping-ttl)

Pros:
* It works using a standard client tool, entirely over ICMP - yay!

Cons:
* It depends on an internal detail of a specific `ping` client
* It's kind of hard to read
* It doesn't work if the actual RTT is too long because the low bits get garbled (this is exacerbated by the fact that responding from userspace is very slow)

Let's see if we can do better ...

# Version 2: mtr
My next idea was to send `TIME_EXCEEDED` messages from a spoofed IP so that the source IP address would show up in the list of hops from a `mtr` or `traceroute` trace.
This is similar to Louis Poinsignon's CV example which originally inspired me.

I found [https://taoshu.in/unix/modify-udp-packet-using-ebpf.html](https://taoshu.in/unix/modify-udp-packet-using-ebpf.html), which helped me get started with modifying packets using eBPF.
It took an embarassingly long time after that until I finally got it working.
I was frustrated by the poor quality of the [eBPF helper documentation](https://man.archlinux.org/man/bpf-helpers.7.en) (e.g. it's not clear how to use bpf_csum_diff() and bpf_lX_csum_replace() in concert, it's not clear which helpers automatically update the checksum[s] for you, etc.).
I was comforted by [this guy who also found the checksums maddening](https://hechao.li/2020/04/10/Checksum-or-fxxk-up/).

Finally, I came up with the following working approach:
* For incoming ICMP ECHO packets, store the following data into a BPF HASH MAP
    * Key: `(ip.saddr, icmp.id, icmp.seq)`
    * Value: [the IP header and first 8 bytes of the ICMP message]
* For outgoing ICMP ECHOREPLY packets, lookup the ingress info using the map
* For low TTL values, change the ECHOREPLY response to be a TIME_EXCEEDED message instead, and spoof the source address

It looks like this:
```console
$ ip -o address show dev eth0
17: eth0    inet 192.168.98.7/24 brd 192.168.98.255 scope global eth0\       valid_lft forever preferred_lft forever
                 ^^^^^^^^^^^^
$ mtr -wn 192.168.98.1
Start: 2023-10-30T19:16:39+0000
HOST: 45e5a1953131   Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- ???            100.0    10    0.0   0.0   0.0   0.0   0.0
  2.|-- 192.168.98.192  0.0%    10    0.1   0.1   0.1   0.1   0.0
  3.|-- 192.168.98.168  0.0%    10    0.1   0.1   0.1   0.1   0.0
  4.|-- 192.168.98.98   0.0%    10    0.1   0.1   0.1   0.1   0.0
  5.|-- 192.168.99.7    0.0%    10    0.1   0.1   0.1   0.1   0.0
  6.|-- ???            100.0    10    0.0   0.0   0.0   0.0   0.0
  7.|-- 192.168.98.1    0.0%    10    0.1   0.1   0.1   0.1   0.0
```
See the source address embedded in the last byte of hops 2, 3, 4, and 5?
You could also use `traceroute -In` for the client command and get a similar output.

Full source code: [https://github.com/lukeyeager/whatsmyip-icmp/tree/main/mtr-hops](https://github.com/lukeyeager/whatsmyip-icmp/tree/main/mtr-hops)

Pros:
* It works using either `mtr` or `traceroute`, entirely over ICMP

Cons:
* It only compiles on a recent Archlinux kernel (even Fedora 38 isn't new enough)
* Address spoofing across the internet doesn't work in many cases (e.g. AWS) because the datacenter provider usually filters out packets sent by unexpected IPs
    * I had hoped that sending packets from the same /24 subnet would address the issue, but it doesn't, at least not on AWS

Future work:
* Get the eBPF code to work on older systems
* Make it work for IPv6 (I suspect this would solve my AWS egress filtering problem)

# Conclusion
I had a lot of fun with this project!
I reminded myself of many things I had forgotten regarding IP, ICMP, and C programming.
And I learned a lot about about eBPF, which was new to me.

I'm sure there are many other ways to accomplish this goal (such as [github.com/blechschmidt/fakeroute](https://github.com/blechschmidt/fakeroute) and [github.com/jordiprats/netfilter-icmp2ip](https://github.com/jordiprats/netfilter-icmp2ip)) - let me know what I missed!
