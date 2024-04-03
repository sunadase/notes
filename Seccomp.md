short for secure computing is a computer security facility in the Linux kernel
seccomp allows a process to make a one-way transition into a "secure" state where it cannot make any system calls except `exit()`, `sigreturn()`, `read()` and `write()` to *already open* file descriptors. Should it attempt any other system calls, the kernel will either just log the event or terminate the process with SIGKILL or SIGSYS. In this sense, it does not virtualize the system's resources but isolates the process from them entirely

There are two ways to activate seccomp: through `prctl(2)` syscall with PR_SET_SECCOMP, or for Linux kernels 3.17 and above, the `seccomp(2)` syscall. The older method of enabling seccomp by writing to /proc/self/seccomp has been deprecated in favor of `prctl()`

seccomp-bpf is an extension to seccomp that allows filtering of system calls using a configurable policy implemented using *Berkeley Packet Filter* rules. It is used by OpenSSH and vsftpd as well as the Google Chrome/Chromium web browsers on ChromeOS and Linux.  (In this regard seccomp-bpf achieves similliar functionality, but with more flexibility and higher performance, to the older systrace-- which seems to be no longer supported for Linux)

Some consider [seccomp](Seccomp.md) comparable to OpenBSD's `pledge(2)` and FreeBSD's `capsicum(4)`

srcs:

* en.wikipedia.org/wiki/Seccomp
* https://man7.org/linux/man-pages/man2/seccomp.2.html
* https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/seccomp
