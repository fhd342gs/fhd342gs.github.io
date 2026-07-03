---
title: "About"
date: 2026-03-14
draft: false
---

## ioctl(fd, WHOAMI, *argp)

There's a syscall buried in every UNIX kernel that doesn't play by the rules. No clean abstractions, no polite interfaces — just a raw channel straight into the guts of the machine. `ioctl()` is how you talk to device drivers when `read()` and `write()` aren't enough. It's the trapdoor into ring 0.

This site is a collection of walkthroughs, notes, and things I've learned along the way.
