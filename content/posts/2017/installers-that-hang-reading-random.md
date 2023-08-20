+++ 
draft = false
date = 2017-03-23
title = "Point a single process to /dev/urandom instead of /dev/random"
description = ""
summary = "Giving a single process (tree) a different view of the filesystem without modifying the process; and showing how to do this manually and via Ansible."
slug = ""
authors = []
tags = ["Ansible", "Performance"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published at my old blog Distracted IT on Blogger.com

Okay, so I'm working on making an Ansible role for deploying an Oracle 10g Webgate, and I want it working on RHEL 6 and RHEL 7. I managed to do that (yay; took a bit of persuading), but quickly noticed that if you don't do something to prevent it, the installer (InstallShield on Linux... yuck, and not just because it wraps Java 6), your entropy pool drains very very quickly and the installer just sits there, hanging.  

You can verify that its blocking because of entropy by using a command such as:  

```bash
watch cat /proc/sys/kernel/random/entropy_avail  
```

If it isn't hovering between 2000 and 3000, then you have a potential issue; if its staying under 1024 or so, then you very likely will be experiencing hanging behaviour, depending on which application is draining the pool (commonly by trying to read a bunch of data from /dev/random)  

I should note that this is in a VMWare environment, so no hardware random number generation for me. Instead, what I typically do is to push out a sysctl change to change `sys.kernel.random.read_wakeup_threshold` from the default of 64 to a more useful 1024, which generally makes things (especially things like WebLogic startup times) much more responsive.  

Additionally, for instances where I'm dealing with Oracle Java < 8, I edit java.security (tip. locate java.security) and set the following:  

```ini
securerandom.source=file:/dev/./urandom  
```

These days, this gets done as part of an Ansible role I have for deploying Oracle Java.  
  
But, in the case of this particular installer, that wasn't really accessible, and although has options for doing a silent install (useful, if insufficiently documented, at least by Oracle), it doesn't have a way of passing in JVM arguments. (ie. `-Dsecurerandom.source=...`)

In my dev environment (a Vagrant VM), I just deleted /dev/random and recreated it with the same major&minor numbers as /dev/urandom, but I didn't want to do that in my real environments, partly to reduce the potential for surprise later on.

But we can do something similar, and much more constrained (I like constrained effects), using part of the Linux namespaces system to use an independent mount table -- part of what makes tools like Docker work. It does require root privilege however (or at least the power to modify the mount table).  

To demonstrate, I wrote this as a one-liner, here reformatted for a bit of enhanced readability. You can copy-paste this into a root shell to demonstrate. (naturally, read it before pasting anything into a root shell). The key component is the use of the `unshare` command, which basically means that the command it runs will have an independent (filesystem) namespace than its parent.

```bash
echo "Before unsharing"; \
  ls -l /dev/*random ; \
  unshare --mount bash -c '
    mount -o bind /dev/urandom /dev/random;
    echo "After unsharing and bind-mounting random device";
    ls -l /dev/*random;
    echo "... finickity installer goes here"
  '; \
  echo "Back to normal"; \
  ls -l /dev/*random
```
  
Here's the output, with some highlighting to make it easier to see the difference.  

```bashsession
Before unsharing  
crw-rw-rw- 1 root root 1, 8 Mar 16 23:59 /dev/random  
crw-rw-rw- 1 root root 1, 9 Mar 16 23:59 /dev/urandom  
After unsharing and bind-mounting random device  
crw-rw-rw- 1 root root 1, 9 Mar 16 23:59 /dev/random  
crw-rw-rw- 1 root root 1, 9 Mar 16 23:59 /dev/urandom  
... finickity installer goes here  
Back to normal  
crw-rw-rw- 1 root root 1, 8 Mar 16 23:59 /dev/random  
crw-rw-rw- 1 root root 1, 9 Mar 16 23:59 /dev/urandom  
```

Note that if the command you want to run should not run as root (quite likely), then you can change the line in the code above with a command that uses sudo, su, runuser, etc. possibly using a wrapper script. Or just use an interactive shell.

## Integration with Ansible

So, let's see how to apply this... this post isn't about Ansible, or about deploying a 10g Webgate using Ansible, but it won't hurt to show the key part. In this particular case, because the webgate will be running in Apache httpd (from RHEL package), the installer needs to run as root; if this was for an OHS deployment, then I would have to wrap this command in a 'sudo' command, and probably also make sure requiretty was disabled (at least for transitioning to the target user).  

Note that this part of the playbook is running as 'root' already.  

```yaml
### Ansible task before integrating this improvement

- name: run the installer ... beware entropy starvation  
  command: /root/webgate_installers/{{ webgate_installers_and_patches[0] }}/{{ webgate_installers_and_patches[0] }} -options /root/webgate_installers/install_options.txt -silent -is:silent  
  args:  
    chdir: /root/webgate_installers/{{ webgate_installers_and_patches[0] }}  
    creates: "{{ webgate_install_location }}/oblix"  

### Ansible task after integrating this improvement

- name: run the installer ... beware entropy starvation  
  command: |
    unshare --mount bash -c '
      mount -o bind /dev/urandom /dev/random
      /root/webgate_installers/{{ webgate_installers_and_patches[0] }}/{{ webgate_installers_and_patches[0] }} \
        -options /root/webgate_installers/install_options.txt -silent -is:silent'
  args:  
    chdir: /root/webgate_installers/{{ webgate_installers_and_patches[0] }}  
    creates: "{{ webgate_install_location }}/oblix"  
```

## Testing

Reverting my change to my Vagrant VM, making /dev/random be what it normally is... (you might reasonably wonder why I didn't just blow it away and restart it, but I had got some other Apache-related work in there), 

Watching entropy_avail while the playbook was running (removing the previous install in order to do the entire webgate install from fresh), the level of available random data showed as stable and bouyant, and the install proceeded at a very healthy rate.

(Note: it would have been faster, but there is a certain, unnecessary, registration activity the install wanted to run that takes a little while to time out).

## Conclusion

Previously, because this particular installer is such a hog, while running this playbook, I would need to open up a new window onto the (thankfully only one) server being executed on, and run commands such as updatedb, rpm --verify --all and any other command I could think of to generate some disk activity -- that being the only source of entropy in that environment.  
  
Now, instead of that, I have a playbook that runs in a fairly constant and minimal amount of time, rather than timing out and requiring manual intervention. Now that's the kind of speed-up I like to see.  

It should be said that this does decrease the quality of the random data... but if you're concerned about that you should first work to reintroduce a sufficient source of high-quality random data into your environment.
