Traditionally, Linux and UNIX systems have used [Discretionary Access Control (DAC)](Discretionary%20Access%20Control%20%28DAC%29.md). [SELinux](SELinux.md) is an example of a [Mandatory Access Control (MAC)](Mandatory%20Access%20Control%20%28MAC%29.md) system for Linux. 

With DAC, files and processes have owners. You can have the user own a file, a group own a file, or other, which can be anyone else. Users have the ability to change permissions on their own files.

The root user has full access control with a DAC system. If you have root access, then you can access any other user’s files or do whatever you want on the system. 

But on MAC systems like [SELinux](SELinux.md), there is administratively set policy around access. Even if the DAC settings on your home directory are changed, an [SELinux](SELinux.md) policy in place to prevent another user or process from accessing the directory will keep the system safe. 

[SELinux](SELinux.md) policies let you be specific and cover a large number of processes. You can make changes with [SELinux](SELinux.md) to limit access between users, files, directories, and more.

srcs:

* https://www.redhat.com/en/topics/linux/what-is-selinux
