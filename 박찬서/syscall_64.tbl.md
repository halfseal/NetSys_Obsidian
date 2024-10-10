---
Parameter: 
Return: 
Location: /arch/x86/entry/syscalls/syscall_64.tbl
---

```c title=syscall_64.tbl
#
# 64-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point>
#
# The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls
#
# The abi is "common", "64" or "x32" for this file.
#

0	common	read			sys_read

23	common	select			sys_select

213	common	epoll_create		sys_epoll_create

232	common	epoll_wait		sys_epoll_wait
233	common	epoll_ctl		sys_epoll_ctl

```


[[read()]]
[[select()]]
