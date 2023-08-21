---
title: "Dont Let Java on Linux Determine Its Own Timezone"
date: 2014-09-01
draft: false
summary: "Your timezone says Pacific/Auckland but Java reports Antarctica/SouthPole; what gives?"
tags: ["Java", "Oracle Applictaions"]
---

Abstract: In this post, I illustrate one reason why its useful not to let Java (at least on Linux) determine its own timezone. This ends up being particularly important when considering inter-version compatibility issues. This post is not specific to any product, but does use Oracle WebCenter as a real-life case-study.

I've been diagnosing a fault on some of our Java environments with Oracle WebCenter 11gR1 on RHEL6. The nature of the fault is such that we don't know when it started occurring, the machine (and the feature involved) is not in routine use.

In Web Center, you can transfer data between environments using an 'Archiver' (which at least in earlier versions is also a ... topic for some other conversation). As part of configuring Archiver, you need to create a 'Provider' (a way of getting information in or out of the system). Each provider has a useful 'Test' button.... or at least, it has a 'Test' button, and when you click it, it should say 'good'. Ours said 'bad', until I fixed it today.

Diagnosing the issue was annoying; the logs were not particularly useful (and I couldn't figure out which module to enable logging to capture that information)... but if you've dealt with this product you perhaps know my frustration.

I think I've seen an error message somewhere in the dim past that may have clued me to the significance of what I'm about to describe to you... but do you think I can find it now? No, but I think it mentioned the timezone in some sorrounding text it was complaining about... I didn't have the luxury to chase it at the time.

Oracle Web Center is based around a scripting (exaggeration?) language called IDOC, and the 'Test' button for testing a provider performs an IDOC PING.... but my knowledge of this part of the system is very limited, so let's move on...

The fault was occurring only between our old-to-new providers; old-to-old and new-to-new was fine. The old and new environments differ in OS release, Java release, and UCM/WebCentre release.

I'm in the fortunate position of being able to inspect traffic going between the two systems, and I could see what was happening at the application-layer protocol. One server would perform a IDOC PING, the other would respond [with an HTTP response]. I captured the traffic and compared the old-to-new and old-to-old and new-to-new interactions to determine what the issue might be. There were a number of differences... but in every case, the transaction reported success.

The most 'interesting' (but, you might think, reasonably benign) difference was that the request (in the old environment) mentioned a timezone of Pacific/Auckland, while the response came from Antartica/McMurdo... odd, I know I live in the south of New Zealand, but not that far south! Another new enviroment was reporting Antarctica/SouthPole. I wonder if that is actually the problem? **So perhaps the problem is not the transaction, but the parsing of the response**. It is a good example of why timezone names are not particularly useful in network interactions [they belong, along with DST, localised to a system]. An old system trying to parse an unknown time-zone would serve to explain this. Curiouser and curiouser, we start to fall down the rabbit hole, although if this is a Wonderland, it is surely [that of Tim Burton](http://en.wikipedia.org/wiki/Alice_in_Wonderland_%282010_film%29).

We have a support-agency that was helping us with this fault, and they were almost adamant (after having lodged a ticket with Oracle) that it must have been a network fault, which was why I was doing another network trace in the first place; as I was pretty certain that wasn't the case.

Allowing ourselves to believe that this timezone issue must in fact be the case, we hit Google, and found this bug report [JDK-6456628 : (tz) Default timezone is incorrectly set occasionally on Linux](https://www.blogger.com/blog/post/edit/3273223626601426546/7272465944165779884#). That gives a very good explanation of how the timezone detection works [on Linux]. In brief, it tries the following:

1. The TZ environment variable (common in UNIX, I gather, but rarely seen on Linux) 
1. It does a faulty attempt to parse /etc/sysconfig/clock 
1. It looks to read the link /etc/localtime, which is not a link in modern Linux systems 
1. It then looks for files under /usr/share/zoneinfo/ that have the same content/hash as /etc/localtime

Step 4 is faulty. In New Zealand, most of the country would use the timezone Pacific/Auckland. But if you happen to work in Antarctica, you might use Antarctica/McMurdo, which despite having the same timezone offset, has a different name; this is not unreasonable at all. But there is no useful sort-order that it can do, and so when traversing /usr/share/zoneinfo/, it comes out in directory-order (which is basically unsorted), so some systems would say Pacific/Auckland, while others might get Antartica/McMurdo etc.

JVMs also supposedly [if what Stack Overflow tells me is correct] allow you to use `-Duser.timezone=Pacific/Auckland`, but when I tried that it didn't work in WebLogic with an Oracle JDK.

So in principle, set the `TZ` environment variable.

Having learned this, I set it globally, in `/etc/profile.d/tz.sh`

```bash
# CRK 2014-09-01 Don't let Java refer to a faulty heuristic for determining
# time zone; otherwise you might end up in Antarctica/McMurdo or similar.
#
TZ=Pacific/Auckland
export TZ
```

Not enough, after restarting WebLogic, I found that it still didn't see the environment variable.

You can verify by looking at the process information. First, determine the PID of the process that would have the information (in my case, the command-line of the process will have Name=UCM_server1 in its arguments.)

```bashsession
$ ps -eo pid,command | grep '[N]ame=UCM_server1' | cut -d' ' -f1
24651
```

and now you can look at that process's environment:

```bashsession
$ cat /proc/24651/environ | tr '\0' '\n' | sort | less -S
```

You should be see that `TZ=Pacific/Auckland` is missing.

So let's set this is `setDomainEnv.sh`. Not dealing with these environments on a routine basis, I struggle to remember exactly where to find everything, but the 'locate' tool is very useful for this. You may like to run 'updatedb' first.

```bashsession
$ locate setDomainEnv.sh
/u01/oracle/domains/some_domain/bin/setDomainEnv.sh
/u01/oracle/domains/some_domain/bin/setDomainEnv.sh.orig
```

Ah, there it. Add the same content (or source it) somewhere near the beginning of that file. Restart WebLogic, and rerun your validation task (looking at the environment). You should now see that `TZ=Pacific/Auckland` in the environment for WebLogic.

And in UCM (WebCenter), the Test button for that Provider will now hopefully work.

Hope this helps someone save a couple of weeks of frustrating support calls.

## Reflection

I think that this illustrates that for a successful implementation of a complex application then it is imperative that you have a team with strong diagnostic ability with the access necessary to see what they need to see.

For example, the role of an Oracle Applications engineer often gets assigned to a [frequently reluctant] Oracle DBA; but I would argue that for a successful deployment of an Oracle application, which generally comes with a lot of issues in my experience, you need to have a team that has the access to see into the Java, OS, network layers to determine what really is going on... basically, they need root access, even if they aren't tasked with OS administration.

You can be sure that a support organisations and consultants generally won't have that level of access. An Oracle Applications engineer (I'm making that title up, if it doesn't already exist), really wants to be part Linux Systems Engineer, part Oracle DBA and part Java developer debugger/deployer/haruspex. But good luck finding one of those unless you work in a big city and can pay a high salary.
