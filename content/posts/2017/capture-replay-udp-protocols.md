+++ 
draft = false
date = 2017-04-25
title = "Capturing and Replaying Connection-less Protocols (eg. IPFIX into Logstash)"
description = ""
summary = "Capture real-world network traffic so you can replay it later / elsewhere for development and testing. An example is given using IPFIX (AppFlow), but directly relevant to UDP Syslog too."
slug = ""
authors = []
tags = ["Networking", "Linux", "Testing", "Logstash", "NetFlow", "AppFlow", "Syslog"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published at my old blog Distracted IT on Blogger.com

> This post should be reusable for tasks such as logging pipeline development (with UDP syslog) as well.

It can be useful to be able to capture AppFlow (IPFIX) data, which in our environment at least is UDP, and replay that on some other machine where you are playing with Logstash (or some other tool that might read in such data from the network). In this page, I show you how you can capture packets using tcpdump, rewrite them post-capture, and replay them as if they were sent to your own machine. We'll also set up a standalone Logstash instance that reads in IPFIX records and just emits them to stdout in a debugging format.  
  
## Step 1: Capture some traffic

This is easy; just remember to use a useful filter (not everything will rewrite easily), and capture the entire packet. 
  
Notes: I'm running this capture on production, so I limited the number of packets, using '-c 10000', that could be captured (to prevent disk blowout) I want the application data, so I'm capturing the entire packet with '-s0' Because I only want IPFIX data, and can only realistically rewrite connectionless protocols (namely UDP), I constrain this to the IPFIX port, using 'udp and port 4739' And because I'm only interested in a particular set of NetScaler instances that I've just enabled AppFlow exporting on, I constrain those too, to limit the amount of traffic I get.  

```bash
tcpdump -s0 -i eth0 -c 10000 -w /tmp/appflow.pcap \
    \( udp and port 4739 \) and \
    \( host netscaler-1.example.com or host netscaler-2.example.com \)
```

> Expect additional pain if you are capturing on an interface that is on a trunked VLAN port. I know I've seen this in subquequent years, but I didn't document how I solved that here.

When you have the capture file (you can ^C early if you've completed any testing activity you wanted), transport the file to your lab environment (a VirtualBox VM on my workstation, in my case). When you have it on your workstation, you should probably open it in Wireshark to ensure it looks like you would expect it to. You can get a nice summary using the 'capinfos' tool, which comes with Wireshark.  
  
```bashsession
$ capinfos /tmp/appflow-rewritten.pcap
File name:           /tmp/appflow-rewritten.pcap
File type:           Wireshark/tcpdump/... - pcap
File encapsulation:  Ethernet
Packet size limit:   file hdr: 65535 bytes
Number of packets:   5,548 
File size:           5,409 kB
Data size:           5,321 kB
Capture duration:    21944 seconds
Start time:          Tue Apr 11 14:52:40 2017
End time:            Tue Apr 11 20:58:24 2017
Data byte rate:      242 bytes/s
Data bit rate:       1,939 bits/s
Average packet size: 959.11 bytes
Average packet rate: 0 packets/sec
SHA1:                49ac53014c3f26e05fe708e686cc8203101049a5
RIPEMD160:           a002b0fa102b5d44839d712cc791328814249814
MD5:                 ecd939142cc242ed0ea84d59732cff14
Strict time order:   True
```

## Step 2: Rewrite the traffic

Okay, so now you've got a traffic capture file on your workstation, and the aim is to have something replay the application traffic – but the capture file has all that layer 2-3 data (destination MAC address, IP) that won't match your development environment. You could mock up a development environment that replicated the exact same network with the exact same addressing, but its often easier just to use a tool to rewrite the traffic inside the capture file.  
  
There are, of course, limitations to this. I wouldn't expect this to work well when layer-3 data bubbles up into the application-layer protocol; it could be done, but it would get messy quickly, and you may well have to use a different tool (I think a tool called bitwiste may be useful for this, but I've never used it).  
  
To do this rewriting, we're going to use a tool called 'tcprewrite', which on RHEL systems (with EPEL) is available via the 'tcpreplay' package. This package also provides the replay tool we shall use later.  
  
There are three things you'll generally want to rewrite:  
  
- map the destination IP address
- map the source IP address (recommended although may not be required if you allow martians)
- map the destination Ethernet address
- ... and then recalculate the checksums in IP headers and such.

For the first two, you can look at the capture file to determine that these were when captured, and then you can use the 'ip a' (or 'ifconfig' if you're old-fashioned). Note that if you're VM is attached to multiple networks (as mine is as per below), you'll need to choose one. Which one you choose will be immaterial with the following notes:  
  
- If your network application (that you're replaying to) is listening on the 'any' address (0.0.0.0 – 'lsof' and friends would show this as '\*', such as '\*:4739'), then it shouldn't matter, so long as the encapsulation is the same (eg. capturing from Ethernet and then replaying to a Loopback would require further work – vice versa would be even more work as Loopback interfaces have a much larger MTU than Ethernet).
- You must use two machines; due to limitations in how tcpreplay injects packets, you cannot receive the packets on the same machine as you inject them on.
- Remember the name of the interface you chose though, as we shall have to specify that when we do the replay.

```bashsession
$ ip a
1: lo: mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:c7:d8:6c brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 64686sec preferred_lft 64686sec
    inet6 fe80::a00:27ff:fec7:d86c/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:85:2a:08 brd ff:ff:ff:ff:ff:ff
    inet «receivers IP address in development environment»/24 brd 172.28.128.255 scope global dynamic enp0s8
       valid_lft 832sec preferred_lft 832sec
    inet6 fe80::a00:27ff:fe85:2a08/64 scope link 
       valid_lft forever preferred_lft forever
4: virbr0: mtu 1500 qdisc noqueue state DOWN qlen 1000
    link/ether 52:54:00:2d:c9:bc brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
5: virbr0-nic: mtu 1500 qdisc pfifo_fast master virbr0 state DOWN qlen 1000
    link/ether 52:54:00:2d:c9:bc brd ff:ff:ff:ff:ff:ff
```

Okay, so let's run the rewrite. I've translated the actual IP addresses to some more meaningful descriptions. We're also rewriting all the destination MAC addresses to be the MAC address of the interface I'm wanting to receive the replayed packets on (assuming you are delivering to a local station -- if you were rewriting this through a router, it would be your gateway's local MAC address, which should be visible in your ARP table).  
  
```bash
$ tcprewrite \
    --infile ~/host\_home/Desktop/appflow.pcap \
    --outfile /tmp/appflow-rewritten.pcap \
    --srcipmap "«IP address of NetScaler (NSIP)»:«senders IP address in development environment»" \
    --dstipmap "«collector IP address currently used in production»:«receivers IP address in development environment»" \
    --enet-dmac 08:00:27:85:2a:08 \
    --fixcsum
```

## Step 3: Prepare to Replay

The 'tcpreplay' tool will replay traffic at the same rate it appears in the capture; but it can also just throw data at a packet rate you specify, which can come in handy later. But before we do that, there are some **important considerations**:  
  
- Because of the way tcpreplay injects its packets, you have to send them from a different machine. (sad)
- If using VirtualBox, don't use Host-Only Networking — use Internal Network instead for this.
- You may want to set some static IPs too, because there was no DHCP for me on that network.
- Beware of martians.
  - It can be useful to turn on logging of martian packets when diagnosing issues. If you can see (using tcpdump) your packets coming in, but iptables counters that should increment don't, then this may be the issue. I found this to be the case for me because I hadn't altered the source IP address at the time.
  - `echo 1 > /proc/sys/net/ipv4/conf/IFACE/log_martians`
- Possibly reverse-path filtering gets in the way too. If you need to turn it off temporarily:
  - `echo 0 > /proc/sys/net/ipv4/conf/IFACE/rp_filter`
- Make sure you can get through any host-based firewall
  - `sudo firewall-cmd --add-port ipfix/udp` # note: --permanent was not specified

## Step 4: Add your application (eg. logstash)
  
I'm not going to describe how to install logstash; that's pretty simple, but here is a configuration fit to demonstrate this article.  
  
Create this as an input 
  
```logstash
input {
  udp {
    host => "0.0.0.0"
    port => 4739
    codec => netflow {
      versions => [10]
      target => ipfix
    }
    type => ipfix
  }
}
```

And an output  
  
```logstash
output {
  stdout {
    codec => "rubydebug"
  }
}
```
  
Rather than having systemd run logstash, I'll have it run in the foreground, because I want stdout (otherwise, you can use `journalctl -u logstash.service -elf`)  

```bash
/usr/share/logstash/bin/logstash --config.reload.automatic --path.config='/etc/logstash/conf.d/*.conf'   
```

Replay the data from your other host  

```bash
sudo tcpreplay -i enp0s8 /tmp/appflow-rewritten.pcap
```
  
And you get the following being output to stdout, as requested. Here's some examples. First is the connection between the NetScaler and an LDAPS server.  

```ruby
{
         "ipfix" => {
                "destinationTransportPort" => 39912,
                     "flowEndMicroseconds" => "2017-04-11T02:53:09.000Z",
                       "sourceIPv4Address" => "«virtual ip address»",
                     "netscalerUnknown329" => 0,
                         "egressInterface" => 0,
                         "octetDeltaCount" => 6600,
                   "netscalerAppNameAppId" => 165707776,
                     "sourceTransportPort" => 636,
                                  "flowId" => 14049270,
                  "destinationIPv4Address" => "«backend server ip address»",
                      "observationPointId" => 472006666,
                   "netscalerConnectionId" => 14049269,
                          "tcpControlBits" => 25,
                   "flowStartMicroseconds" => "2017-04-11T02:53:09.000Z",
                        "ingressInterface" => 2147483651,
                                 "version" => 10,
                        "packetDeltaCount" => 16,
                  "netscalerRoundTripTime" => 0,
              "netscalerConnectionChainID" => "00000000000000000000000000000000",
                               "ipVersion" => 4,
                      "protocolIdentifier" => 6,
                     "netscalerUnknown331" => 0,
                     "netscalerUnknown332" => 0,
                      "exportingProcessId" => 0,
                      "netscalerFlowFlags" => 1090527232,
                  "netscalerTransactionId" => 342306495,
        "netscalerConnectionChainHopCount" => 0
    },
    "@timestamp" => 2017-04-11T02:53:09.000Z,
      "@version" => "1",
          "host" => "«senders IP address in development environment»",
          "type" => "ipfix"
}
```
  
And here is an HTTP request being made to the NetScaler from the client.  
  
```ruby
{
         "ipfix" => {
               "netscalerHttpReqUserAgent" => "",
                "destinationTransportPort" => 443,
                  "netscalerHttpReqCookie" => "",
                     "flowEndMicroseconds" => "2017-04-11T02:52:49.000Z",
                     "netscalerHttpReqUrl" => "/example",
                       "sourceIPv4Address" => "«my workstation IP»",
                  "netscalerHttpReqMethod" => "POST",
                    "netscalerHttpReqHost" => "example.com",
                         "egressInterface" => 2147483651,
                         "octetDeltaCount" => 1165,
                   "netscalerAppNameAppId" => 36274176,
                     "sourceTransportPort" => 59959,
                                  "flowId" => 14043803,
           "netscalerHttpReqAuthorization" => "",
                 "netscalerHttpDomainName" => "",
                    "netscalerAaaUsername" => "",
                "netscalerHttpContentType" => "",
                  "destinationIPv4Address" => "«virtual IP address»",
                      "observationPointId" => 472006666,
                     "netscalerHttpReqVia" => "",
                   "netscalerConnectionId" => 14043803,
                          "tcpControlBits" => 24,
                   "flowStartMicroseconds" => "2017-04-11T02:52:49.000Z",
                        "ingressInterface" => 1,
                                 "version" => 10,
                        "packetDeltaCount" => 1,
                     "netscalerUnknown330" => 0,
              "netscalerConnectionChainID" => "928ba0c1da3300000145ec5805800e00",
                               "ipVersion" => 4,
                      "protocolIdentifier" => 6,
                  "netscalerHttpResForwLB" => 0,
                 "netscalerHttpReqReferer" => "",
                      "exportingProcessId" => 0,
               "netscalerAppUnitNameAppId" => 0,
                      "netscalerFlowFlags" => 151134208,
                  "netscalerTransactionId" => 342305773,
                  "netscalerHttpResForwFB" => 0,
        "netscalerConnectionChainHopCount" => 1,
           "netscalerHttpReqXForwardedFor" => ""
    },
    "@timestamp" => 2017-04-11T02:52:51.000Z,
      "@version" => "1",
          "host" => "«senders IP address in development environment»",
          "type" => "ipfix"
}
```

Sweeeet.....
