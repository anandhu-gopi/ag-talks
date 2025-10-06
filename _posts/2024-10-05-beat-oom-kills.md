---
layout: single
title: "🥊 Beat OOM Kills"
date: 2024-10-05
read_time: true
comments: true
share: true
tags: [kubernetes, oom, memory, containers, devops]
categories: [DevOps]
excerpt: "Strategies to handle OOM kills in Kubernetes and prevent brutal process termination."
author_profile: true
---

![Beat OOM Kills]({{ site.url }}/assets/images/beat-oom-kills/beat.jpeg)

Most people who have worked with Kubernetes have likely encountered the infamous OOM Kill at least once. In cases like when your process exceeds its memory limit, it receives a SIGKILL from the oom killer and is brutally terminated, often without any opportunity for cleanup.

While you can't instruct Kubernetes to send a SIGKILL instead of SIGTERM (consider SIGTERM a polite way to terminate a process, allowing it some time for cleanup, whereas SIGKILL is a forceful termination) there are several strategies you can adopt to send warning signals before an OOM situation is likely to happen

1. **Use a Sidecar Container**: Go with a sidecar container that monitors your memory usage and sends signals when thresholds are breached. ([https://lnkd.in/gqvZrZ9v](https://lnkd.in/gqvZrZ9v))

2. **Write a DaemonSet**: Alternatively, you could create a DaemonSet that monitors the memory usage of your deployments and sends signals accordingly.

With relevant tools to monitor and alert your process memory usage, you can make your application respond to these warning signals, allowing for cleanup before the OOM killer terminates your process. Writing signal handlers for these warning signals is quite straightforward. For instance, in the Python world, you can use the signal library to achieve this. (I am attaching sample code that handles the SIGUSR1 signal.)

P.S: As long as the OOM killer continues sending SIGKILLs, unfortunately, we have to look for workarounds like the ones mentioned above. If you have any other strategies to tackle this, please share in the comments!

---

[Original post from Linkedin](https://www.linkedin.com/posts/anandhu-gopi-691b35144_beat-oom-kills-most-people-who-have-activity-7248299846984474624-vjGj/)
