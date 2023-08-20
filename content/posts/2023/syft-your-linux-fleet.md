+++ 
draft = false
date = 2023-06-19
title = "Syft your Linux Fleet Looking for Shellshock, Log4J.... and Oracle Java?"
description = ""
summary = "I explore using Syft and Grype, demonstrate use of jq and Go's text templates, and explore using Syft against a server rather than a container."
slug = ""
authors = []
tags = ["SBoM", "Compliance", "Security", "Syft", "Grype"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published on the [Pythian blog](https://blog.pythian.com/).

What do ShellShock, Log4J, and Oracle Java have in common? ShellShock and Log4J are similar in that they represent vulnerable dependencies that were widely **embedded** and caused IT teams to invest effort in scouring Servers, Appliances, Containers, and even Desktop fleets to find and mitigate the threat they represented. Oracle Java represents a different driver for a similar kind of activity: licence compliance. Management needs confidence that software risk is being attended to.

Recent years have seen a lot of action in terms of the Software Bill of Materials (SBoM). In essence, if you can’t fix what you can’t see, put a lot more effort into improving visibility. When you have a thorough understanding of the entirety of your software dependencies, you have a much easier job finding where you have code that is running with known (and fixed) vulnerabilities.

That’s great when you have a CI/CD pipeline that churns out new versions of software to push to a container image registry, and that’s where a lot of tooling effort has been placed. But this tooling is just plumbing to assemble parts of a filesystem together and then have a deep look inside the filesystem. So scanning something like a server’s filesystem should be conceptually easier. Let’s test that assumption, though, as Syft is not really designed for this use-case. Playing with the tool and exploring its limitations is still a good opportunity.

Obviously, there is a lot here that has already been done. Red Hat Satellite will tell you when packages need to be updated. Vulnerability management tools that scan the network will tell you that you have vulnerable versions found. None of that is new. But these kinds of tools have some inherent limitations, namely:

- **They are limited in their scope**. They might only look at the RPM packages installed or similar. They often won’t find instances of Pythian virtual environments, packages installed via Node Package Manager (NPM), Ruby gems, Java dependencies encapsulated within Java Archives (jars), local container images, etc. These are arguably the more important things to find because they are more likely to form part of an attack surface, and they are not going to be updated when you do your regular OS patching.
- **They often assume software version strings**, leading to many false positives. This is particularly true when using an enterprise distribution of Linux that will backport security fixes. If Nessus scans a web server and finds that it’s using Apache httpd version 2.4.8, it’s going to report that this is vulnerable. Still, it doesn’t know that Red Hat has been backporting all the various security updates so that its customers don’t suffer a breaking change in functionality. To help fix this lack of external-facing information, Nessus can also have an agent running on the inside.

Newer SBoM tools focus mostly on the first objection. The second objection becomes much less of a concern because they operate with richer data sets about vulnerabilities and what’s installed.

There are multiple tools out there today. The tool I want to introduce to you today is Anchore’s Syft and Grype CLI tools, which are Open Source. Anchore also has an Enterprise offering that collects the SBoM and makes it easier to see and ask questions about your software estate at scale. But this post will focus on using just Syft and Grype, and our reporting will go far enough to show the value and let you imagine what comes next.

Syft sounds like ‘sift’: examine (something) thoroughly so as to isolate that which is most important. Grype sounds like ‘gripe’: to complain. Syft is the tool that creates a Software Bill of Materials (SBoM), and Grype is the tool that compares the artefacts listed in the SBoM against vulnerability databases to tell you which artefacts have listed vulnerabilities.

If we were to run this over a fleet of servers, Syft is the tool you would run on each server, and Grype you could run elsewhere, particularly if your servers had limited access to the internet. Anchor’s Enterprise offering helps to join these tools, having a place to collect and present the information. You’ll see I end up doing some work with ‘jq’ to simplify the data.

Find a non-production server, and let’s play. Try to choose something that has software installed from multiple sources and mechanisms. The system I’m working with has software installed from Yum, Python Pip, Go binaries I’ve downloaded off GitHub (multiple versions of Terraform, Kubernetes, etc.), and more besides. If you don’t have a server handy, you could launch a container instead. Normally with Syft you point it at a container image rather than run it inside the container you’re inspecting. We just need to scan a filesystem, so running it within a container this way is a good analog of a server for our needs, with the notable difference in size and, therefore, time-to-scan). Real servers tend to have more history too.

## Creating a Software Bill of Materials (SBoM)

Download Syft. Releases are made frequently. https://github.com/anchore/syft/releases. Normally I get mine using ‘brew’, but that’s on macOS. Anchore do make use of GitHub Actions package signing for their releases. The ‘Recommended’ way to install it is as follows:

```bash
$ curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
```

Let’s do an initial scan; it won’t take long, but we take the precaution of exempting some parts of the filesystem, such as dynamic filesystems. It would be useful to exempt network shares and filesystems that don’t contain software (eg. database storage).

```bashsession
$ syft packages dir:/ --exclude ./dev --exclude ./proc --exclude ./sys
   Indexed /
   Cataloged packages      [2202 packages]
NAME                            VERSION           TYPE
./api                           (devel)           go-module
./api/auth/kubernetes           (devel)           go-module
...
@discoveryjs/json-ext           0.5.6             npm
@types/eslint                   8.2.1             npm
...
Deprecated                      1.2.13            python
Flask                           2.2.5             python
...
adduser                         3.118ubuntu5      deb
adwaita-icon-theme              41.0-1ubuntu1     deb
...
aws-lambda-java-core            1.2.0             java-archive
...
aws-sam-cli                     1.84.0            python
aws-sam-translator              1.66.0            python
azure-appconfiguration          1.1.1             python
azure-batch                     13.0.0            python
azure-cli                       2.48.1            python
...
github.com/pierrec/lz4/v4       v4.1.15           go-module
github.com/pierrec/lz4/v4       v4.1.8            go-module
...
go                              1.19.3            binary
...
node                            20.2.0            binary
...
```

Wow! Look at the **number of distinct software ecosystems** that it has identified. You can see the full list of supported ecosystems at [https://github.com/anchore/syft](https://github.com/anchore/syft). There will always be room for more, but they even support Erlang and Haskell… and Jenkins plugins (!?). One notable omission: Perl (CPAN).

We short-changed ourselves, though: we let Syft output in the default table format, which only shows a small fraction of the data it collects. Syft can output in many different formats, and within the evolving SBoM space, there are at least a couple of different competing / complementary formats: CycloneDX and SPDX… both of which have JSON and XML variants. Thankfully, we can just output in Syft’s own lossless format (syft-json), and then convert to other formats as needed.

```bashsession
$ syft packages dir:/ \
>   --exclude ./dev --exclude ./proc --exclude ./sys \
>   --name your-hostname-here \
>   --output syft-json --file your-hostname-here.syft.json
   Indexed /
   Cataloged packages      [2202 packages]

$ jq '.descriptor.configuration.name' your-hostname-here.syft.json
"your-hostname-here"

$ syft convert your-hostname-here.syft.json -o syft-table
[0000]  WARN convert is an experimental feature, run `syft convert -h` for help
NAME                           VERSION                    TYPE
./api                          (devel)                    go-module
...
zlib1g-dev                     1:1.2.11.dfsg-2ubuntu9.2   deb
```

I’ve given it a name; you could imagine this being the hostname. The actual hostname isn’t captured in the file, not generally being very useful in a container. It gets tucked away in the ‘.descriptor.configuration.name’ path within the JSON. You could imagine we take that JSON and copy it somewhere else for inspection, perhaps even analysed on a routine basis (as would be the case for a Container Image Registry). While the SBoM only needs to be updated when the software has changed, in a fleet context, you’d probably just run this on a schedule; some software will auto-update, plus, there is no clear signal to say “new software has been deployed.” Remember that the SBoM Syft has generated a recording of which artefacts have been found, not about vulnerabilities that have been found against those artefacts. That’s Grype’s job, and we look at this next.

Compare the SBoM against Vulnerability Databases
Installation is much the same as with Syft:

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
```

It should be noted that Grype can also do what Syft does, so we technically don’t need to run Syft ourselves, but it’s useful to do so because it decouples scanning from analysis. You might also want to run Grype multiple times to get different types of reports, and you don’t want to run Syft multiple times, as it’s necessarily a slow kind of process. Grype will also download vulnerability database definitions and cache them, so you don’t want all your servers to have to reach out to the internet.

Let’s get started, using our SBoM as input. Nice and easy, but you’ll likely find a lot of output which will make it challenging to manually filter.

```bashsession
$ grype sbom:your-hostname-here.syft.json
  Vulnerability DB        [updated]
  Scanning image...       [629 vulnerabilities]
    3 critical, 80 high, 301 medium, 213 low, 32 negligible
    92 fixed
NAME                            INSTALLED              FIXED-IN         TYPE       VULNERABILITY        SEVERITY
bash                            5.1-6ubuntu1                            deb        CVE-2022-3715        Low
binutils                        2.38-4ubuntu2.1                         deb        CVE-2017-13716       Low
binutils                        2.38-4ubuntu2.1                         deb        CVE-2018-20657       Negligible
...
github.com/aws/aws-sdk-go       v1.25.35               1.34.0           go-module  GHSA-76wf-9vgp-pj7w  Medium
github.com/aws/aws-sdk-go       v1.25.35               1.34.0           go-module  GHSA-7f33-f4f5-xwgw  Low
github.com/aws/aws-sdk-go       v1.25.35               1.34.0           go-module  GHSA-f5pg-7wfw-84q9  Medium
github.com/cloudflare/circl     v1.1.0                                  go-module  CVE-2023-1732        High
github.com/cloudflare/circl     v1.1.0                 1.3.3            go-module  GHSA-2q89-485c-9j2x  Medium
github.com/docker/distribution  v2.8.1+incompatible    2.8.2-beta.1     go-module  GHSA-hqxw-f8mx-cpmw  High
github.com/hashicorp/terraform  v1.4.6                                  go-module  CVE-2018-9057        Critical
github.com/hashicorp/terraform  v1.4.6                                  go-module  CVE-2021-36230       High
...
```

In practice, a lot of the Low and Negligible reports end up being the sort of almost-nuisance CVEs that won’t be fixed anyway, so you’d likely want to filter that. You’re going to be more interested in Critical and High. Again, we’ve also short-changed ourselves by letting its output in a lossy tabular format. Did you notice that it doesn’t tell you **where** this vulnerability was found in the filesystem; this is very important to know when remediating as you might have multiple versions of a library at different locations.

```bashsession
$ grype --output json=your-hostname-here.vulns.json \
>   --output table=your-hostname-here.vulns.txt \
>   sbom:your-hostname-here.syft.json
   Vulnerability DB        [no update available]
   Scanning image...       [629 vulnerabilities]
       3 critical, 80 high, 301 medium, 213 low, 32 negligible
       92 fixed
Report written to "your-hostname-here.vulns.json"

$ grype --output=table --file your-hostname-here.vulns.txt sbom:your-hostname-here.syft.json
...
Report written to "your-hostname-here.vulns.txt"

$ grype --output=table --file your-hostname-here.fixed-vulns.txt --only-fixed \
>   sbom:your-hostname-here.syft.json
...
Report written to "your-hostname-here.fixed-vulns.txt"
```

In this particular system, I have multiple versions of Terraform installed, which is (hopefully) why there are so many vulnerabilities with fixes listed. Let’s have a peek at the text version showing fixed vulnerabilities:

```bashsession
$ egrep -v -e '\(suppressed\)' -e ' (Negligible|Low|Medium) *$' your-hostname-here.fixed-vulns.txt
NAME                            INSTALLED                            FIXED-IN                             TYPE       VULNERABILITY        SEVERITY
cryptography                    38.0.3                               39.0.1                               python     GHSA-x4qr-2fvf-3mr5  High
d3-color                        1.4.1                                3.1.0                                npm        GHSA-36jr-mh4h-2g58  High
github.com/docker/distribution  v2.8.1+incompatible                  2.8.2-beta.1                         go-module  GHSA-hqxw-f8mx-cpmw  High
golang.org/x/crypto             v0.0.0-20190308221718-c2843e01d9a2   0.0.0-20200124225646-8b5121be2f68    go-module  GHSA-cjjc-xp8v-855w  High
golang.org/x/crypto             v0.0.0-20190308221718-c2843e01d9a2   0.0.0-20200220183623-bac4c82f6975    go-module  GHSA-ffhg-7mh4-33c4  High
...
golang.org/x/text               v0.3.0                               0.3.7                                go-module  GHSA-ppp9-7jff-5vj2  High
golang.org/x/text               v0.3.0                               0.3.8                                go-module  GHSA-69ch-w2m2-3vjp  High
setuptools                      59.6.0                               65.5.1                               python     GHSA-r9hx-vwmv-q579  High
terser                          5.10.0                               5.14.2                               npm        GHSA-4wf5-vphf-c2xc  High
webpack                         5.64.4                               5.76.0                               npm        GHSA-hc6q-2mpp-qw7j  High
wheel                           0.37.1                               0.38.1                               python     GHSA-qwmp-2cf2-g9g6  High
```

As you can see, despite some filtering, it’s still quite busy. We’re almost at the point where we have some **actionable insight**; what’s missing is a succinct listing of **where** there are fixable actions I should take. Ideally, one line per action I need to take.

Before we iterate further though, how do we navigate the JSON to find the paths where the vulnerable software was found? Take the Vulnerability column and search for that in the JSON version. It’s a verbose JSON to trawl through, so let me guide you:

```json
{
 "matches": [
                                  //...
  {
   "vulnerability": {
    "id": "GHSA-cjjc-xp8v-855w",  // you search for this
    "dataSource": "https://github.com/advisories/GHSA-cjjc-xp8v-855w",
    "namespace": "github:language:go",
    "severity": "High",
    "urls": [
     "https://github.com/advisories/GHSA-cjjc-xp8v-855w"
    ],
    "description": "Panic in malformed cerftificate",
                                  //...
   "artifact": {                  //  scroll down to the artifact section
    "id": "c43c381873340d0c",
    "name": "golang.org/x/crypto",
    "version": "v0.0.0-20190308221718-c2843e01d9a2",
    "type": "go-module",
    "locations": [                // you want the paths under here
     {
      "path": "/root/go/pkg/mod/golang.org/x/net@v0.0.0-20190628185345-da137c7871d7/go.mod"
     }
                                  //...
```

We *could* use jq or similar to get a report on this, as shown…you could certainly make a script for that:

```plain
$ jq '
  .matches[] 
  | select(.vulnerability.id=="GHSA-cjjc-xp8v-855w")
  | { "vulnerability_id": .vulnerability.id,
      "description": .vulnerability.description,
      "severity": .vulnerability.severity,
      "type": .artifact.type,
      "locations": (.artifact.locations[]|.path),
      "version": .artifact.version,
      "fixed_in": .vulnerability.fix.versions
    }' your-hostname-here.vulns.json
{
  "vulnerability_id": "GHSA-cjjc-xp8v-855w",
  "description": "Panic in malformed cerftificate",
  "severity": "High",
  "type": "go-module",
  "locations": "/root/go/pkg/mod/golang.org/x/net@v0.0.0-20190628185345-da137c7871d7/go.mod",
  "version": "v0.0.0-20190308221718-c2843e01d9a2",
  "fixed_in": [
    "0.0.0-20200124225646-8b5121be2f68"
  ]
}
{
  "vulnerability_id": "GHSA-cjjc-xp8v-855w",
  "description": "Panic in malformed cerftificate",
  "severity": "High",
  "type": "go-module",
  "locations": "/usr/local/bin/aws-okta",
  "version": "v0.0.0-20190701094942-4def268fd1a4",
  "fixed_in": [
    "0.0.0-20200124225646-8b5121be2f68"
  ]
}
{
  "vulnerability_id": "GHSA-cjjc-xp8v-855w",
  "description": "Panic in malformed cerftificate",
  "severity": "High",
  "type": "go-module",
  "locations": "/root/go/pkg/mod/github.com/99designs/keyring@v1.1.6/go.mod",
  "version": "v0.0.0-20190701094942-4def268fd1a4",
  "fixed_in": [
    "0.0.0-20200124225646-8b5121be2f68"
  ]
}
```

Well, it is at least better than looking at the JSON by hand, but Grype does allow us to provide our own Golang text template. If we find the template it uses for its own tabular format, we could improve on that to include the location. Unfortunately, Grype doesn’t use the Text templating engine for its table output. Instead, it uses a table writer that knows about column alignment, etc. But we still have an [example](https://github.com/anchore/grype/blob/main/grype/presenter/template/test-fixtures/test.template) in the test fixtures.

Put this into a file; I called mine fixed-vulnerability-report.template

```go template
Name: {{.Descriptor.Configuration.Name}}
Time: {{.Descriptor.Timestamp}}
Identified distro as {{.Distro.Name}} version {{.Distro.Version}}.
{{- range .Matches}}
    {{- if eq .Vulnerability.Fix.State "not-fixed" }}{{continue}}{{- end}}
    {{- if eq .Vulnerability.Severity "Negligible"}}{{continue}}{{- end}}
    {{- if eq .Vulnerability.Severity "Low"}}{{continue}}{{- end}}
    {{- if eq .Vulnerability.Severity "Medium"}}{{continue}}{{- end}}
    Vulnerability: {{.Vulnerability.ID}} ({{.Vulnerability.Severity}})
    Package: {{.Artifact.Name}} version {{.Artifact.Version}} ({{.Artifact.Type}})
    Fix state: {{.Vulnerability.Fix.State}}
    Fixed in: {{range .Vulnerability.Fix.Versions}}{{.}} {{end}}
    {{- if eq .Artifact.Type "deb" }}
    {{- else if eq .Artifact.Type "rpm" }}
    {{- else}}
    Locations:
    {{- range .Artifact.Locations }}
      - {{.RealPath}}
    {{- end}}
    {{- end}}
{{end}}
```

Now when you run it… well, it’s far from great; it really needs some grouping by artefact, rather than grouping by vulnerability. But at least you can easily search for the vulnerability and get some useful text you can copy-paste into a Jira ticket or such, ready to assign to someone to work on. You could even change the template to output Jira markup.

```bashsession
$ grype --template fixed-vuln-report.template --output template sbom:your-hostname-here.syft.json
   Vulnerability DB        [no update available]
   Scanning image...       [629 vulnerabilities]
       3 critical, 80 high, 301 medium, 213 low, 32 negligible
       92 fixed
Name:   <<syft bug: this data is not surfaced in the model presented to the template>>
Time: 2023-05-31T02:12:57.819319627Z
Identified distro as ubuntu version 22.04.
    Vulnerability: CVE-2018-20225 (High)
    Package: pip version 23.1.2 (python)
    Fix state: unknown
    Fixed in:
    Locations:
      - /venv.ansible/lib/python3.10/site-packages/pip-23.1.2.dist-info/METADATA
      - /venv.ansible/lib/python3.10/site-packages/pip-23.1.2.dist-info/RECORD
      - /venv.ansible/lib/python3.10/site-packages/pip-23.1.2.dist-info/top_level.txt
...
    Vulnerability: GHSA-69cg-p879-7622 (High)
    Package: golang.org/x/net version v0.0.0-20190628185345-da137c7871d7 (go-module)
    Fix state: fixed
    Fixed in: 0.0.0-20220906165146-f3363e06e74c
    Locations:
      - /usr/local/bin/aws-okta
...
```

## Fleet Operations

If we were to consider orchestrating this using Ansible, it’s worth asking the question, “Should we be installing anything (permanently) on the system at all?” Syft moves at a fairly brisk pace. It would be useful to ensure we’re operating with the latest release of Syft and Grype when we undertake such an audit activity. Syft is written in Go and so deploying is it really just copying a single static binary, but is it really that simple?

Like any environment, the fleet is made up of multiple different OS releases, and it’s common to have systems that are long overdue for an upgrade. Go is quite modern, but how much of our environment might it be incompatible with? Will it still support RHEL 6? What about RHEL 5? RHEL 5 kernel lacks some of the system API that Go requires, so Go won’t run there. RHEL5 also has issues when using Ansible these days too, so we can mostly forget about RHEL5. If you don’t have that luxury — hey, we’ve all been there — you could take a disk image (perhaps a backup) and mount it on another machine with a newer OS. What about RHEL6? Yes, that does indeed work.

Working towards something you might script, here’s part of a script that can copy syft to a server, run syft, copies the SBoM back, and then cleans up.

```bash
scp syft root@a-server:
ssh root@a-server TMPDIR=/var/tmp/ nice \
  ./syft --output syft-json --file a-server.sbom.syft.json \
    --exclude ./proc --exclude ./sys --exclude ./dev --exclude ./run \
    dir:/
scp root@a-server:a-server.sbom.syft.json .
ssh root@a-server rm syft a-server.sbom.syft.json
jq '.artifacts[] | select(.name == "java")
    | {"name": .name, "cpes": .cpes, "paths": .locations | map(.path)}
    ' a-server.sbom.syft.json
```

Things to improve, lest they bite you:

- The resulting SBoM files can be quite large, even 100MB; you may find that /tmp/ runs out of space. Use TMPDIR to point to /var/tmp/ if /tmp/ is likely to be too small.
- You can expect the SBoM generation to take about 1GB of memory, and something like at least 10 minutes per server. Much longer than a container image. It would be nice if this intermediate processing could be done on a different machine.
- Syft doesn’t have an option (as of version 0.82.0) to exclude filesystem types, so you’ll need to determine which parts of the file hierarchy should be excluded; a wrapper script will likely be desired.

## Looking for…Oracle Java?

Part of the premise of this blog post is the ability to look for particular software. Yesterday that might have been looking for instances of log4j. Today’s mission is to find and replace all instances of Oracle Java to ensure we don’t have to start paying for it due to [licence changes](https://redresscompliance.com/oracle-java-licensing-changes-explaned-free/) which clients (particularly universities) are very concerned with due to the number of ‘users’ they are considered to now have.

Doing this sort of ad-hoc querying is likely something that the Anchore’s Enterprise offering should support, but let’s see if we can demonstrate some value using the existing tooling that might help people to justify the procurement without requiring querying developer skills to query a bunch of JSON documents.

As output, I need to know:

- which servers have Java installed
- which distribution (eg. Oracle, IBM, Azul Platform Core, Eclipse Temurin, Red Hat OpenJDK, Amazon Corretto, etc.)
- which version(s) were found
- JDK or JRE?
- where it was found
- also nice to know would be if there were any traces that might indicate whether any of the commercial features were used
- would be useful to know if it’s part of a wider software product that might confer a Right to Use.

All of this information could then be used as an early step in helping plan a migration effort. Bearing in mind that it can’t be just a search and replace because of considerations such as:

- JAVA_HOME and similar environment variables; class-paths.
- Java’s java.properties and security.properties files etc, may have been updated to allow for TLS settings, Java Cryptography Extension (JCE), proxy settings, and more.
- There may well be additional libraries (Java and native) that will need to be considered (eg. database drivers)
- Some environments may make use of Oracle Java specifics
- You might need/want to incorporate other upgrades at the same time (eg. to satisfy a compatibility matrix)

It would be useful to integrate the SBoM data, which Syft provides, as part of a multi-tool ‘detect and then enrich’ strategy.

Rather than try to simulate a server by installing multiple versions of Java, I’m going to test with an actual old server. Setting up a test environment would involve downloading Oracle Java, which involves creating a login, including giving an address… and I’d rather neither volunteer that information nor give false information.

When looking for particular pieces of software, it is useful to know what the CPE would be. [CPE](https://nvd.nist.gov/products/cpe) is part of the (American) National Vulnerability Database (NVD).

> CPE is a structured naming scheme for information technology systems, software, and packages. Based upon the generic syntax for Uniform Resource Identifiers (URI), CPE includes a formal name format, a method for checking names against a system, and a description format for binding text and tests to a name.

Knowing this, it is reasonable to ask the CPE for Oracle Java, or more accurately, the Oracle JRE and JDK. You could [search](https://nvd.nist.gov/products/cpe/search/results?namingFormat=2.3&orderBy=CPEURI&keyword=Oracle+JDK&status=FINAL&startIndex=0), although I ran a scan and then looked around at the resulting JSON:

```bashsession
# jq '.artifacts[] | select(.name == "java") | {"name": .name, "cpes": .cpes, "paths": .locations | map(.path)}' myserver.sbom.syft.json
{
  "name": "java",
  "cpes": [
    "cpe:2.3:a:oracle:openjdk:1.8.0_332-b09:*:*:*:*:*:*:*",
    "cpe:2.3:a:java:java:1.8.0_332-b09:*:*:*:*:*:*:*"
  ],
  "paths": [
    "/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.332.b09-1.el7_9.x86_64/bin/java",
    "/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.332.b09-1.el7_9.x86_64/jre/bin/java"
  ]
}
```

That server was just OpenJDK… trying some other servers, we see some other CPE, such as this one for IBM Java.

```plain
"cpe:2.3:a:ibm:java:1.8.0-foreman_2022_01_20_09_33-b00:*:*:*:*:*:*:*"
```

But it didn’t find everything…on one server in particular, it should have found the following instances of Oracle Java, but it didn’t:

```bashsession
# /opt/oracle/product/12.2.0/client_64/jdk/bin/java -version
java version "1.8.0_91"
Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.91-b14, mixed mode)
# /opt/oracle/product/12.2.0/client_32/jdk/bin/java -version
java version "1.8.0_91"
Java(TM) SE Runtime Environment (build 1.8.0_91-b14)
Java HotSpot(TM) Server VM (build 25.91-b14, mixed mode)
```

## Learnings

Ah, so our proof-of-concept has experienced “rapid unplanned disassembly” and has not quite achieved lift-off. But we still can give some useful commentary about where Syft can fit into this problem domain.

- Syft does a very good job at finding library dependencies, particularly in software ecosystems that have a structured way of obtaining software dependencies.
- Syft will still struggle to identify applications where the packaging of an application is not a system concern (e.g., zip/tar rather than yum/apt). This is natural, and you’ll need flexibility to do this well. This will also be particularly true of web applications, but it will likely do a fine job of telling you about the dependencies within those applications.
- Syft does do a fairly remarkable job with modern languages such as Go and Rust, which create static executables, so it can certainly provide value with the ‘downloaded a release binary from GitHub’ type of deployment.
- Scanning tools such as Syft build up a large dataset during their operation:
  - be sure to consider its impact on system and network resources
  - servers are generally larger than container images; Syft’s footprint will be larger
  - running Syft on servers is not really the design environment for Syft
  - tune the exclusions for each server…but don’t forget that software assets can be found over NFS and SMB also, as well as in artefact repositories.
- In the case of finding Oracle Java, you’re likely going to get a better result with a combination of ‘find,’ ‘grep,’ and then looking inside the ‘…/jdk/release’ file. You might expand that to then look for other traces that might tell you whether the commercial aspects of Oracle Java have ever been used.
- When evaluating Enterprise type of offerings, look for flexibility in adding custom scans so your enterprise offering doesn’t come with blinkers attached.
- For custom scanning, work towards a common strategy so you can quickly integrate a small script and have a pre-developed story around querying and reporting on that data.
- Whether you use Syft or something more bespoke, you’re going to get better value if you can leverage a tool such as Ansible to orchestrate the workload.

Overall, I would say that Syft is a highly **complementary** system that would pair well with more established Software Inventory / Discovery solutions that also target Licence Compliance use cases, such as Manage Engine or Microsoft Defender Vulnerability Management. For development teams, tools such as Syft are essential, and an ideal integration point is within Artefact Repositories in general and Container Image Registries in particular. For systems management use cases, Syft will have a DIY feel to it when used by itself. It’s certainly not something I would want to run routinely as something added to existing systems, but it is a useful tool to have available when dealing with the next library-based vulnerability like Log4Shell or ShellShock, etc.