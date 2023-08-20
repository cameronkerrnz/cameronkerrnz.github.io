+++ 
draft = false
date = 2023-08-15T21:54:45+12:00
title = "Reimplementation of Learning :: a Python to Go Retrospective"
description = "Encouraging System Admins / Engineers to Improve their Programming"
summary = "PoC coding and reimplementation of Python and Go for metrics and log parsing using an example of 389 Directory Service."
slug = "A Python to Go Retrospective"
authors = ["Cameron Kerr"]
tags = ["Retrospective", "Programming", "Growth", "Python", "Go"]
categories = []
externalLink = ""
series = []
+++

> This post was previously drafted for the [Pythian blog](https://blog.pythian.com/).

This post is written for those engineers who are rising to the challenge of being an SRE, to inspire you and to give you confidence to try out new ways of solving problems. In my experience it is relatively easy to find Systems Engineer / Systems Administrator etc. with some skill in Python. Python is a wonderful language which I’ve made a lot of use of in the past.

I love learning and creative problem solving; I consider these primary traits for success as an SRE and many other parts of the IT industry. As an SRE, distinct from other roles with ‘Developer’ in their job title, most of the programming I have done can be considered tooling and integration. This is great if you consider yourself a ‘polyglot’ programmer (someone who will use/learn many languages).

In this post, I hope to inspire you to try using Go for a small project, to give you some confidence to try, and perhaps even set some expectations. However, this is not a post about programming… you won’t even see a single line of code. Hopefully you’ll come away with an understanding of why/when this can be a good idea, how much time it took, the breakdown of that time as relates to someone who is a competent programmer, but is still relatively new to Go, and the qualitative outcomes.

There are actually two small related projects I want to discuss here, but before I introduce them I should give you some context so you can appreciate the problems they are trying to solve, and thus the value they might provide.

## The Problem

One of my clients uses an LDAP directory service called 389 Directory Server. You may have heard of this by its previous name of Fedora Directory Server, or its commercially-supported equivalent of Red Hat Directory Server. LDAP servers play a critical role in the Enterprise, and when you have problems with LDAP you can expect problems elsewhere too, such as users not being able to login… to anything.

This is the problem I was given to solve. Because of the load on the service from various LDAP synchronization jobs, which would make a lot of heavy queries on the service, it was making the service less available for other queries. In other words, the service was too saturated.

As is often the case when an SRE is first given such a problem, there are a few broad tasks. In no particular order, they are:

- Look for appropriate metrics to gain missing visibility
  - This will give us some objective information for before/after changes
- Understand the mechanics that relate to the symptoms
  - In this case, we consulted the performance tuning guide to learn more about how requests are processed by a pool of worker-threads,  how this - worker-pool is sized, and how we could measure how often a request had to wait before it could be serviced.
- Understand what workloads are present in the system
  - This will help us determine who/what is most affected, and who/what might be causing the problem, and thus help us design an effective - remediation / mitigation plan, as well as tell us what nuances our monitoring strategy should try to capture.
  - This level of detail ceases to be ‘metrics’ and is instead an exploration of the access log.

For these retrospectives, I’ll try and break down the time as follows:

- Time spent learning the language, its common libraries and its tooling (reflecting junior–intermediate developer upskilling)
- Time spent researching problem-specifics (time an experienced developer would still need to spend)
- Time spent solving for the functional (what it needs to do) and nonfunctional requirements (eg. performance)

Since I can’t show you the code, I thought I’d try using `scc` (the [Succinct Code Counter](https://github.com/boyter/scc)) to show you how much code was involved. I haven’t included the COCOMO estimations that tool produces, because I have no confidence in its output… it does make me look like a 10x programmer though 🙂.

By doing such retrospectives, developers can close the feedback loop for estimating future work, and this is a habit I'm hoping to foster in myself.

## Access-Log (part 1: Python)

> ‘Started work on a performance-improvement job for [client], but decided it would be time well spent developing a script to get 389 Directory Server (LDAP) access logs parsed into a form whereby I could do meaningful things with the data, like ask what type of queries are the slowest due to being unindexed.’

I started this exploration with the access log. LDAP access logs are extremely busy, and can be hard to work with both because of their size and because the request is logged separately from the response which is logged separately from the authentication.

Other issues are that the query (LDAP filter) is not aggregatable. I’d very much like to aggregate similar queries together. This enrichment activity is something that can provide a lot of value. Example: I can’t do something like a GROUP BY with these two lines; they are different:

```plain
(&(objectClass=inetOrgPerson)(|(cn=ckerr)(uid=ckerr)(mail=ckerr))
(&(objectClass=inetOrgPerson)(|(cn=jsmith)(uid=jsmith)(mail=jsmith))
```

But if I can generalize it to show the query structure and attributes (and type of indices) used, then I can certainly do a GROUP BY on this, and thus get some useful aggregate performance information:

```plain
(&(objectClass=inetOrgPerson)(|(cn=...)(uid=...)(mail=...))
```

So to get the data in a form that useful to understand the workloads, we need to do a few things with regard to the access-log:

- Parse the access log
- Join the requests with their responses for each operation
- Track what user (‘Bind DN’) was active at each time
- Denormalise the data where appropriate
- Enrich the data to make filtering and aggregation convenient
- Output (or store) the data as appropriate

To prevent scope creep, I avoided:

- Running this as a service
- Tailing files
- Integration with any log processing pipeline (although as it - produces JSON, this is nicely reusable)
- Filtering (instead I showed an example of filtering in the documentation using jq)

Since I needed to solve for production data, a script was written which would collect the recent logs from the various production LDAP servers so I could process this snapshot of the logs on the jump-box rather than burdening the LDAP servers with my processing.

This code started out as Python. As I was writing this code, I was mostly concerned with:

- Iterating the parser development so I could at least capture the data I needed and get to the end of the 10GB of data I had available. I allowed my program to crash with an exception when it failed to parse a line.
- How to parse the LDAP filter and make a generalized version of it.
- Making it easy to carry the code between my development environment and where the data lived — meaning that it was one Python file.

It worked, but it was much too slow. But I did get some of the insights I needed. For future use, there was clearly a performance issue that needed to be dealt with. I didn’t learn any new Python skills, so this was a competent Python programmer doing what could be considered some routine type of development work. Let’s have a look at how the time was spent.

```
───────────────────────────────────────────────────────────────────────────────
Language                 Files     Lines   Blanks  Comments     Code Complexity
───────────────────────────────────────────────────────────────────────────────
Markdown                     1       133       36         0       97          0
Plain Text                   1         1        0         0        1          0
Python                       1       334        2        10      322          0
Shell                        1        11        2         2        7          1
───────────────────────────────────────────────────────────────────────────────
Total                        4       479       40        12      427          1
───────────────────────────────────────────────────────────────────────────────
```

| Type of activity                                            | Hours |
|-------------------------------------------------------------|-------|
| Learning the language and its tooling                       |     1 |
| Researching problem-specifics                               |     4 |
| Implementing functional and non-functional requirements     |    10 |

In summary: an MVP (Minimum Viable Product) that delivered a result but was too slow in order to deliver value through reuse.

## Metrics (Go)

I had the insight I needed from the access-logs, and needed to move onto the metrics. My initial PoC was essentially ldap-search and AWK, which delivered me the insights I needed to come up with a recommendation for the client, backed up by some useful data. But it still had some manual steps and wouldn’t be useful for seeing metrics over time.

It has been my common experience that most of the really interesting metrics to collect are custom metrics… because you’ve identified something (commonly because of a problem in production) that you’ve learned you need to monitor. Beyond basic CPU, memory and disk monitoring; if you’re not monitoring usage of pooled resources such as database connection pools, worker thread pools, etc. then you’re likely missing essential metrics that are useful to expose and monitor.

The client did have a platform that allowed for metrics, but wasn’t getting much value from it. With the client’s consent, I endeavored to deliver these metrics to their existing platform as a demonstration of its value.

When it comes to metrics, my default disposition is generally Prometheus, but that’s not what the client uses. Instead, they have Metricbeat feeding into Elastic. Metricbeat is geared very much to vendor-provided-value and doesn’t have a compelling user-experience for custom metrics. However, it does have support for collecting (‘scraping’) metrics from a Prometheus exporter, and that does have a nice user-experience for custom metrics. It also meant that if the client decided to move to a different platform there should be less risk of wasted value.

Within the Prometheus eco-system the Go language is a first-class citizen; so as long as Go can talk to LDAP (it can) it should be easy to create a suitable exporter, so that’s what I did.

This would be my second project I’ve done in Go. The previous time being circa 2017; about Go 1.9. The requirements start out fairly simple:

- Read some configuration from YAML
- Connect over LDAP
- *Query for the particular performance-diagnostic objects*
- *Parse the list of data that comes back*
- Categorize the workload according to the client service (the Bind DN) making the query
- (Use Prometheus libraries to) calculate histogram metrics for each type of workload
- Create an HTTP listener and have Prometheus client library respond to the /metrics endpoint
- Add another endpoint for diagnostics, root page (/) etc.
- Dashboard resources to visualize the data in Kibana

Not stated above are some other realities that we need to consider:

- Suitable unit testing and performance testing to ensure fit for production
- Deployment resources (eg. SystemD service, configuration management)
- Development environment (ie. Docker compose configuration that sets up same version of Metricbeat, Elasticsearch and Kibana)
- Documentation

But more interesting was how to make it reusable. The metrics I was interested in are useful to categorize into small/quick type of queries, and large/slow type of queries (I’m simplifying, but hopefully you get the idea). The rules are not useful to generalize in code, but it turned out that business-rules could be expressed largely in terms of the Bind DN (client service) and the duration of the connection etc.

Rather than try and force some YAML schema on my users, I looked to see how I could use the Common Expression Language (CEL) to make my configuration much more flexible. I had learned of CEL in a book I had recently been reading about API Design Patterns, and I knew it used in various places where policy-as-code was in vogue. Completing Google’s CEL codelab, I then took what I had learned and ended up with something much more reusable and user-friendly.

One additional issue though… one that I had anticipated during my research: although current versions of Elasticsearch do support a Histogram data-type, my client’s deployed environment did not, so I had to come up with a workaround by outputting multiple gauge metrics… one for each performance bucket in the histogram. A bit ugly — I called it a ‘histocrutch’ —  but much less effort than asking the client to upgrade a platform they weren’t sure they wanted to keep.

So let’s see how a rusty Go novice got on with this:

| Type of activity                                            | Hours |
|-------------------------------------------------------------|-------|
| Learning the language and its tooling                       |     3 |
| Researching problem-specifics                               |     7 |
| Implementing functional and non-functional requirements     |     8 |

It’s hard to trust the numbers in terms of how much time is spent re-learning a language, because I often find myself looking up basic things… as a polyglot, it’s common to need a refresher on some basic language constructs, such as how to do a particular style of iteration, or how to use common libraries such as for regular expressions or date/time handling. Some of that time is more conscious and often frustrating, such as how to properly lay out your project files to keep packages/modules etc. happy; and other times such lookups are fast and don't kick you out of your flow state.

**I can pleasantly report that with rich editor integration (I used VSCode), the type system, code completion, unit-testing and debugger integration made the language very easy to (re)learn, refactor and proceed with confidence.**

Certainly if I were to do this same work in a (default configuration of) vim my results would be very different. This is why I wanted to keep my development environment separated from where the data was constrained to live… although use of SSH port forwarding was useful.

I wanted a break-down showing how much of this was test code:

    ───────────────────────────────────────────────────────────────────────────────
    Language                 Files     Lines   Blanks  Comments     Code Complexity
    ───────────────────────────────────────────────────────────────────────────────
    Go                           9       909      185        38      686        155
    ───────────────────────────────────────────────────────────────────────────────
    ~xporter/cmd/389ds-exporter.go       352       54        15      283         61
    ~rnal/classifier/classifier.go       109       25         8       76         18
    ~classifier/classifier_test.go       133       33         2       98         26
    ~rnal/connection/connection.go        60       14        13       33          0
    ~connection/connection_test.go        15        4         0       11          2
    ~internal/connection/parser.go        69       16         0       53         21
    ~nal/connection/parser_test.go        30        5         0       25          2
    ~orter/internal/source/ldap.go        80       14         0       66         19
    ~/internal/source/ldap_test.go        61       20         0       41          6
    ───────────────────────────────────────────────────────────────────────────────

Even as a relatively novice Go programmer I think this was a highly productive project; one that doesn’t cut corners unnecessarily and makes use of good development practices. I’m proud to say I worked on this. Probably the most frustrating part was setting up the initial project structure (packages, modules, where to put test files), but once that was solved it was fun and exciting work.

## Access Log (part 2: Python refactoring and improvement)

My purpose in this task was to unlock future value by making the tool reusable… which largely means addressing the performance issues. I already knew that I lost much performance when I introduced the code to generalize the LDAP filter, so that was my first thing to fix. After realizing that it was actually a simpler problem than I first expected, I ended up throwing out the third-party library I had been using (and monkey-patched, yuck) and replaced it with a regular expression based solution that used a function to generate the replacement text.

But I was also intent on improving some of the software engineering aspects. As such, there was significant refactoring to make the code cleaner, introducing Python type hinting, refactoring stateless and stateful code, creation of unit tests, use of coverage tests, etc.

The refactoring went well, largely because I put the effort into making the code more testable and employing unit-tests. Looking back on the time I logged, I seem to have enjoyed it:

> ‘Having a generally wonderful and productive time refactoring some log processing code, adding unit tests, learning how to do coverage tests and resolving a bug.’

Most of the credit for the pleasant refactoring experience must again go to my IDE (in my case VSCode) and the various and deep integrations that it makes available, most notably Pylance. 

Once the obvious refactoring and testability concerns were covered, profiling suggested strongly to separate parsing and enrichment into separate threads… which in Python really means separate processes (multiprocessing versus multithreading). I wanted to be able to scale this out so that log-lines relating to the same connection would be processed by the same handler, so I needed to do some research and make a bit of a PoC to improve my understanding of this. I’d used the multiprocessing library before, although it didn’t seem to have anything that nicely mapped to my multiple-workers-keyed-with-a-hash type of use-case.

I don’t mind saying I had a frustrating experience trying to implement something that would have been reasonably simple in languages that feature message-passing, such as Erlang or Go.

> ‘Having a very annoying time trying to get a not-quite-so-basic Python 'multiprocessing' multi-Process/multi-Queue combination to work correctly without deadlocking.’

> ‘Decided Python was not a good tool for the LDAP access-log parser, which needs higher performance than what I'm getting with Python. And given the painful architecture that Python multiprocessing is forcing on me, with this particular job I'd have a better time with Go.’

I should at this stage mention that I’ve done a similar project in Go: reassembling high-volume log fragments into complete structures and doing some parsing and enrichment before emitting  it as JSON. That was my first experience using Go, and it was circa 2017… about Go version 1.9. In that experience I was using Go’s coroutines with message passing and it was an entirely positive experience, delivering production value with minimal maintenance over its history. So I was fairly confident that a reimplementation in Golang would be similarly successful.

By the end of it, here’s what the code metrics suggest; the story behind this is essentially restructuring into a module, replacing a library with some of my own code, adding unit tests, and then a PoC to see how to make a multiprocess workqueue with the hashing behavior I wanted.

    ───────────────────────────────────────────────────────────────────────────────
    Language                 Files     Lines   Blanks  Comments     Code Complexity
    ───────────────────────────────────────────────────────────────────────────────
    Python                       7       627       82        26      519         37
    ───────────────────────────────────────────────────────────────────────────────
    ~p-access/389ds-access-to-json        32       11         2       19          5
    ~/port389ds_access/__init__.py         0        0         0        0          0
    ~s_access/access_log_parser.py       299        9         0      290          5
    ~ss/ldap_filter_fingerprint.py        52        1         0       51          0
    ~ss/port389ds_access/mqueue.py       116       35        23       58         16  (PoC)
    ~ess/test_access_log_parser.py        86       16         0       70          9
    ~st_ldap_filter_fingerprint.py        42       10         1       31          2
    ───────────────────────────────────────────────────────────────────────────────

| Type of activity                                            | Hours |
|-------------------------------------------------------------|-------|
| Learning the language and its tooling                       |     4 |
| Researching problem-specifics                               |     1 |
| Implementing functional and non-functional requirements     |     3 |

At the end of the day, the decision to reimplement was based on a few factors:

- Certainty of experience. I knew Go was entirely capable of what I wanted, because I had done something very similar previously.
- Performance (throughput) has a higher raw CPU throughput than (pure) Python and has a less constrained threading model. Because it has higher performance.
- As a CLI tool (not a service), a single binary produced by Go is more readily deployed compared to Python + suitable virtual environment + code.

In summary, this was essentially wasted effort, but much was learned during the process and my Python software engineering skills improved markedly; the opportunity to develop a testing habit and to make greater use of Python’s type hinting (and it’s integration with VSCode via Pylance) was valuable training for me.

## Access Log (part 3: Reimplementation in Go)

My third-ever serious Go program, and I was reimplementing something I had written in Python, so the logic and regular expressions were mostly resuable. Go gives me some good capabilities as well as problems to solve.

Much of the heavy lifting was done by a few libraries for regular expressions, date/time parsing, JSON encoding and unit-testing. The rest was coded by myself. Only a small part seemed to be ‘batteries not included’ and that was in the regular expression support, where I needed to find some code that would implement a “replace pattern by output of function” type of replacement.

The first major design challenge I needed to overcome was data-structures. Go is a typed language, and I’m intending to do this in a type-safe manner so I can make the most use of the compiler and the ‘red squigglies’ in the editor integration. The biggest annoyance I found there was the lack of something like a type-safe union (or sum type)... I wanted to say  a Request is either a RequestBind or a RequestSearch, etc. Go doesn’t have objects and it doesn’t have inheritance… it has generics and ‘struct embedding’, and the ‘any’ type. I did spend some time trying to determine the most Go-like way to approach this; this is likely where a good book on Go would be useful. I ended with a solution I was still happy with and satisfied my desire for type safety; although I did try  a few different dead-end paths first.

(Naturally, I was also wondering what the experience would be like in Rust… another language I have done some work with, but I digress…)

The other major annoyance was Go’s peculiar way of dealing with date and time formatting/parsing. Sometimes I think Go just wants to try and do things differently… and results in some odd syntax at times. But we work to adjust ourselves to the reality of the ecosystem in which we work, and not the other way around.

I started this project with the goal of embracing Test Driven Development; I think that went very well. I refactored the code several times throughout the process, and the combination of compiler integration and unit-tests did a great job in highlighting all the places I needed to change. VSCode even helpfully highlighted the code coverage results after I ran the unit-tests for the package. Being able to debug a particular unit-test with a single click was also highly useful. When I found a problem I would write a test case for it, see it fail, fix it, see it pass (and hopefully not break other things). To quantify how successful my goal was, the last thing I wrote was the command-line application that took the command-line input, read the files and fed that data to the parser.

Did it catch all the bugs? No it did not; once it was given 10GB of real-world log data to work on, it did crash in a couple of places. Thankfully the panic messages pointed me to the location in the code (code coverage showed that branch was not tested). Write a test for it, fix the bug, see the test succeed, compile, upload, test with real data and repeat until happy. The program was mildly multi-threaded; there was only one issue relating to that due to some messages that pointed to shared data that ended up being deleted by one thread by the time it was consumed by another… a good reminder that it is still possible to have race conditions even when you are communicating by passing messages.

**It was satisfying to find that bug, write a test for it, and then see the bug squashed when I fixed it. It was even more satisfying that it only took 3h to fix a race-condition bug.**

A seasoned developer would (should) find that an obvious statement, but it’s my common experience to find people coming from life as a Systems Administrator or a Systems Engineer who haven’t had that culture of testing embossed into them. To many, it’s a different way of thinking that doesn’t come naturally but is insteast thought of as an overhead.

Part of the desire for reimplementation was relating to performance. I’m happy to say that it took a process that would take tens of minutes (typically resulting in a ^C) and made it complete in about 3 minutes. That’s without doing any fancy to improve performance. I’m not going to spend additional effort to make it faster until it’s needed; there are other aspects that would be useful to work on first, such as using the output in some pre-canned reports.

    ───────────────────────────────────────────────────────────────────────────────
    Language                 Files     Lines   Blanks  Comments     Code Complexity
    ───────────────────────────────────────────────────────────────────────────────
    Go                          11      2400      369        86     1945        185
    ───────────────────────────────────────────────────────────────────────────────
    ~esslog/cmd/389ds-accesslog.go        90       18         2       70         18
    ~l/389ds/accesslog/emission.go       134       22        20       92         14
    ~ds/accesslog/emission_test.go       158       31         1      126         16
    ~ccesslog/operation_parsers.go       442       26         2      414         18
    ~log/operation_parsers_test.go       144       40         0      104          0
    ~/accesslog/operation_types.go        63        6         2       55          1
    ~nal/389ds/accesslog/parser.go       380       89        27      264         73
    ~89ds/accesslog/parser_test.go       704       92        10      602         11
    ~rnal/389ds/accesslog/types.go       107       15         8       84          0
    ~lter/generalise/generalise.go        85       11        14       60         16
    ~generalise/generalise_test.go        93       19         0       74         18
    ───────────────────────────────────────────────────────────────────────────────

Some of the testing files include multi-line JSON strings and this will inflate the line count somewhat… it would be better if I had explored some better ways of testing the produced JSON documentations.

| Type of activity                                            | Hours |
|-------------------------------------------------------------|-------|
| Learning the language and its tooling                       |     9 |
| Researching problem-specifics                               |     3 |
| Implementing functional and non-functional requirements     |    22 |

34h; let’s say a 1.5 person weeks. My own observations was that productivity in using Go increased quickly at about the half-way point with no further time needed to learn the language  common libraries and tooling.

## Retrospective

Was it worth it? I suppose that might depend on who you ask. The metrics work was the part the client asked for, and I think that will provide a lot of confidence around how the service is managed in future.

The access-log processing was partly for training and development, by taking learned knowledge from one person and crystallizing it into tooling to provide future value to this and other clients.. As such its value is less tangible at the moment; but it’s helped to improve my own skills, and will help to improve those of my team, current and future clients… and now hopefully you, dear reader.

There was certainly some wasted effort, but in the spirit of lean development, I think the decision to reimplement was made at a useful time; the skills and practices learned were at least still useful and will be utilized in the future.

What is worth more comparison however is the initial Python access-log parser and the Go reimplementation. 15 hours versus 34 hours. Certainly the Python version was faster to develop and iterate on, but exhibited much less software engineering rigor. Python is good for this exploratory phase… and if performance allowed it would have stayed as Python. **I think the biggest lesson here is to be careful when moving on from that exploratory phase. Before you start refactoring, make sure you’re _still_ using the right tool for the job.**

I want to leave you with a video I watched recently by an Engineer who has a very well presented YouTube channel. In one video he looks at the question of what an Engineer means by ‘best’. The notion of 'best' is a very relevant term here.

{{< youtube p8IO9u9IuOs >}}

