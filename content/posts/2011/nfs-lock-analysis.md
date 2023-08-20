+++ 
draft = false
date = 2011-08-28
title = "NFS Lock Analysis with tshark (Wireshark) and Python"
summary = "NFS locking can impede the performance of your clustered application. Learn how to capture and analyse NFS traffic to see where your locking issues are."
description = ""
slug = ""
authors = []
tags = ["Networking", "Performance", "NFS", "Python", "Programming"]
categories = []
externalLink = ""
series = []
+++

> This post was originally posted on my original Humbledown.org blog

I had a need to investigate what was happening on an NFS export due to a performance related concern for an enterprise application spread over multiple servers. I wanted to know which files were being locked, how often, and how long it took for locks to be granted. I decided I would use the tshark command, which is the command-line equivalent of Wireshark. However, since tshark isn't on most servers, I recorded 200,000 NFS-related packets using tcpdump, then analysed it on my Mac laptop using tshark and a Python script.

Before I delve into how you can do this, let's first look at the final result:

```bashsession
$ nfs-lock-report -f nfs.pcap
inode   lk_den  lk_reqs  lk_ok_min  lk_ok_max  lk_int_min  lk_int_max  filename
22753   0       4        0.0        60.0       60.0        120.0       <UNKNOWN>
22754   0       4        0.0        60.0       60.0        120.0       <UNKNOWN>
22759   0       0        0.0        0.0        empty       empty       <UNKNOWN>
22795   0       0        0.0        0.0        empty       empty       <UNKNOWN>
22796   0       0        0.0        0.0        empty       empty       <UNKNOWN>
22813   0       0        0.0        0.0        empty       empty       <UNKNOWN>
22814   0       0        0.0        0.0        empty       empty       <UNKNOWN>
351289  0       5        0.0        60.0       60.0        120.0       <UNKNOWN>
352109  0       33       0.0        10.0       10.0        20.0        <UNKNOWN>
352110  0       68       0.0        5.0        5.0         10.0        <UNKNOWN>
352117  0       10       0.0        30.0       30.0        60.0        <UNKNOWN>
355131  0       68       0.0        5.0        5.0         10.0        <UNKNOWN>
537175  35      34       empty      empty      10.0        20.0        <UNKNOWN>

real    0m1.663s
user    0m1.395s
sys     0m0.187s
```

We can see that with the exception of inode 537175 all locks are being granted pretty much immediately. Inode 537175, on the other hand, has been denied every time in this sample (35 lock requests denied, versus 34 lock requests made: apparently the sample must have started between a lock request and reply). As there were no LOOKUP requests in this sample [that relate to the files being locked] the filenames are unknown. We can still lookup an inode using `find ... -inum XXX -print`, or something a bit more intelligent. We might then find something enlightening...

The report also shows [though I won't go into details because I don't yet trust the results] the range of lock grant times, and the range of intervals between subsequent lock requests for the same file.

So first thing first: capture the sample. This could be messy. Some notes:

You will need to capture not less than 256 bytes of each packet, in order to capture sufficient RPC and NFS/NLM-related data: `-s256`

When dealing with Sun RPC-related data (such as the various NFS services), use `rpcinfo -p <server>` to determine which ports to use...

...and you can determine which protocol (TCP or UDP) to use by using `nfsstat -m` to output the negotiated NFS options. UDP is used by default, but TCP is also available on some systems, and will likely be used by default if supported on both client and server.

You might end up with something like this, which is meant to capture NFS and NLM (locking) traffic. Note that your command will be different depending on the ports used:

```bashsession
$ tcpdump -w /tmp/nfs.pcap -c 200000 'host NFS-SERVER and ((tcp and port 2049) or (tcp and port 52170))'
```

Now to start analysing the data. I suggest you use both Wireshark and tshark at the same time. This is because Wireshark can be useful for finding out the names of the fields that you can have tshark print out. The name of the field is shown in the leftmost part of the status-bar when you click on a packet field: that is what shall be used when we use tshark to output certain fields.

Copy the capture file to where you want to analyse it. Open it in Wireshark and navigate to a NFS packet. Drill down into the file handle and right click on a field, such as the length (not all fields have the right menu options). Enter the NFS protocol options menu, and choose the type of server you have.

One thing we need to know in order to analyse NFS traffic, is that NFS file handles are opaque 32-byte strings that only have meaning to the NFS server implementation -- this is what makes NFS 'stateless'. Because the (opaque) structure of the NFS file handle is particular to the NFS server, and the implementation is not communicated in the packet, you need to tell Wireshark/tshark which implementation to interpret NFS file handle data, otherwise the inode (contained in the file handle) will be misinterpreted). There are a number of NFS server implementations that are known to Wireshark/tshark:

```bashsession
$ tshark -G defaultprefs | grep -C5 -i nfs
# A decimal number.
#newmail.default_port: 0

# Whether the dissector should snoop the FH to filename mappings by looking inside certain packets
# TRUE or FALSE (case-insensitive).
#nfs.file_name_snooping: FALSE

# Whether the dissector should snoop the full pathname for files for matching FH's
# TRUE or FALSE (case-insensitive).
#nfs.file_full_name_snooping: FALSE

# With this option display filters for nfs fhandles (nfs.fh.{name|full_name|hash}) will find both the request                       and response packets for a RPC call, even if the actual fhandle is only present in one of the packets
# TRUE or FALSE (case-insensitive).
#nfs.fhandle_find_both_reqrep: FALSE

# When enabled, this option will print the NFSv4 tag (if one exists) in the Info column in the Summary pane
# TRUE or FALSE (case-insensitive).
#nfs.display_nfsv4_tag: TRUE

# When enabled, shows only the significant NFSv4 Operations in the info column.  Others (like GETFH, PUTFH, etc) are not displayed
# TRUE or FALSE (case-insensitive).
#nfs.display_major_nfsv4_ops: TRUE

# Decode all NFS file handles as if they are of this type
# One of: Unknown, SVR4, KNFSD_LE, NFSD_LE, KNFSD_NEW, ONTAP_V3, ONTAP_V4, ONTAP_GX_V3, CELERRA
# (case-insensitive).
#nfs.default_fhandle_type: Unknown

# Whether the dissector will track and match MSG and RES calls for asynchronous NLM
# TRUE or FALSE (case-insensitive).
#nlm.msg_res_matching: FALSE
```

The option that concerns us at the moment is the `nfs.default_fhandle_type` option. Included is support for SVR4 (simple/basic Unix systems), KNFS (various Linux kernel-mode NFS server, with LE presumably being Little-Endian, and NEW perhaps being endian neutral) NFSD (Linux User-mode NFS server), ONTAP (NetApp enterprise filing appliance) and CELERRA (EMC's enterprise filing appliance).

The command above outputs the various options that can be used for configuring the protocol disectors. You pass any options using tshark's `-o` option. Other options that are handy to specify are the filename snooping options. However, it's entirely likely that you will end up with a lot of inodes that do not have an associated LOOKUP operation, so you may well need to do some looking later to determine what path(s) an inode corresponds to.

Now here's a script that could be used, with modifications, to monitor such traffic. It's not in a releasable state (I'd prefer to release it for multiple NFS server types), so I've left it as an exercise for the reader to make it suitable for your environment, look for the two COMPLETEMEs (one is in a comment, so it shouldn't take much work to complete it for all supported server types, except to test it):

```python
#!/usr/bin/env python
#
# Script to process a PCAP file and determine which files are facing lock contention,
# and for how long locks are being contended.
#
# Currently only engineered for COMPLETEME NFSv3 servers, but should be able to customised
# for other NFS servers and versions.
#
# Requires tshark to do much of the heavy lifting.
#
# Cameron Kerr <cameron@humbledown.org>
#
from optparse import OptionParser
import shlex, subprocess
import string

parser = OptionParser()
parser.add_option("-f", "--file", dest="pcap_filename", help="read packets from FILE", metavar="FILE")

(options, args) = parser.parse_args()


def try_int(s, fail_val=-1):
    """Parse an integer from a string, or return 'fail_val'"""
    try:
        return int(s)
    except:
        return fail_val


class Packet:
    """Represents a packet as reported from our tshark NFS-related output format"""

    FIELD_FRAME_NUMBER         = 0
    FIELD_FRAME_TIME_EPOCH     = 1
    FIELD_RPC_XID              = 2
    FIELD_RPC_PROGRAM          = 3
    FIELD_RPC_PROCEDURE        = 4
    FIELD_RPC_MSGTYP           = 5
    FIELD_NFS_FS_OBJ_INODE     = 6
    FIELD_NLM_STAT             = 7
    FIELD_NFS_NAME             = 8

    RPC_PROGRAM_NFS            = 100003
    RPC_PROGRAM_NLM            = 100021

    RPC_NFS_PROCEDURE_LOOKUP   = 3

    RPC_NLM_PROCEDURE_LOCK     = 2

    RPC_MSGTYP_REQUEST         = 0
    RPC_MSGTYP_REPLY           = 1

    NLM_LOCK_GRANTED           = 0
    NLM_LOCK_DENIED            = 1

    def __init__(self, line):
        """Takes a line obtained from our tshark output and splits it into its fields"""
        self.packet = string.split(line.rstrip('\n'), '\t')

        self.packet[self.FIELD_FRAME_NUMBER]       = try_int  (self.packet[self.FIELD_FRAME_NUMBER])
        self.packet[self.FIELD_FRAME_TIME_EPOCH]   = float    (self.packet[self.FIELD_FRAME_TIME_EPOCH])
        self.packet[self.FIELD_RPC_PROGRAM]        = try_int  (self.packet[self.FIELD_RPC_PROGRAM])
        self.packet[self.FIELD_RPC_PROCEDURE]      = try_int  (self.packet[self.FIELD_RPC_PROCEDURE])
        self.packet[self.FIELD_RPC_MSGTYP]         = try_int  (self.packet[self.FIELD_RPC_MSGTYP])
        self.packet[self.FIELD_NFS_FS_OBJ_INODE]   = try_int  (self.packet[self.FIELD_NFS_FS_OBJ_INODE])
        self.packet[self.FIELD_NLM_STAT]           = try_int  (self.packet[self.FIELD_NLM_STAT])

    def is_nfs(self):
        """Is the packet an NFS packet?"""
        return self.packet[self.FIELD_RPC_PROGRAM] == self.RPC_PROGRAM_NFS

    def is_nfs_lookup(self):
        """Is the packet an NFS LOOKUP packet?"""
        return self.is_nfs() and self.packet[self.FIELD_RPC_PROCEDURE] == self.RPC_NFS_PROCEDURE_LOOKUP

    def is_rpc_request(self):
        """Is the packet an RPC request?"""
        return self.packet[self.FIELD_RPC_MSGTYP] == self.RPC_MSGTYP_REQUEST

    def is_rpc_response(self):
        """Is the packet an RPC response/reply?"""
        return self.packet[self.FIELD_RPC_MSGTYP] == self.RPC_MSGTYP_REPLY

    def is_nlm(self):
        """Is the packet an NLM packet?"""
        return self.packet[self.FIELD_RPC_PROGRAM] == self.RPC_PROGRAM_NLM

    def is_nlm_lock(self):
        """Is this a NLM lock message?"""
        return self.is_nlm() and self.packet[self.FIELD_RPC_PROCEDURE] == self.RPC_NLM_PROCEDURE_LOCK

    def is_nlm_lock_request(self):
        """Is this a NLM request packet?"""
        return self.is_nlm_lock() and self.is_rpc_request()

    def is_nlm_lock_granted(self):
        """Was a lock request granted?"""
        return self.is_nlm_lock() and self.is_rpc_response() and self.packet[self.FIELD_NLM_STAT] == self.NLM_LOCK_GRANTED

    def is_nlm_lock_denied(self):
        """Was a lock request denied?"""
        return self.is_nlm_lock() and self.is_rpc_response() and self.packet[self.FIELD_NLM_STAT] == self.NLM_LOCK_DENIED

    def frame_number(self):
        """Return the frame number of this packet in the packet capture"""
        return self.packet[self.FIELD_FRAME_NUMBER]

    def frame_time(self):
        """Return the timestamp of this packet in the packet capture, in seconds and sub-seconds"""
        return self.packet[self.FIELD_FRAME_TIME_EPOCH]

    def xid(self):
        """Return the RPC transaction ID (opaque string) of a packet."""
        return self.packet[self.FIELD_RPC_XID]

    def inode(self):
        """Returns the inode communicated in this packet"""
        return self.packet[self.FIELD_NFS_FS_OBJ_INODE]

    def filename(self):
        """Returns the filename communicated in this packet"""
        return self.packet[self.FIELD_NFS_NAME]

tshark_command_string = '''
    tshark 
       -o nfs.file_full_name_snooping:TRUE 
       -o nfs.default_fhandle_type:COMPLETEME
       -o nfs.fhandle_find_both_reqrep:TRUE
       -o nlm.msg_res_matching:TRUE

       -r %s 

       -T fields 
       -e frame.number 
       -e frame.time_epoch 
       -e rpc.xid 
       -e rpc.program 
       -e rpc.procedure
       -e rpc.msgtyp
       -e nfs.COMPLETEME(you're looking for the inode field)
       -e nlm.stat
       -e nfs.name

       -R '(nlm.procedure_v4 == 2) 
           || ((rpc.msgtyp == 0) && (nfs.procedure_v3 == 3))'
''' % (options.pcap_filename)

tshark = subprocess.Popen(shlex.split(tshark_command_string), stdout=subprocess.PIPE)

filenames = {}
xid_inode_map = {}
lock_requests = {} # key is inode, value is a direction with 'request_frame', 'request_time' and 'count' keys

for line in tshark.stdout:
    packet = Packet(line)

    if packet.is_nfs_lookup() and packet.is_rpc_request():
        filenames[packet.inode()] = packet.filename()

    if packet.is_rpc_request() and packet.is_nlm_lock_request():
        #print "Inode %d with transaction ID %s made a lock request" % (packet.inode(), packet.xid())
        xid_inode_map[packet.xid()] = packet.inode()
        if packet.inode() in lock_requests:
            lock_requests[packet.inode()]['request_count'] += 1
            if lock_requests[packet.inode()]['previous_request_time'] is not None:
        lock_requests[packet.inode()]['lock_request_intervals'].append(
            packet.frame_time()
            - lock_requests[packet.inode()]['previous_request_time'])
        lock_requests[packet.inode()]['request_time'] = packet.frame_time()
            lock_requests[packet.inode()]['previous_request_time'] = lock_requests[packet.inode()]['request_time']
        else:
            lock_requests[packet.inode()] = {
                'request_frame':packet.frame_number(),
                'request_time':packet.frame_time(),
                'previous_request_time':None,
                'request_count':0,
                'denied_count':0,
                'granted_count':0,
                'lock_request_durations':[],
                'lock_request_intervals':[]}

    if packet.is_rpc_response() and packet.is_nlm_lock_denied():
        inode = xid_inode_map[packet.xid()]
        lock_requests[inode]['denied_count'] += 1
        #print "Request %s (Inode %d) had a lock request denied, now denied %d times" % (packet.xid(), inode, lock_requests[inode]['denied_count'])

    if packet.is_rpc_response() and packet.is_nlm_lock_granted():
        inode = xid_inode_map[packet.xid()]
        #print "Request %s (Inode %d) had a lock request granted" % (packet.xid(), inode)
        lock_requests[inode]['lock_request_durations'].append(packet.frame_time() - lock_requests[inode]['request_time'])
        lock_requests[inode]['granted_count'] += 1;

        

# filenames_items = filenames.keys()
# filenames_items.sort()
# 
# for inode in filenames_items:
#     print "Inode %d is file named '%s'" % (inode, filenames[inode])

lock_requests_items = lock_requests.keys()
lock_requests_items.sort()

def render_min(l):
    try:
    return round(min(l),1)
    except:
        return "empty"

def render_max(l):
    try:
    return round(max(l),1)
    except:
        return "empty"
        
print "inode   lk_den  lk_reqs  lk_ok_min  lk_ok_max  lk_int_min  lk_int_max  filename"
for inode in lock_requests_items:
    if inode in filenames:
        filename = filenames[inode]
    else:
        filename = '<UNKNOWN>'
        
    print "%-7d %-7d %-8d %-10s %-10s %-11s %-11s %s" % (
        inode,
        lock_requests[inode]['denied_count'],
        lock_requests[inode]['request_count'],
        render_min(lock_requests[inode]['lock_request_durations']),
        render_max(lock_requests[inode]['lock_request_durations']),
        render_min(lock_requests[inode]['lock_request_intervals']),
        render_max(lock_requests[inode]['lock_request_intervals']),
        filename
    )
```