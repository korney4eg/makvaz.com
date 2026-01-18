---
layout: post
title: Book "Learning eBPF"
draft: false
archives: "2026"
tags: [books, ebpf]
---
<img src="/2026/01/20/book-learning-ebpf/book-cover.png" alt="" width="220"
  style="float: left; margin-right: 15px">

_I always knew that eBPF was one of those technologies hidden deep inside the
kernel, far away from everyday users, but incredibly powerful._

_At work, I learned that Google Kubernetes Engine clusters using Dataplane V2
rely heavily on eBPF under the hood. That felt like a good opportunity to
finally dive into something new, so I grabbed the book **Learning eBPF** from the
office bookshelf._

_This post is my review of that book._
<!--more-->

At the beginning, [Liz Rice](http://www.lizrice.com) covers the history of eBPF
as a technology and explains its core working principles. What impressed me the
most is that eBPF allows you to load programs into the Linux kernel without
recompiling the kernel itself and without rebooting the operating system.
Thanks to the built-in `verifier`, it is much harder to load a program that
could harm the system.

After the introduction, the book moves quickly to practice. We write our first
“Hello, world”–style program in Python, which loads and runs eBPF code in the
kernel. The next step is writing eBPF programs in C, compiling them, and
attaching them to the kernel manually.

Step by step, the book dives deeper into different aspects of eBPF and how it
works in practice. Closer to the end, the author shows how eBPF is used in
Kubernetes networking and how modern security tools rely on this technology.

This is one of my favorite types of technical books: focused, with no unnecessary
fluff, but full of concrete examples and hands-on practice. The book is written
in clear and accessible language, although I can’t remember the last time I felt
so intellectually exhausted while learning something new.

If you are a power user of Linux systems, comfortable with installing and
compiling software, but have always been a bit afraid to explore what is
happening at the kernel level, I can definitely recommend this book. Moreover,
it is [available for free](https://cilium.isovalent.com/hubfs/Learning-eBPF%20-%20Full%20book.pdf).
