---
layout: post
title: Why do I use Linux for .NET development?
---

The question of OS and IDE choice for .NET development comes up from time to time. In this post I'll summarize my thoughts on the topic.

# Performance

My main reason for choosing a certain tool (software or hardware) is productivity and performance.

As Joel Spolsky said in [his famous "12 steps to better code" article](https://www.joelonsoftware.com/2000/08/09/the-joel-test-12-steps-to-better-code/), 
"always use the best tool money can buy".

And Linux is simply faster in every way than Windows (I mention Apple products down below):
* Git is a lot faster, because it was designed for Linux first. This is very noticeable on large repos.
  * TBD: Branch switch benchmark
* Docker works natively, without a VM. This is a huge difference in performance and usability. IO within containers is as fast as on the host. Memory usage is lower. Startup times are lower.
* File system is faster. And programming is all about files. IDE startup, indexing, build times, everything is affected.
* OS updates are faster. A typical update on Ubuntu runs for 20 seconds and does not require a restart. Windows is much worse, it nags you for hours and then reboots your machine.

# Usability

The browser and the IDE look and work the same on Windows/Linux/macOS, and those are the only tools most developers require for their work. Terminal is pretty much the same, too.

The rest does not matter, I don't see a big difference across OSes. 

# Why not Apple

* Recent ARM hardware is great, but it is not faster than top Intel laptops as of 2023. Yes, Apple Silicon has much lower power consumption and better battery life, but my laptop never leaves the desk, so it does not matter.  
* Docker runs in a VM, similar to Windows, leading to increased memory usage, slower startup times, slower IO.
* ARM can still be an issue for certain software.
