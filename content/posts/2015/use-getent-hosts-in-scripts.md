---
title: "Use Getent Hosts in Scripts"
summary: "It is common practice to use tools like 'dig' to lookup DNS results, but this has very bad performance. If you just need to lookup a name or address, there's an easier, friendly way."
date: 2015-04-15
draft: false
tags: ["DNS", "Programming"]
---

Okay, so the excrement of the day is flying from the fan and you need to get some quick analytics of who/what is being the most-wanted of the day. Perhaps you don't currently have the analytics you need at your fingertips, so you hack up a huge 'one-liner' that perhaps takes a live sample of traffic (tcpdump -p -nn ...), extracts some data using sed, collates the data using awk etc. and then does some sort of reporting step. Yay, that's what we call agile analytics (see 'hyperbole'); its an all-too-common fallback methodology that does prove to be immensely useful as a first step.

Okay, so you've got some report, perhaps it lacks a bit of polish, but it contains IP addresses, and we'd like to see something more recognisable. So, you scratch your head a bit and have the usual internal debate "do I do this bash, or fall up to awk, perl/python". At this point (if you go with bash etc. or awk), you'll perhaps think of using `dig +short -x $IP` to get (you hope) the canonical DNS name associated with it.

Oh, oh. A common point of trouble at this part is that you end up calling `dig` many times... often for the same name in a short period of time. Perhaps you'll request things too fast and make a nuisance of yourself. As someone who looks after DNS servers, I urge you to stop. There is a much better way, and that way is the cache-friendly `getent hosts`... it will likely do what you need.

When you call `dig`, `host`, or `nslookup`, you are going directly to DNS, as they are specifically DNS querying and diagnostic tools. Any local resolver is bypassed. What a regular program does when it calls `gethostbyname(...)` or `getnameinfo(...)` and related functions, is to look for the presence of a local cache (eg. it will look explicitly for the presence of a UNIX-domain socket /var/run/nscd/socket) and it will consult files like /etc/resolv.conf. Both of these get used to potentially cache DNS queries.

> Hopefully you're not using nscd to cache 'hosts' if you care about small TTLs, but you'll often find things like dnsmasqd or SystemD resolver.

This is why sometimes it can be useful to use 'ping' to lookup DNS resolution... but if your doing that, you perhaps don't know about 'getent'.

The `getent` command is part of your standard Linux environment; it comes with glibc. It's purpose is as a lookup/testing tool for things like /etc/passwd, /etc/groups, /etc/services, and also name resolution and many others --- see the manual page for getent(1). As `getent` is simply a command that exposes API functionality from glibc, you get that potentially huge benefit from caching.

To do a forward lookup, you can use the 'hosts' database. Forward lookups are not as pleasant compared to say `dig +short -t a ...` if you expect an IPv4 address (I'll leave you to play with forward lookups yourself). Reverse lookups are very simple.

Let's get a test-case:

```bashsession
$ host www.blogger.com
www.blogger.com is an alias for blogger.l.google.com.
blogger.l.google.com has address 216.58.220.105
blogger.l.google.com has IPv6 address 2404:6800:4006:801::2009

$ host 216.58.220.105
105.220.58.216.in-addr.arpa domain name pointer syd10s01-in-f9.1e100.net.
```

Okay, so if we do a reverse lookup on 216.58.220.105, we expect to see (this will vary depending where you in the world; topologically it seems I'm close to India) syd10s01-in-f9.1e100.net.

```bashsession
$ getent hosts 216.58.220.105
216.58.220.105  syd10s01-in-f9.1e100.net
```

Hooray. Let's see what a negative result looks like.

```bashsession
$ getent hosts 210.7.45.44
   ... 0 lines of output
```

The output is meant to be exactly what /etc/hosts could contain; a single IP address, a canonical names, and a list of aliases.

> Remember, 'dns' and 'files' are both valid backends for 'hosts' in /etc/nssswitch.conf

## Sample

Nice and script-friendly. Let's see how to pull that in AWK (or GAWK), with a simple example first. Let's start off with some input -- perhaps lines with an IP address and some count. I've also include a threshold, just as a reminder that it's good to minimise the number of lookups.

```bashsession
$ echo -e '1.1.1.1 2\n8.8.8.8 12\n8.8.4.4 25' \
  | awk '
    BEGIN {threshold = 5}
    $2 > threshold {
      "getent hosts " $1 | getline getent_hosts_str;
      split(getent_hosts_str, getent_hosts_arr, " ");
      print $1, getent_hosts_arr[2], $3
    }'
8.8.8.8 google-public-dns-a.google.com
8.8.4.4 google-public-dns-b.google.com
```

## Real-world example

Well, the previous example was reasonably real-world, but its useful to see something a bit more fully-fledged.

I have a script called `dns-live-sample-frequent-each-second` which runs `tcpdump`, and basically outputs the most frequent client (with a minimum threshold) every second. So this 'one-liner', which is not yet encapsulated into a script, will look at 100 samples (ie. 100 seconds) and output a table of the common sources of such spikes.

The first script (`dns-live-sample-frequent-each-second`) has output like the following (I've anonymised the IP addresses). The fields are time, IP and count of requests that client made that second. I'm not going to show the script contents in this post.

```plain
13:19:36 1.8.2.1 10
13:19:37 1.8.2.1 36
13:19:38 1.8.5.2 30
13:19:39 10.2.3.4 23
```

Here's the script that takes the output from above and performs the lookups and make it look a bit pretty.
If you care to expand it, you'll see that its using the `PROCINFO["sorted_in"]` functionality of `gawk` to sort an associated array by its values in descending numerical order. That's certainly a trick worth knowing.

```bash
#!/bin/bash

num_seconds="${1:-120}"

echo Looking for spiking clients; sampling each second for $num_seconds seconds...

#!/bin/bash

dns-live-sample-most-frequent-each-second \
  | head -n "$num_seconds" \
  | gawk '
    {clients[$2] += 1}
    END {
        print "CLIENT_IP CLIENT_DNS TOP1%";
        rest = 0;
        PROCINFO["sorted_in"] = "@val_num_desc";
        for (client in clients) {
            if (clients[client] >= 0.05*NR) {
                "getent hosts " client | getline getent_hosts_str;
                split(getent_hosts_str, getent_hosts_arr, " ");
                print client, getent_hosts_arr[2], int(clients[client] / NR * 100)
            } else {
                rest += clients[client]
            }
        }
        print "REST -", int(rest/NR*100)
    }' \
  | column -t
```

This produces the following:

```plain
CLIENT_IP  CLIENT_DNS         TOP1%
1.2.3.9    blah.example.com   9
1.2.3.32   croak.example.com  18
REST       -                  73
```

Right, now to go have a chat with whoever looks after croak.example.com.
