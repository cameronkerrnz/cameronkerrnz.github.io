+++ 
draft = false
date = 2022-11-22
title = "Amazon EFS Top-Talkers"
description = ""
summary = "Use Amazon's VPC flow logs (similar to NetFlow) to identify NFS clients and servers and enrich that IP-level information with useful names so you can recognise EC2 instances, EFS endpoints etc."
slug = ""
authors = []
tags = ["AWS", "EFS", "NFS"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published on the [Pythian blog](https://blog.pythian.com/).

Recently, I was helping a client with their AWS Elastic File Service (EFS), which is a managed NFSv4.1 service, with some additional connectivity and authentication capabilities. I was new to this client, so had little prior knowledge of their NFS workloads.

This blog post will cover how you can gain visibility into EFS traffic flows when VPC flow data is available, and then turning that IP data into AWS objects we can recognise. As well as, will show how to do this from first principles using shell-scripting to get that ‘just-enough-automation’ feeling to be useful. You can piece all these together into one script if you like; it’s certainly not a finished product. In a future blog post, I hope to show you how you can use the AWS Go SDK to turn this into a nicer tool.

## Lack of Console Visibility

After looking in the AWS Console and CloudWatch Metrics, you may find it difficult to know which NFS clients are using which NFS exports. The EFS console doesn’t tell you this information; but if you are lucky enough to have VPC Flow Logging enabled, you can use the fact that EFS is TCP port 2049 to make a list of top-talkers.

The VPC Flog Logs are (presumably) captured using NetFlow, but they are presented to us CSV files in S3 that are still quite raw; they are still essentially Layer-4 data, so we have IP addresses, which by themselves are not very recognisable.

Once we’ve queried our flow logs – we need to then enrich that data. I want to know which EC2 instances (etc.) accessed which EFS services using names my client’s business should be able to recognise.

## What You’ll Need

If you wish to play along at home, you’ll need an AWS account with at least the following:

- VPC Flow Logging setup and being sent to a CloudWatch Log Group
- At least one EFS service; preferably multiple
- Various EC2 instances that will use the EFS service
- Optionally some Fargate/ECS instances, Lamba instances… other services which can use EFS other than just EC2.

## Obtain VPC Data

We should repeat this report at different times of the day and night to get a useful sampling while keeping the amount of data manageable; think of daily/nightly processing/backup jobs that frequently leverage EFS. However, for this blog post, we show just one 5-minute time-slice.

To do this using the AWS CLI, you need to have at least the following:

- start a query of AWS VPC Log Insight
- wait for the query to complete
- retrieve results and save them as JSON

Expressed as simple shell script:

```bash
#!/bin/bash
flow_logs_bucket="my-vpc-flow-logs"
query_id=$(
  aws logs start-query \
    --log-group-name "${flow_logs_bucket}" \
    --start-time $(date +%s --date="5 minutes ago") \
    --end-time $(date +%s) \
    --query-string '
      fields @message
      | filter dstPort = 2049 and protocol = 6
      | stats sum(bytes) as bytesTransferred by srcAddr, dstAddr
      | sort bytesTransferred desc
      | limit 50
    ' \
    --output json \
  | jq -r .queryId)
echo -n "Awaiting query completion ..."
while sleep 5
do
    query_status=$(
        aws logs describe-queries --log-group-name "${flow_logs_bucket}" \
        | jq -r '.queries[] | select(.queryId == "'"${query_id}"'") | .status'
    )
    echo -en "\rAwaiting query completion ... ${query_status} ($(date))"
    case "${query_status}" in
        Scheduled|Running)
            continue
            ;;
        Complete)
            echo ''
            break
            ;;
        *)
            echo -en "\nUnhealthy or unhandled query status of $query_status"
            break; # exit 1
            ;;
    esac
done
echo "Retrieving query results"
aws logs get-query-results --query-id "${query_id}" --output json \
  > logs.json
echo "Converting for Tab-Separated-Values format"
jq -r '.results[]|[.[0].value, .[1].value, .[2].value] | @tsv' \
  < logs.json \
  > logs.tsv
```

At this point it would be worth considering trimming the result-set so that it only has bytesTransferred of greater than say 400 bytes. This is because you might have non-EFS traffic that just happens to go to port 2049.

I could have just imported it into a SQLite datafile; that would have made some of the following a bit easier perhaps, but SQLite was not available to be at the time.

## Looking up IP address information

To start on this process, have a look at how to find the owner of an unknown IP address, although it only provides the start of the process and that document lacks depth of tooling.

I wish to avoid duplicate lookups, as they are slow and definitely cache-worthy, so let’s get the unique list of IPs:

```bash
< logs.json \
  jq -r '.results[][] | select(.field == "dstAddr" or .field == "srcAddr") | .value' \
  | sort -u \
  > unique_ips.tsv
```

It would be nice to be able to convert each IP to some recognisable name; there’s no tooling that AWS provides for this, so we have to build it ourselves, somewhat. We won’t try and get all the way to an ARN (which aren’t particularly recognisable anyway), but we’ll aim to make it useable for actionable insight.

Ideally we’d have an IPAM (IP Address Management) service allocated which would allow us to query historical assignment of addresses, but that’s not the default experience, and the IPAM is not cheap, but should be attractive to those with high compliance and auditing requirements. So instead we just look at current assignments. Beware short-term use-cases such as ECS or Lamba environments, which will be more likely to be missed if not being used at the time of enrichment.

```bash
cat unique_ips.tsv \
  | while read ip
    do
      echo -en "$ip\t"
      aws ec2 describe-network-interfaces \
        --filters Name=addresses.private-ip-address,Values="$ip" \
        --query 'NetworkInterfaces[0] | [Attachment.InstanceId, Attachment.InstanceOwnerId, NetworkInterfaceID, InterfaceType, Description]' \
        --output text
    done \
  > ipdata1.tsv
column -s$'\t' -t ipdata1.tsv
```

That gets us part of the way, we can point to EFS endpoints, ELB backends, and more. To get further information, we would need to look at the interface type and act accordingly (eg. the ‘interface’ interface type would be an EC2 instance (possibly managed by AWS), but the interface type might also be ‘lambda’, ‘load_balancer’, ‘nat_gateway’, ‘vpc_endpoint’ and various others.

In the case of EFS though, an EFS endpoint would have a Description field similar to the following:

    EFS mount target for fs-xxxxxxxx (fsmt-xxxxxxxx)

There is a little more information on [RequesterManaged services](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/requester-managed-eni.html) to know more about whether an AWS service requested this on your behalf.

But keeping this brief and actionable, let’s just lookup EC2 and EFS endpoints. Let’s start with EC2, getting the Name tag, falling back to the InstanceId if the Name tag doesn’t exist.

```bash
instance_id='i-xxxxxx'
aws ec2 describe-instances --instance-ids "$instance_id" \
    --query 'Reservations[].Instances[].not_null(Tags[?Key==`Name`]|[0].Value, InstanceId)|[0]' \
    --output text
```

One of these days I’ll be able to rattle off jq and JMESpath expressions in one attempt… but not today.

Let’s have a look at EFS

```bash
efs_id='fs-xxxxxx'
aws efs describe-file-systems --file-system-id "$efs_id" --query 'FileSystems[*].Name' --output text
```

Let’s integrate that, enriching our previous data:

```bash
cat ipdata1.tsv | while IFS=$'\t' read ip instance_id owner_id interface_id interface_type description
do
    label="$ip"
    if [ "$interface_type" = "interface" ]; then
        if [ "$owner_id" = "amazon-elb" ]; then
            # description is actionable enough already
            label="${description}"
        else
            label=$(aws ec2 describe-instances --instance-ids "$instance_id" \
                --query 'Reservations[].Instances[].not_null(Tags[?Key==`Name`]|[0].Value, InstanceId)|[0]' \
                --output text)
        fi
    elif [ "$interface_type" = "load_balancer" ]; then
        efs_id=$(echo "$description" | sed -rne 's/^EFS mount target for (fs-[^ ]*) .*$/\1/p')
        if [ -n "$efs_id" ]; then
            label=$(aws efs describe-file-systems --file-system-id "$efs_id" \
                --query 'FileSystems[*].Name' --output text)
        else
            label="$description"
        fi
    fi
    echo -e "${ip}\t${label}"
done | tee ip-labels.tsv
```

Okay, so now we have a file that has IP to label mappings, it’s time for some scripted search and replace.

```bash
cat logs.tsv | awk '
    BEGIN {
        IFS="\t"
        OFS="\t"
        while (getline < "ip-labels.tsv") {
            replacements[$1] = $2
        }
        print "SOURCE",                          "DESTINATION",                     "BYTES"
        print "===============================", "===============================", "==========="
    }
    {
        $1 = replacements[$1]
        $2 = replacements[$2]
        print
    }
' | column -s $'\t' -t
```

## The Big Reveal 

One completely fair criticism is that some rows now appear to have the same pair of source and destination. I’ve marked these rows with <--. This is because the data for this table used source and destination IP and Port, but we then translate those to names which are shared, such as two EC2 instances sharing the same Name tag.

The last line should be filtered out. I’ve left it in to highlight the fact that NetFlow data doesn’t capture the direction in which a connection is made. In this case, we see an ELB backend connecting from port 2049; showing us that ELB uses a wider-than usual range of ephemeral ports.

```plain
SOURCE                           DESTINATION                      BYTES
===============================  ===============================  ===========
management-server                myapp1-efs-general-purpose       500123456
management-server                myapp2-efs-high-io               71364323
job-server                       myapp2-efs-high-io               4492812 <--
job-server                       myapp2-efs-high-io               2039831 <--
application-server               myapp2-efs-high-io               1592830
job-server                       myapp2-efs-high-io               1382237
job-server                       my-efs-home-dirs                 5423
job-server                       my-efs-home-dirs                 4321
management-server                my-efs-home-dirs                 3212
some-other-nfs-server            my-efs-home-dirs                 2445
job-server                       myapp1-efs-general-purpose       2124
application-server               my-efs-home-dirs                 1731
management-server                some-other-nfs-server            884
job-server                       some-other-nfs-server            624
123.34.45.67                     ELB                              120
```

## Limitations

The flow information makes the assumption that all EFS traffic will be TCP/2049, which is generally true. However, it won’t catch other NFS traffic (eg. if you have an NFS service hosted on an EC2 server). It will also catch other traffic that happens to try and connect to TCP/2049, so to filter those out it would be useful to filter out flows with insignificant amounts of traffic (TODO: find a suitable filtering mechanism)

## Next Time

There’s not a lot of benefit here to using Bash for this tooling. We need to have tools like the AWS CLI and jq installed. It would have been nice to have SQLite or Pandas available, but that wasn’t available to me in the particular circumstance. Using an official AWS SDK would have been much preferable, and more correct (did you notice the lack of any pagination in what I have shown?). When doing anything rich in use of APIs, it is important to have good ergonomics with regard to caching, parallel operation, smarter request management, logging and often authentication support.

When using the AWS SDK, a lot of people might default to using Python and the boto3 library… but this brings its own (often much larger) set of dependencies and maintenance concerns. I would like a tool that I could easily bring with me and just run; subject to client permission of course… so I think this will be a good opportunity to try the AWS Go SDK and sharpen my Go skills.

But until then, I at least have a process that I (or my colleagues) could follow and even automate, so value delivered!