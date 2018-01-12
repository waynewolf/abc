---
title: how binder transfer fd across process boundary
date: 2014-01-11 20:20:45
tags:
- android
---

## Problem
Android ashmem can map file descriptor to share memory across process boundary. Android binder help transfer fds between different processes, this is very basic and important infrastructure for graphic buffer sharing in android. However, fd cannot be sent across processes by socket without extra code, how does binder do this?

<!--more-->
## Principle
fd cannot be shared between processes, but kernel `struct file *` pointer can.

## Sample Code
    struct file *file = fget(fd_in_process_a);
    int fd_in_process_b = task_get_unused_fd_flags(target_proc, O_CLOEXEC);
    task_fd_install(target_proc, fd_in_process_b, file);

The second process(target_proc) installs the fd, which is associated with `struct file *` used in the first process.

## Alternatives
1. sendmsg/recvmsg only on unix domain socket,  cmsg_type set to SCM_RIGHTS，additional permission needed.
2. stream based pipe, ioctl I_SENDFD, seems deprecated on Linux, not sure.

## Reference:
<http://infohost.nmt.edu/~eweiss/222_book/222_book/0201433079/ch17lev1sec4.html>

