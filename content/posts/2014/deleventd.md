---
title: "(Ab)using Samba and inotify to implement simple menu of privileged actions"
date: 2014-04-29
draft: false
summary: "When you want SSH and sudo, but all you have is a Samba share. Similar to using a flag file in a network share to trigger some scripted action, this solution uses inotify to watch for files that get deleted, and then even shows the status of each job."
tags: ["Automation", "Shift-Left", "Python"]
---

## Part 3: Demonstration

(Yes, I'm starting at the exciting bit, and why not?)

{{<youtube B2L1sjK1z6s>}}

Here's the gist of the video. In the video you can see a bunch of files in a Samba network share. Each of these files is like a flag-file, but the filename also contains a status, like 'ready', 'in-progress', etc. When you *delete* a file the job will start; a new version of the flag file will be created showing the current status. Eventually the job completes and the the flag file is deleted by the system and replaced with its 'ready' version.

## Part 1: Design Analysis

I expect there will be at least one other part that covers the implementation, and another part covering how to use it.

Let's say you offer some form of software as a service to customers, such as a website with a database, middle-ware and web tier. In order to limit exposure, you have a policy not to allow console access via tools such as SSH or RDP. You might instead offer access to various directories using tools such as Samba, and perhaps remote access to the database (over SSL) if required. Samba could also provide access to the logs.

> Note from 2023: when I say "customers", think like a small group of departmental users.

Ah, but if someone has access to change something in a configuration, such as in the application; how then would they restart things? A few options come to mind. The first might be some restricted access via SSH where the user is forced into a menu-driven interface. Another might be some web-interface (such as cPanel). Those would be the obvious contenders, let's look at each before deciding if it is worth looking at something outside the square.

A limited access using SSH is interesting, but difficult to securely implement if the intent is to prevent shell access. Remember that a user can specify a command from the client, and modifying a user's shell would be impractical if the account details are centralised in LDAP.

A web-based solution seems like a lot of extra work, and would be undesirable for restarting the web tier. Additionally, in the case of Apache, what you implement the solution in may be incompatible with the primary application (eg. mod_python and mod_php would prefer a different Apache MPM).

Setting up a separate web server on a separate port would do a lot to resolve those issues, but you'd want to have a different web server in place, so as to keep the configuration and runtime support state separate and non-conflicting.

Aside: Maybe this would be a good reason to revisit a self-contained web-server solution; something like the Yesod web-programming framework for Haskell, or perhaps if Python is more your thing, a bit of Green Unicorn? Just so long as it does SSL well, authenticates with your usual kit, does sane access control and auditing, and is understandable by the rest of your team... and runs on all your supported platforms (so if RHEL5 is a requirement, that would mean Python 2.4). That might be a big ask for a glorified set of Big Red Switches.

Surely there must be an easier way that would allow us to do something as throw in a flag file which indicates "restart web tier" or such, and has ready access to logs etc., and some reasonably mature method of doing authentication and access control?

But wait, I do, and I'm already using it. Samba!

I'm not a big fan of Samba or CIFS or SMB in general, but in this case, it gets the job done as well as can be expected. So here's the idea: create a Samba share what will be used for the storage of flag files. Some agent on the server will then observe the state of flag files and run some action in response.

So some initial design points: what is the most correct way of implementing such flag files, and how can we implement them with some degree of interactivity?

Flag files could take be implemented as edge-triggered or level-triggered. If the action is to be triggered only once then level-triggered would require some additional locking construct. If edge-triggered flag-file actions are used, then the edge could be take a number of forms, such as file creation, file rename, or file deletion.

We have to be careful there though; our architectural underpinning may be undone with crayons such as editors, caching, WAN optimisation, client implementation funkiness in a mixed-vendor environment (read, Mac OS X), resource-forks and various indexes, anti-virus software, backup and file-replication, Unicode normal-forms... oh, and things like trash.

Do we dear to tread in this direction then? Yes, all things considered, if we select an appropriate edge, it should be manageable. So then, which edge?

If we take file creation as an edge, then that means we have to know which file to create, which lessens the discoverability of the interface, and potentially increases user-support requirement. When the task is completed the flag-file might the. Be removed... this might happen rather suddenly so could be an avenue for user confusion. It's less simple compared to typing a command, as it also depends in how the user creates the new file, and windows has a annoying trait of putting extensions on things, and not showing them (myfile.dat.txt)

If we take file deletion as an edge, then that means people could easily see the available edges. Delete the file and the corresponding action fires, and the flag file gets recreated.... which again has a potential avenue for user confusion. It is about as simple as pushing a button though. One nice thing about this is that it is (almost) implicitly synchronised. One limitation is that it doesn't provide for much of a use interface.

What about renaming or moving a file? Moving a file brings a user-interface closer to drag-and-drop. Imagine a folder or actions, with another folder called something like 'execute' or 'run'. Dragging available flag-files into such a directory would execute the corresponding action. It also allows for the file to always exist thougout the operation. The file, if named carefully, could bear some form or status. The content of the file might even be useful for storing action output.

### User-Interface design

Let's mock up the UI (perhaps now would be a good time to give this form of user-'interface' an acronym... FUI?)

So imagine a share, named Controls or some such, which we keep distinct from the rest of the data shares so we can implement different share settings, which provides for a level of authorisation and accounting. Inside the share are the following:

- Execute/
- Restart_Apache
- Restart_Tomcat
- Redeploy_Application
- Snapshot_Application
- Summon_Support

Let's say the user drags Restart_Apache into the Execute folder. The folder structure now looks like this:

- Execute/Restart_Apache
- Restart_Tomcat
- Redeploy_Application
- Snapshot_Application
- Summon_Support

Some file-system watcher would notice that the file has been moved into the Execute folder, lookup the appropriate action for that flag-file, and run it.

It could then move the file back to where it was, but with a file name showing it has a new state. The user would hopefully see this immediately after dragging and dropping.

- Execute/
- \[Running\]_Restart_Apache
- Restart_Tomcat
- Redeploy_Application
- Snapshot_Application
- Summon_Support

When the action finishes, the file should be further renamed. Perhaps the `[Running]` could be replaced with some resultant moniker such as `[Done]` or `[Fail]`. Bracketing monikers would help to separate it from the action identifier in the flag file.

Hmm, having any state moniker at the beginning of the file name would affect sorting, which is likely to be undesirable. Having it at the end could affect usability if the end if the entire file name is not shown due to display truncation; it could appear as if the file wasn't moved.

However, eventually all flag-files will have `[Done]` or such, and so either there should be some other invocation-count moniker or some invocation date-time moniker, or the file should be renamed to remove the resultant moniker after some period. Date-times get unwieldy, and an invocation count is kinda meaningless (who would notice what it was before executing?) Having a invocation time (no date) moniker that was removed after some delay could be useful, but you still be issues of timezone differences, and DST changes to boot. Erase-resultant-moniker-post-delay seems to be the best.

If we change the flag-file name to have a .txt extension, then that file could have [a copy of] the action output and history in its content.

Wait, I just realised something. My previous list of actions was too small and led me astray. Consider the following three actions:

- Execute/
- Start Apache
- Stop Apache
- Restart Apache

The key thing to realise is that they are mutually exclusive and that they need synchronised. Let's get our state (objects) separated from out actions (methods). How might a refactor of that look?

- Start/
- Stop/
- Restart/
- Apache
- Tomcat

Now each logical object in the system is only represented by one flag-file. The state-moniker would have to change to being a method moniker.

We get two issues further to deal with though: what if an object doesn't support the method you invoke on it? It could instead be passed to a some default method which rendered a result-moniker of `[Blocked]` or similar.

The other issue is that we haven't really solved the synchronisation issue, because now we have to make sure that we don't drag to start, and then to stop before its ready. We can address that by checking for an existing method moniker when processing a drag/move.

Before I forget about it, another type of drag/move would be to move it out of the folder/share. This type of behaviour will need to be caught in order to restore the state. It is an example of why system state must not be stored (reflected is okay) in the FUI.

Let's revisit that list further above and redraw for the new version:

- Restart/
- Redeploy/
- Snapshot/
- Summon/
- Apache
- Tomcat
- Application
- Support

That won't scale well. Perhaps the objects should be modelled as directories.

- Apache/Start
- Apache/Stop
- Apache/Restart
- Tomcat/Start
- Tomcat/Stop
- Tomcat/Restart
- Application/Redeploy
- Application/Snapshot
- Support/Summon

That seems much cleaner. It also has an easy serialisation mechanism: when one action is called, the other actions in a group can be deleted, or have their state changed, by the watcher. When the action completes, the actions become available again.

### How to implement

Okay, so after a few iterations if design analysis, I think I know what the end-goal looks like. But how to get there?

Watching for file system changes would be inotify driven, and RHEL 6 has a python inotify library available. (TODO Check RHEL 5). Using inotify will be Linux specific, but that is completely acceptable for this project.

I'm doing more things in Python these days, and our development team is using more Python, so we have some useful in-house expertise to fall back on and provide some useful code review and learning opportunities. Main drawback with Python is that if I want this to work on RHEL 5, then that means using Python 2.4 (TODO: does RHEL5 support inotify?). Whereas if I use Perl, which I also know, then then version difference in RHEL 5 and 6 is not so great. I do prefer to avoid Perl these days though (largely due to nested data-structure sigil pain).

Configuring the behaviour would likely be done using Yaml, which I've used previously and enjoyed. Ideally, this will turn into a nicely reusable tool I could package and deploy.

Another alternative, as is done in the Python/Django world, would be to write the configuration in Python. I'm not sure I really like this approach; while the input would be coming from the trusted administrator, and could gain a lot of integration flexibility, it could well make it a bit harder to report on configuration errors.

The FUI agent would need to be able to do privileged operations, but the watcher and executor could be separate, and forced to go through sudo. The watcher should be implemented as a service, so should run in the background.

Access control would be done with share-level permissions, but file-system permissions could also be used if configured by the FUI agent.

If SElinux is in enforcing mode, then some policy work may be required. From a security standpoint, the only user input that the monitor would be exposed to would be the naming of the files, and ensuring that the flag files are actual files and not something like symbolic links, so the security footprint isn't very large, and all action invocation should be going through sudo. We will need to be creating flag-files and potentially setting ownership on them, so running as root should be acceptable, but not a requirement if setting ownership is not needed.

Note that Samba is incidental to this (although useful for providing share-level access-control). It's role can begin when testing.

So now we have an idea of what our FUI will look and feel like, and some rough idea of how we might go about implementing it, but how to configure it in a way that will make future deployment a pleasure, and prevent problems that might lead to service issues?

The first observation is that order of definition shouldn't matter when defining flag-files and actions, although they may do if groups of actions are used (a group being a set of actions that require serialisation). The configuration should never be terribly large or complex (if it is, then its probably time to look at something a bit larger). To assist in deployment and configuration management, a directory of configuration elements should be used (eg. /etc/cron.d/), from which files with only a known extension would be combined to make a whole configuration. Here is what such a snippet might look like.

```yaml
#
# Standard Apache httpd actions
#
object: Apache
    #
    # When an action runs, all other actions in that action-group get disabled until that
    # action completes.
    #
    # An action group would also be used to support different permissions or locations,
    # which could be useful in supporting multiple user-groups.
    #
    action-group:
        directory:
            path: /var/local/flagfile-actions/
            user: root
            group: wwwadmins
            mode: 0770 # Ideally allow deletion and not creation...
        file:
            user: root
            group: wwwadmins
            mode: 0640 # These permissions are less important, but still useful
        command:
            # These should be seen as defaults for the actions in this action-set,
            # which could be overridden in an individual action.
            user: root
            group: root
        actions:
            - action:
                  name: Start
                  command: /sbin/service httpd start
                  user: root # pointless example of how to override the default
                  group: root
            - action:
                  name: Stop
                  command: /sbin/service httpd stop
            - action:
                  name: Restart # case insensitive, but case-preserving
                  command: /sbin/service httpd restart
```

## Part 2: Proof-of-Concept

This was my proof-of-concept

```python
#!/usr/bin/env python

import pyinotify
import os
import time
from threading import Timer
import shlex
import subprocess

trigger_directory = '/home/cameron/tmp/fui/triggers/'
command = r''' /bin/echo 'Oh my gosh it was deleted' '''

def remove_resultant_moniker(trigger):
    print "Removing resultant moniker from ", trigger

class EventHandler(pyinotify.ProcessEvent):
    def process_IN_DELETE(self, event):
        print "Removing ", event.pathname
        args = shlex.split(command)
        print "Args: ", args
        subprocess.Popen(args)
        Timer(2.0, remove_resultant_moniker, ['TODO']).start()
    def process_IN_CREATE(self, event):
        print "Created ", event.pathname
        Timer(2.0, remove_resultant_moniker, ['TODO']).start()

mask = pyinotify.IN_DELETE | pyinotify.IN_CREATE

watch_manager = pyinotify.WatchManager()

handler = EventHandler()
notifier = pyinotify.Notifier(watch_manager, handler)
wdd = watch_manager.add_watch(trigger_directory, mask, rec=True)
notifier.loop()
print 'Ending'
```

Running the above in a terminal, and in another terminal run:

```bash
rm -f triggers/deleteme && sleep 3 && touch triggers/deleteme
```

in another window, I get the following output (with output appearing at the times I expect)

```plain
Removing  /home/cameron/tmp/fui/triggers/deleteme
Args:  ['/bin/echo', 'Oh my gosh it was deleted']
Oh my gosh it was deleted
Removing ressultant moniker from  TODO
Created  /home/cameron/tmp/fui/triggers/deleteme
Removing ressultant moniker from  TODO
```

I haven't done anything with Yaml at the moment, its too early for that. The next step is to verify that this works when the user deletes the trigger via SMB / CIFS. I'm already confident that it won't work if the trigger files are stored on SMB/CIFS, as Linux doesn't have inotify support for that. Samba should be able to pick up the changes (I hope) and (with a client that support Directory Change Notifications) have the client reflect any new state.

> Note from 2023: I see I never completed the last part of this blog series. I did actually complete the code, and it is shown running in the video at the start. Alas, I never did publish the code, but it did work reasonably well over Samba and was problem free. I think it's a nice spin on the flag-file concept, but you could easily outgrow this ... dare I call it a solution?
