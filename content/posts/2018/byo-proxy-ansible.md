+++ 
draft = false
date = 2018-01-25
title = "Deploying with Ansible when Outgoing Access to Internet is Unavailable (BYO proxy)"
description = ""
summary = "Network gynmastics using SSH and a local proxy for servers that don't have internet access but still need to 'yum update' etc."
slug = ""
authors = []
tags = ["Ansible", "Proxies", "SSH", "Security"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published at my old blog Distracted IT on Blogger.com

So, you've completed your Dev environment in some nice throwaway VMs on your workstation (perhaps using Vagrant); your Ansible playbook is ready with a nice sheen to it, and your new shiny VMs are ready and just begging to receive instruction from the playbook you have so lovingly crafted \[over the past few weeks, and expect to deploy to Test and Prod in a matter of a few days\].  
  
... wait, what you mean I don't have outgoing internet access; how am I supposed to get to my third-party repositories etc.? Is there a proxy I can use at least... not available in the DMZ; I see ...  
  
These sort of struggles are common as soon as you leave the nice comfortable confines of your development environment (where outgoing access is often easy) and start entering the real world of your network.  

**Moral: don't expect initial deployments to Test/QA to be terribly easy; there will be gotchas if its a different environment to Dev. Test should however be the same as Prod.**

So we need to do something. The fact that we're deploying to a DMZ raises some interesting security-related questions (eg. can we expose our internal proxy servers to the hosts in the DMZ --- perhaps best to assume not).  
  
But if you only need outgoing access (via a proxy) at the time when your playbook is running, you could do some SSH remote-port-forwarding magic to point your playbook (through appropriate environment variables) to use a short-lived proxy-server on the loopback interface that gets forwarded through to the proxy-server.  
  
Let's sketch this out a bit:  
  
![Ansible SSH Proxy](/post-images/distracted-it/Ansible-SSH-Proxy.jpg "Ansible SSH Proxy")
  
Okay, so you're sitting on the laptop/workstation at the bottom, where you run Ansible. You have access to a proxy server; this could also be a Squid instance you set up on your laptop. Firewall policies either side of the DMZ restrict traffic such that the DMZ host can not make outbound connections to the Internet or to the LAN; although we assume that some inbound traffic can make it in to the host; although that is outside the scope of todays discussion.  

The box in the DMZ is the machine you will be deploying to (possibly multiple, in which case each will be the same).  
  
The red tunnel is the SSH connection, made from your workstation (by Ansible). Half of the secret sauce is getting Ansible (or rather SSH) to create a 'remote port forward', such that if a process on the host (eg. yum) were to connect to 127.0.0.1:3128, then that request will be securely tunnelled back through the SSH connection to your workstation, where SSH will establish a TCP connection to the proxy server -- in this case also port 3128; but there is no requirement they be the same -- and forward the bytes to/fro. That will effectively connect yum to the Squid instance running on the proxy. The proxy will see the connection coming from your workstation, which is significant in terms of any ACLs that need to be applied in the proxy configuration. Any authentication data (eg. username/password, must be supplied by yum in this case -- probably via the `http_proxy` environment variable.)
  
Note that you will also want `https_proxy` set the same as `http_proxy` (or at least set), otherwise you will likely find that https will not work. You may also want to set `no_proxy`. For this instance, I only intend to set `http_proxy` and `https_proxy` exactly where and when I need access.  

## OpenSSH
  
Before we jump into Ansible, let's show this from first principles. First we establish an SSH session from my workstation to my test host (host1). Then I set the proxy environment variable when I run curl to ensure I can get to the mirror-list URL of a repository I have configured. We see the header indicating that it come via the proxy, so we know the process works.  

```bashsession
[WORKSTATION] $ ssh -R 3128:proxy.example.com:3128 host1.dmz.example.com  
  
[TEST] $ `http_proxy`=127.0.0.1:3128 curl -i 'http://mirrorlist.centos.org/?release=7&arch=x86\_64&repo=os&infra=stock'  
HTTP/1.0 200 OK  
...  
X-Cache: MISS from proxy.example.com  
... 
... it works!  
```

Exit the shell once done.

Now to make this magic happen every time we fire up SSH to that host. To do this, we make use of the `~/.ssh/ssh_config` client configuration (man ssh\_config for details). The important part is to set the RemoteForward for the hosts of interest.

```plain
Host host\*.dmz.example.com

  RemoteForward 3128 proxy.example.com:3128
```

Good, with that set, we can repeat without the the -R argument.

```bashsession
[WORKSTATION] :~ $ ssh host1.dmz.example.com

[TEST] $ `http_proxy`=127.0.0.1:3128 curl -i 'http://mirrorlist.centos.org/?release=7&arch=x86\_64&repo=os&infra=stock'

HTTP/1.0 200 OK
...
X-Cache: MISS from proxy.example.com
...
... it still works!
```

## Ansible

Okay, now to get my Ansible playbook to [set the proxy environment](http://docs.ansible.com/ansible/latest/playbooks_environment.html). Here's how I've done it:  
  
Put this in some vars block. You might already have some specified in eg. `group_vars/all.yml`  

```yaml
proxy_env:  
   http_proxy:  http://proxy.example.com:3128  
   https_proxy: http://proxy.example.com:3128  
   no_proxy:    example.com,\*.example.com,127.0.0.1,localhost  
```

You should create a group for your DMZ hosts (that will have this configuration applied to them). Let's say you create a group 'dmzhosts' and then put this in `group_vars/dmzhosts.yml`

```yaml
proxy_env:  
   http_proxy:  http://127.0.0.1:3128  
   https_proxy: http://127.0.0.1:3128  
   no_proxy:    example.com,\*.example.com,127.0.0.1,localhost
```

Then you need to get Ansible to use those environment variables when needed. However, this will require Ansible 2.4 due to the group prioritisation to be useful.

Here's an example task, although you could well consider other ways of doing it, as described in the linked article from Ansible.

```yaml
- name: install the YUM Priorities plugin
  yum:
    name: yum-plugin-priorities
    state: latest
  environment: "{{ proxy\_env | default( {} ) }}"
```

Also worth mentioning; make sure the group you use is in the inventory your playbook uses; I had an issue whereby my playbook (which I launch with a script) was using a per-project inventory (that did not have the dmzhosts group), while when using plain ansible to test with, I was using a site-wide inventory.  

```bash
ansible -i inventory host1 -m debug -a 'var=proxy_env'
```

## Things to be aware of

This configuration is only present when you SSH in (as yourself). If other people do this they will have to duplicate the configuration in their `~/.ssh/ssh_config`  

For yum repositories, consider that the repositories may be enabled, but won't be reachable. This may disrupt your routine patching. You may like to disable such repositories, or enable `the skip_if_unavailable` option.  
  
Don't be left thinking that this is a particularly beautiful design (actually, outside the limitations of a DMZ in an enterprise environment, I would consider it be a bit yuck, but does have some useful security features -- namely the access is temporary).  
  
Oh, and I've only shown OpenSSH. If you're using anything different for Ansible (eg. Paramiko), you will have to rework how you approach this.
