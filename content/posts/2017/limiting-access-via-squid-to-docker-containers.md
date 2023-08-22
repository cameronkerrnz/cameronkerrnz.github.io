---
title: "Limiting Access via Squid to Docker Containers"
date: 2017-11-02
draft: false
summary: "I wanted to grant just the access needed for different Docker containers through Squid. I used Squid's ability to use Ident as a way to look up a 'user' for a connection, and made a custom identd server to provide this information."
tags: ["Python", "Docker", "Squid"]
---

> This goes into the category of 'it works, but you might rightly question the rightness of it.' It certainly improved our security posture, although there were issues with the Squid configuration. Overall, you should consider this rightly 'archived'.

You have a Docker environment in your network; you limit outgoing traffic from your servers using a whitelisted proxy configuration (Squid) which can then be used for auditing. You wish to allow some containers access to some sites; perhaps even allow some containers access to any site.

How then, can you grant access based on the container? You need to add an extra level of authentication.

You could set up user-credentials for each container that needs access, but that is an administrative burden, and further a lot of software doesn't work well with a proxy that needs user authentication, which is a support concern.

Or we could use ident lookups to determine the 'user', which can then be used to evaluate ACL conditions. To do that, we would need to set up some specialised 'identd' type service on our docker host(s) that was able to return the container name associated with a given source-dest TCP port pair.

This turns out to be entirely feasible, and has the added benefit of adding a level of authentication without the container having to do anything more than just point to the proxy. I found solving this problem to be very instructive and satisfying:

- Lots more Docker experience
- Some more Python experience
- Lots more Systemd experience
- Solves a problem in what I think is an interesting way
- Enables the use of containerised environments to more classes of problems, thereby providing some real business value.

I've posted the code and further information about how this works to Github at [https://github.com/cameronkerrnz/docker-identd](https://github.com/cameronkerrnz/docker-identd)

Have an awesome day, I know I did.
