---
layout: single
title: "Dumb Init and Docker: Why You Might or Might Not Need It"
date: 2024-10-13
author_profile: true
read_time: true
comments: true
share: true
categories: [DevOps]
tags: [docker, init, zombies, dumb-init, containers]
excerpt: "Zombie processes, PID 1 responsibilities, and when to use dumb-init in Docker."
---

![Dumb Init]({{ site.url }}/assets/images/dumb-init/dumb-init.jpeg)

Have you ever heard of the zombie reaping problem or the need for an init system in Docker containers? If not, let me explain why it’s something you should consider next time you're running or building Docker images.

1. Responsible Parent Processes and Child Processes:
In Linux systems, when a parent process spawns a child process, it must "wait" for the child to finish, in order to collect its exit status. This is similar to how a responsible parent checks on what happens to its child. Once the child process is done, the parent process collects its exit status by calling wait

2. Irresponsible Parent Processes:
Now, imagine a scenario where the parent process spawns a child but doesn’t wait for it. Perhaps the parent process dies before the child finishes (e.g., it was stopped or completed its own work). In this case, the child process can turn into a zombie process

3. Zombie Process?
A zombie process is a child process that has completed execution but its parent hasn’t collected its exit status. The Linux system keeps an entry for such processes in the process table, allowing the parent to call wait() later to retrieve the exit status.

4. Zombie Processes in Docker Containers
When your container’s main process and its child processes die, if the main process didn’t wait for the child, a zombie process is created. and these zombie processes will appear in the host's Linux kernel process table (since containers are isolated processes using the same kernel and resources)

5. Filling Up the Process Table
Although zombie processes don’t resources, they still take up entries in the process table. If you have multiple containers creating zombie processes, these entries will accumulate. Eventually, if the process table fills up, Linux won’t be able to create new processes, blocking you from spinning up new, live ones.

6. The Fix: Use an Init System
If your container’s main process can’t wait for its child processes, a simple and lightweight init system like dumb-init can handle it.

7. Alternatives to Using Dumb Init
- Using bash as PID 1: You can use bash to run your process since bash can reap zombie processes. but, there’s a caveat—bash may not forward signals (like SIGTERM) to your app's child processes. This can prevent your application from performing graceful shutdowns and clean-up tasks

- Running One Process Per Container/Make your PID-1 take care of your reaping zombies

In Closing:
Sometimes, fortunately/unfortunately you might run third-party applications in Docker containers. These applications may spawn child processes but may not properly wait for them. In such cases, it’s essential to use an init system like dumb-init or tini. Check out these articles from the reference to know more 

References:
- [https://lnkd.in/gn3rd5Tk](https://lnkd.in/gn3rd5Tk)
- [https://lnkd.in/gFC2pY5k](https://lnkd.in/gFC2pY5k)


[Original post from Linkedin](https://www.linkedin.com/posts/anandhu-gopi-691b35144_dumb-init-and-docker-why-you-might-or-might-activity-7251219245135503360-3_HH)
