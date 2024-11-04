---
title: "Bloom Filter Lightning Talk"
date: 2024-11-04T20:03:55+13:00
draft: false
summary: A Lightning talk I gave to Dunedin Code Craft in November 2024
tags: ['Talking', 'Performance', 'Programming', 'Software Development']
---

I regularly listen to the Security Now podcast, and really enjoyed a recent episode on [Cascading Bloom Filters](https://twit.tv/shows/security-now/episodes/989). Having used Bloom Filters a little in the past, I thought it would be a great topic for a Lightning Talk on the topic, since I believe I have a good idea how to communicate the concept easily.

<iframe src="https://docs.google.com/presentation/d/e/2PACX-1vQikQQf4yaVmYPUmoP-EXrBkUDzB3cKnYglXT8H6LrLzUW4GzZXWGGREZ-YQpJKcSSlFbGhQI6UpvCP/embed?start=false&loop=true&delayms=3000" frameborder="0" width="960" height="440" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true"></iframe>

While trying to use the CRLite functionality for myself (and failing), I had to do some spelunking in the Firefox code management system; see the speakers notes for some details. Looks like the current behaviour (November 2024) is to use CRLite for the definate case (not revoked), and immediately fall back to OCSP for the 'likely revoked' case.

Fingers crossed I can fit this in to however many minutes I get for this Lightning Talk.
