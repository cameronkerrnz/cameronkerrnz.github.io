+++ 
draft = false
date = 2011-08-28
title = "Build an inode Lookup Database using SQLite3"
description = ""
slug = ""
authors = []
tags = ["NFS", "SQLite"]
categories = []
externalLink = ""
series = []
+++

> This post was originally posted on my original Humbledown.org blog

Occassionally, such as when investigating [NFS traffic](/posts/2011/nfs-lock-analysis/), you need to determine which filename(s) correspond to a given inode, or set of inodes. Now, you could answer this question using a command such as find /someplace -inum 12345 -print, but that will generally be quite slow, and very cumbersome if you wanted to lookup multiple inodes.

I decided to do this a different way, by creating a database to map between inode and path. Furthermore, I decided to do this using sqlite3, because a) it ought to be faster than grepping through a file looking for a particular inode, b) sqlite3 is pretty useful and I thought this would be a good excuse to practice using it, c) it was easy to do so.

One important thing to note: if you model a table using an inode as a primary key (which must by definition be unique), then you will run into problems. This is because although inodes are unique in a filesystem, the paths that correspond to an inode are not, so you might have multiple rows with the same inode field. So, although we would want the 'inode' field indexed, it is not a primary key. SQLite3 doesn't care though, as each row has an internal row number.

To generate the database (which is held in one file; SQLite3 is very convenient like that), we can do the following:

```bashsession
$ sqlite3 inodes.sqlite3db 'CREATE TABLE inodes (inode INTEGER, path STRING)'
```

Done. Now we just need to populate it with data. We can use the **find** command to help with this. Since inodes are unique to a filesystem, I'm using the **\-xdev** option to prevent crossing over into other filesystems. I'm using the vertical bar to separate the fields in the output, as this is the default for sqlite3's **.import** function, although it can be changed.

```bashsession
$ find / -xdev -printf "%i|%p\\n" | head
2|/
261121|/boot
261131|/boot/vmlinuz-2.6.32-27-generic
261154|/boot/initrd.img-2.6.32-29-generic
261153|/boot/initrd.img-2.6.32-21-generic
261126|/boot/vmlinuz-2.6.31-20-generic
261140|/boot/System.map-2.6.32-21-generic
261179|/boot/abi-2.6.32-29-generic
261132|/boot/System.map-2.6.32-26-generic
261695|/boot/abi-2.6.32-32-generic
```

Okay, so that shows what the import data is like. We would write the whole thing to a file and then import that, but it makes more sence to pipe it directly into sqlite3. Note that **.import** can actually read from stdin, but you need to specify it as /dev/stdin, as it doesn't understand \-. I'm using **sudo** to prevent a lot of potential Permission denied messages, although I still get one, which is the Gnome Virtual File System.

```bashsession
$ time sudo find / -xdev -printf "%i|%p\\n" \
>  | sqlite3 inodes.sqlite3db '.import /dev/stdin inodes'
find: \`/home/cameron/.gvfs': Permission denied

real    1m32.258s
user    0m3.708s
sys     0m11.265s
```

Verifying...

```bashsession
$ sqlite3 inodes.sqlite3db 'SELECT \* FROM inodes LIMIT 5;'
2|/
261121|/boot
261131|/boot/vmlinuz-2.6.32-27-generic
261154|/boot/initrd.img-2.6.32-29-generic
261153|/boot/initrd.img-2.6.32-21-generic
```

Good, so the data has been imported, and we are ready to begin querying. Let's step back a little first: note that we have directories as well, as directories are just a special type of file that lists what is in the directory. We can use the **debugfs** command to output the contents of an inode (in this example, inode 2):

```bashsession
$ sudo debugfs -R 'cat <2>' /dev/sda5 | hexdump -C
debugfs 1.41.11 (14-Mar-2010)
00000000  02 00 00 00 0c 00 01 02  2e 00 00 00 02 00 00 00  |................|
00000010  0c 00 02 02 2e 2e 00 00  0b 00 00 00 14 00 0a 02  |................|
00000020  6c 6f 73 74 2b 66 6f 75  6e 64 00 00 10 00 00 00  |lost+found......|
00000030  14 00 0b 07 76 6d 6c 69  6e 75 7a 2e 6f 6c 64 00  |....vmlinuz.old.|
00000040  01 f4 0b 00 0c 00 03 02  6d 6e 74 00 01 fc 03 00  |........mnt.....|
00000050  0c 00 04 02 62 6f 6f 74  0f 00 00 00 18 00 0e 07  |....boot........|
00000060  69 6e 69 74 72 64 2e 69  6d 67 2e 6f 6c 64 00 00  |initrd.img.old..|
00000070  0e 00 00 00 10 00 05 07  63 64 72 6f 6d 00 00 00  |........cdrom...|
00000080  01 f6 09 00 0c 00 03 02  65 74 63 00 01 f0 0f 00  |........etc.....|
00000090  10 00 05 02 6d 65 64 69  61 00 00 00 04 f0 0f 00  |....media.......|
...
```

Anyway, back to querying. To query a single inode is pretty easy:

```bashsession
$ time sqlite3 inodes.sqlite3db 'SELECT \* FROM inodes WHERE inode = 1044571;'
1044571|/root/.synaptic/lock

real    0m0.238s
user    0m0.136s
sys     0m0.092s
```

To query multiple random inodes is a bit more annoying:

```bashsession
$ time sqlite3 inodes.sqlite3db '
    SELECT \* FROM inodes WHERE (inode = 2419) or (inode = 1044571);'
1044571|/root/.synaptic/lock
2419|/var/lock

real    0m0.313s
user    0m0.108s
sys     0m0.184s
```

But we can overcome that nicely:

```bashsession
$ time echo -e '784412\\n784499\\n784499' \
>  | while read inode
>  do
> echo "SELECT * FROM inodes WHERE inode = $inode;" 
>  done | sqlite3 inodes.sqlite3db
784412|/var/log/syslog.7.gz
784499|/var/log/user.log.3.gz
784499|/var/log/user.log.3.gz

real    0m0.688s
user    0m0.360s
sys     0m0.296s
```

So, it took about 1m30s to build the database, less than a second to query it multiple times. Compare that to using `find ... -inum X -print` multiple times, each of which takes about 1m30s (ignoring any filesystem caching)

Limitations of this method include: must be periodically updated; is not real-time, so you might not have all short-lived files; an inode might not always map to something currently in existance (eg. open a file by name, delete the file \[entry in directory\], then when the file is actually closed, the file will be deleted: this is common for temporary files).

Using this method in conjunction with something like monitoring NFS traffic, you could use it to find out which files are being accessed. When watching/sampling NFS traffic, you will only determine a mapping between inode and pathname when a LOOKUP request is made, not when operating on a file. If you sample 200,000 NFS packets, you might not find any mappings, depending on the nature of the traffic. So this can be very useful to determine what is going on.