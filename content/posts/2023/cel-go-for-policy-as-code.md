+++ 
draft = false
date = 2023-08-15T23:21:07+12:00
title = "Using Common Expression Language (CEL) in... a Metrics Exporter for LDAP"
description = "An Example of using the Common Expression Language"
summary = "Use the Common Expression Language (CEL) to allow user-friendly flexible policy rules. Show this with YAML to group/filter data to expose as metrics."
slug = ""
authors = ["Cameron Kerr"]
tags = ["Programming", "Metrics", "Go"]
categories = []
externalLink = ""
series = []
+++

> This post was previously drafted for the [Pythian blog](https://blog.pythian.com/).

As a Site Reliability Engineer, one of the kinds of problems I’m often trying to solve is how to gain essential insights into the operational health of a service. Commonly we might do this using a metrics solution such as Prometheus. When using such metrics, it’s important to consider the ‘cardinality’ of your metrics; in this post I’m going to show you how I used the Common Expression Language (CEL) within a custom exporter to allow the user to recognise different types of workloads, so we can see performance of small workloads against large workloads.

Here’s a teaser; you can see the CEL as the body of each ‘rule’

![CEL is being used to specify a policy rule in some YAML](/post-images/cel-go-for-policy-as-code/teaser.png "CEL used within YAML")

Recently I’ve had the pleasure of understanding why a client’s LDAP service was exhibiting timeouts. The CPU and Memory of the system didn’t look strained, but instead it ended up being that the worker thread-pool was too small. Increasing the CPU count on the system improved matters, but it leaves the obvious question: how can I understand the performance of the system? Is the system performance better or worse than before? How close are we to the limit?

The LDAP service in question is 389 Directory Server (previously Fedora Directory Server); the commercially supported version is Red Hat Directory Server. I’ll refer to it as 389DS from now on.

When a request (like an LDAP search operation) is made, the system will try to assign a worker thread to process it. The size of the worker thread-pool is auto-configured scaled on the number of CPU cores. If no worker-threads are available, the operation is delayed and will be attempted soon after. Thus, a request can have a small processing time, but could have a long queueing time. **It’s the queueing time we need to make visible**.

> I recently discovered that in newer version of 389 Directory Server, the access-log now has 'wtime' and 'optime'; this records the queueing time and processing time respectively, with 'etime' now being effectively the sum of those. While useful to know, it doesn't make help us in capturing metrics efficiently.

This LDAP service supports various different types of workloads, and this is important to understanding the performance of the system. Here’s a brief guide to the shapes of workload you can experience with an LDAP service; and is broadly applicable to most other client-server systems.

**Small-short** workloads are those where the client connects, performs a quick sequence of operations and then disconnects. The burden imposed by the requests is small and the connection duration is short. The statistics reported against the connection is relatively recent, but connections do leave the system quickly; we need to be careful about what we observe here because the sampling will naturally be biased to show more longer connections.

**Small-long** workloads are characterised by applications that make use of LDAP connection pooling; the connections that it opens  can stay open for hours, even days. Any such statistics are going to cover a longer period, so it’s useful to be able to separate these out.

Small workloads are much more likely to be involved with a user-facing flow; a performance issue here is going to be experienced by the user, either as a slow login, or some ungraceful timeout error.

**Large** workloads (whether long or short; I tend to just consider them all as large-long) are where we see our elephant workloads; typically things like synchronisation jobs that query for a large list of objects to populate them into a database, SaaS service, etc. These workloads are the ones you will most want to manage for the following reasons:

- They make resources less available, which are going to impact small workload.
- Even if the search would use appropriate indices, 389DS may determine that the number of intermediate results is beyond a memory threshold and will opt to revert to the equivalent of a full-table-scan.
- Many Large clients don’t use LDAP paging. Much of the time spent servicing a request is assembling the response, and so a very large response can exceed a timeout situation. With paging enabled we can provide much more immediate results without incurring such a resource burden on the server.

Before I go on, let’s revisit what actually happened so we can how this impacted the business. A service (representing a Large workload) was in the process of being upgraded, and in the interim the new version of the application was installed on new servers and running concurrently. Unknowingly, the addition of this one additional Large workload meant that Small workloads now spent even longer waiting to be assigned a worker thread. Meanwhile, client applications were experiencing time outs and the users would be presented with a bad UX and unable to access their services …. particularly bad when the service in question is an identity provider and thus is a dependency for many other important user-facing services. Hence the desire for some enhanced visibility.

## Visibility _and_ Architecture

With appropriate visibility we can see problems appearing, or reason about problems. But with good architecture we can help to prevent them, and in this case there is a direct analogue with a database architecture. After all, an LDAP service is essentially a specialised kind a client-server database system.

With databases we talk of On-Line Transaction Processing (OLTP) and On-Line Analytics Processing (OLAP); which are respectively the Small and Large workloads presented earlier. OLTP is to be kept nice and fast; OLAP is where you may perform your big, slow analytics queries. OLAP workloads often use a replica database so they don’t interfere with OLTP workloads.

A similar pattern could be used with LDAP too; have some LDAP cluster members be dedicated to Large workloads, leaving the rest that are used for everything else. You may not need to worry about this, but if you have a lot of Large workloads, then it may be worth considering. Visibility will help to make an informed decision.

## Getting the Data

We first need to consider how to obtain the data we’ll use to formulate the metrics. The [documentation for RHDS 11 (and earlier) describes the ‘connection’ attribute in the ‘cn=monitoring’ object](https://access.redhat.com/documentation/en-us/red_hat_directory_server/11/html/configuration_command_and_file_reference/core_server_configuration_reference#cnmonitor). However the version for [RHDS 12 seems to have a different format compared to earlier versions](https://access.redhat.com/documentation/en-us/red_hat_directory_server/12/html/configuration_and_schema_reference/assembly_cn-monitor_config-schema-reference-title#ref_connection_assembly_cn-monitor), but they show essentially the same information. I’m showing the older format.

There are considerations for obtaining this data, similar to polling a large SNMP table: the LDAP service will need to lock some data briefly to present consistent information, and the ‘connection’ attribute is multi-valued; you’ll get one value for each connection currently in the system. That said, I’ve found that it will return the data very quickly (<<1s) even on a busy server.

The documentation does also state that (because of the potential for service disruption if polled too frequently) that it is only provided if you bind to the service as the directory manager. I’ve yet to investigate if an ACI can be used to mitigate this.

Let’s get the data using ldapsearch to begin with.

```bash
ldapsearch -LLL \
    -h '127.0.0.1' -p 389 \
    -x -D 'cn=directory manager' -w \
    -b 'cn=monitor'
```

The output is as follows, but I'm only going to show a small part of it:

```plain
#
# LDAPv3
# base <cn=monitor> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#
# monitor
dn: cn=monitor
objectClass: top
objectClass: extensibleObject
cn: monitor
version: 389-Directory/1.3.9.1 B2019.305.1445
threads: 24
connection: 64:20230704220139Z:11:11:-:uid=service2,cn=services,dc=example,dc=com:0:0:0:551203:ip=10.1.2.3
connection: 65:20230704235921Z:3:3:-:uid=pam_ldap,cn=services,dc=example,dc=com:0:0:0:578719:ip=10.1.2.3
connection: 66:20230630035838Z:35340:35340:-:uid=service1,cn=services,dc=example,dc=com:0:1:0:90093:ip=10.1.2.3
connection: 82:20230704172423Z:2652:2652:-:uid=service1,cn=services,dc=example,dc=com:0:555:1346:535925:ip=10.1.2.3
connection: 90:20230705000029Z:7:7:-:cn=replication manager,cn=config:0:112:235:578786:ip=10.91.48.55
...
```


## Conceptualising the Metric

Within each ‘connection’ there are four fields that are useful:

- the timestamp when the connection entered the system (I’ll refer to it as ConnectionTime)
- the number of requests that have been made over the connection (OperationsReceivedCounter)
- the Bind DN (BindDN)
- the number of time a request has been blocked (had to wait) to get processed (OperationsDelayedCounter)

We can estimate service saturation by looking at the ratio of OperationsDelayedCounter to OperationsReceivedCounter. If nothing had to wait, the ratio would be very low; if the service was saturated, the ratio will be >>1. As an example, OperationsReceivedCounter might be 100, but OperationsDelayedCounter might be 400, meaning it took on average 4 times for requests on this connection to get worked on.

I would like to see this information as a histogram, so each bucket is for a given ratio. But I’d like to see this broken down by the workload type eg. small-short, small-long and large-long (large).

Here’s a before and after picture off adding CPU cores, before doing any real coding. Originally I had manipulated the output from ldapsearch using AWK… it was enough to capture a before and after, but it would be good to get something over time.

![Small/Short workload contention](/post-images/cel-go-for-policy-as-code/small-short.png "Small/Short workload contention")
![Large/Long workload contention](/post-images/cel-go-for-policy-as-code/large-long.png "Large/Long workload contention")

A small delay ratio (smaller bucket values) means that the worker thread pool was more available; higher ratios would indicate more delay in getting processed.You can see that for both Small-Short workloads and Large-Long workloads the distribution moved very much towards 0 after adding cores. The distribution moved very much to the head and the tail thinned out and shortened enormously.

Imagine for a moment that it went the other way; that performance was getting worse? How might we create an alert about that? It shouldn’t be very hard to create an expression with the intent **“alarm if less than (eg. 95)% of the small-short workload is in a bucket less-than-or-equal-to (eg. 5) on this ratio scale”**

## Classifying the Workloads: CEL-Go

The greatest predictor of something that is a large-long (given the information available in the ‘connection’ metrics) is the BindDN. This assumes you’re giving your different services their own service accounts to use to query your LDAP service. Not all services would be

Small-long workloads are recognisable by the age of their connection.

You could perhaps recognise large-short workloads by the lower age of their connection and the number of operations made on the connection. I haven’t explored that, but you could easily do so.

That leaves everything else as small-short… although there is always room for something that is known but ‘weird’… the ‘NULLDN’ Bind DN is an example of this… maybe related to monitoring or cluster operations.

So, the crux of this post is as follows:

**How can I create a suitably expressive way for the user (admin) to create expressions that can be used in a policy?**

I tried exploring this in different YAML schemas… but YAML by itself is not expressive enough; it would introduce too much coupling between the user and the developer. In other words it’s not flexible enough and the YAML schema and implementation would need to evolve too much and get too complex.

In the past, I’ve used embedded languages such as Javascript and Lua to implement custom business logic. That would certainly work here too, and would be even more customisable (eg. have it consult some other service). But that is rather more heavy-weight than we want here. It also needs to have complex sandboxing, and incurs more of a performance concern when run in the core data-path.

Enter the Common Expression Language (CEL), brought to us by Google. It’s not a complete language (not ‘Turing Complete’). Tt’s expressly designed to run in nanoseconds to microseconds, and there are implementations for multiple languages. CEL programs are additionally type-checked! CEL can be found in a number of places today:

- [Kubernetes API uses CEL](https://kubernetes.io/docs/reference/using-api/cel/) to declare validation rules, policy rules, and other constraints or conditions.
- Google Cloud Certificate Authority Service, Envoy, the Open Policy Agent and more
- CEL is a great tool for allowing an API (or human) user to specify filter conditions (eg. when requesting a set of API resources)

I liken CEL to be similar to (but a bit heavier than) a regular expression. You often use a regular expression in your data-path; it has a performance cost that you don’t tend to worry about too much, so long as you preprepare them. CEL programs, like regular expressions, really should be compiled and checked once, such as when you start your program, and then run many times (such as for every request).

I didn’t know CEL previously, but I learned CEL for Go using the [CEL-Go Codelab](https://codelabs.developers.google.com/codelabs/cel-go/index.html#0). It’s free. I should mention too that it’s been a while since I used Go, so I was relearning that at the same time. It was a fun few days.

CEL has implementations for a few languages: Go, C++ and Java would be the best supported, because they are developed and used by Google. Other implementations exist for Python, Rust, Ruby and more, in varying stages of completeness. Tip: if you want to search for your favourite language, try searching for ‘TypeScript common expression language “cel”‘ (substitute TypeScirpt for the language in question). Searching for just ‘TypeScript “cel”‘ is not the happy path.

## Code Highlights

I don’t want this post to explain too much in terms of the code, so I’ll focus just on the CEL specifics, and leave anything else on the ‘cutting room floor’.

Unfortunately I can't share the full code with you, as it is internal.

When reading the following code, remember that a Classifier is one of the classifiers in this YAML, and that it contains a workload and a rule.

```yaml
classifiers:
  - workload: "large-long"
    rule: |
      bind_dn in [
        "uid=atlassian,cn=service_accounts,dc=example,dc=com",
        "uid=print_management,cn=service_accounts,dc=example,dc=com"
      ]
  - workload: "small-long"
    rule: connection_age_seconds > 120
  - workload: "unknown"
    rule: bind_dn == "NULLDN"
  - workload: "small-short"
    rule: true
```

After we read in the YAML containing the workload definitions, at startup, we then do the CEL phases:

- Creating an execution environment which will represent the CEL program, its runtime options, functions, variables and type information we add, and more.
- Parse the CEL program
- Type-Check the parsed program
- Check that the CEL program returns the type we expect (eg. a Boolean)
- Obtain the compiled program, ready to use it many times; we could even persist it.

```go
func CompileConnectionClassifiers(classifiers *[]Classifier) error {
    for classifier_index, classifier := range *classifiers {
        // TODO nowhere are we reporting in an error which actual rule we're failing on
        // Prepare the execution environment
        env, err := cel.NewEnv(
            // I'd prefer to use an ObjectType, but that seems to assume ProtoBufs?... maybe use a MapType instead?
            cel.Variable("bind_dn", cel.StringType),
            cel.Variable("connection_age_seconds", cel.IntType),
        )
        if err != nil {
            return err
        }
        // Parse the program
        ast, issues := env.Parse(classifier.Rule)
        if issues.Err() != nil {
            return issues.Err()
        }
        // Type-check
        checked, issues := env.Check(ast)
        if issues.Err() != nil {
            return issues.Err()
        }
        // Check return type of program is boolean
        if !reflect.DeepEqual(checked.OutputType(), cel.BoolType) {
            return fmt.Errorf("return type of rule #%d %q is not %v and not %v",
                classifier_index+1, classifier.Rule, checked.OutputType(), cel.BoolType)
        }
        program, err := env.Program(checked)
        if err != nil {
            return err
        }
        // 'classifier' is pass-by-copy
        (*classifiers)[classifier_index].ruleProgram = program
    }
    return nil
}
```

When we go to actually run the program, we Evaluate it, passing it the required arguments that will be bound to the variables we’ve declared when we created the environment.

```go
func EvaluateConnectionClassifiers(classifiers *[]Classifier, connection pec.Connection) (string, error) {
    for classifier_index, classifier := range *classifiers {
        result, _, err := classifier.ruleProgram.Eval(
            map[string]interface{}{
                "bind_dn": connection.BindDN,
                "connection_age_seconds": time.Now().Unix() - connection.ConnectionTime.Unix(),
            },
        )
        if err != nil {
            return "unknown", fmt.Errorf("classification rule #%d with rule %q failed: %v",
                classifier_index, classifier.Rule, err)
        }
        rule_matched := result.Value().(bool)
        if rule_matched {
            return classifier.Workload, nil
        }
    }
    return "unknown", nil
}
```

Here’s part of the unit tests I created which shows how they relate. I haven’t shown the Connection related structures, which contain the parsed values of the ‘connection’ LDAP attributes; neither have I shown the ClassifiersDoc, which is just the basic structure representing the YAML.

```go
package classifier

import (
	pec "port389ds-exporter/internal/connection"
	"testing"
	"time"
)

var workload_weight_duration_classification_yaml = `
    classifiers:
    - workload: "large-long"
        rule: |
        bind_dn in [
            "uid=atlassian,cn=service_accounts,dc=example,dc=com",
            "uid=print_management,cn=service_accounts,dc=example,dc=com"
        ]
    - workload: "small-long"
        rule: connection_age_seconds > 120
    - workload: "unknown"
        rule: bind_dn == "NULLDN"
    - workload: "small-short"
        rule: true
    `

var workload_weight_duration_classification_doc *ClassifiersDoc

// This show preparing the CEL programs

func setupWorkloadClassifier(t *testing.T) {
	var err error

	workload_weight_duration_classification_doc, err = LoadClassifiersFromYaml(
		workload_weight_duration_classification_yaml)

	if err != nil {
		t.Errorf("failed to load yaml: %v", err)
	}
}

func TestClassificationLargeLong(t *testing.T) {
	
    if workload_weight_duration_classification_doc == nil {
		setupWorkloadClassifier(t)
	}

    // sets up some dummy data that would be read in from LDAP 'connection' attribute
	connection := pec.NewConnection()
	connection.BindDN = "uid=atlassian,cn=service_accounts,dc=example,dc=com"
	connection.ConnectionTime = time.Now().Add(-time.Duration(1) * time.Hour)

    // just gets the list of CEL programs, one for each rule
	classifiers := &workload_weight_duration_classification_doc.Classifiers

    // this is where the CEL programs (the classifiers) are run
	workload, err := EvaluateConnectionClassifiers(classifiers, connection)
	
    if err != nil {
		t.Errorf("failed to classify: %v", err)
	}
	if workload != "large-long" {
		t.Errorf("expected large-long, but got %v", workload)
	}
}
```

My intent here is not to teach you CEL in this blog-post, but instead to encourage you to try the [CEL-Go Codelab](https://codelabs.developers.google.com/codelabs/cel-go/index.html#0). It’s free, and it’s a fun weekend project.

Can you think of any use-cases where your user’s would benefit from being able to specify policy as code?
