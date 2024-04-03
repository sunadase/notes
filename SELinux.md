Security-Enhanced Linux (*[SELinux](SELinux.md)*) is a [Linux kernel Security Module](Linux%20Security%20Modules%20%28LSM%29.md) that provides a mechanism for supporting access control security policies, including [Mandatory Access Control (MAC)](Mandatory%20Access%20Control%20%28MAC%29.md).

SELinux is a set of kernel modifications and user-space tools that have been added to various Linux distributions. Its architecture strives to ==seperate enforcement of security decisions from the security policy==, and streamlines the amount software involved with security policy enforcement.

It was originally developed by the United States National Security Agency (NSA) as a series of patches to the Linux kernel using Linux Security Modules (LSM).

A Linux kernel integrating [SELinux](SELinux.md) enforces [Mandatory Access Control (MAC)](Mandatory%20Access%20Control%20%28MAC%29.md) policies that confine user programs and system services, as well as access to files and network resources. Limiting privilege to the minimum required to work reduces or eliminates the ability of these programs and daemons to cause harm if faulty or compromised (for example via buffer overflows or misconfigurations). ==This confinement mechanism operates independently of the traditional Linux [discretionary access control](Discretionary%20Access%20Control%20%28DAC%29.md) mechanisms. It has no concept of a "root" superuser, and does not share the well-known shortcomings of the traditional Linux security mechanisms, such as a dependence on setuid/setgid binaries.==

The security of an "unmodified" Linux system (a system without SELinux) depends on the correctness of the kernel, of all the privileged applications, and of each of their configurations. A fault in any one of these areas may allow the compromise of the entire system. In contrast, the security of a "modified" system (based on an [SELinux](SELinux.md) kernel) depends primarily on the correctness of the kernel and its security-policy configuration. While problems with the correctness or configuration of applications may allow the limited compromise of individual user programs and system daemons, they do not necessarily pose a threat to the security of other user programs and system daemons or to the security of the system as a whole.

From a purist perspective, SELinux provides a hybrid of concepts and capabilities drawn from [mandatory access controls](Mandatory%20Access%20Control%20%28MAC%29.md), *mandatory integrity controls*, *Role-based Access Control (RBAC)*, and type enforcement architecture. Third-party tools enable one to build a variety of security policies.

When an application or process, known as a subject, makes a request to access an object, like a file, SELinux checks with an access vector cache (AVC), where permissions are cached for subjects and objects.

If SELinux is unable to make a decision about access based on the cached permissions, it sends the request to the security server. The security server checks for the security context of the app or process and the file. Security context is applied from the SELinux policy database. Permission is then granted or denied. 

If permission is denied, an "avc: denied" message will be available in /var/log.messages.

##### Features

* Clean seperation of policy from enforcement
* Well-defined policy interfaces
* Support for applications querying the policy and enforcing access control (for example, crond running jobs in the correct context)
* Independence of specific policies and policy languages
* Independence of specific security-label formats and contents
* Individual labels and controls for kernel objects and services
* Support for policy changes
* Seperate measures for protecting system integrity (domain-type) and data confidentiality (*multilevel security*)
* Flexible policy
* Controls over process initialization and inheritance, and program execution
* Controls over file systems, directories, files, and open file descriptors
* Controls over sockets, messages, and network interfaces
* Controls over the use of "capabilities"
* Cached information on access-decisions via the Access Vector Cache (AVC)
* Defualt-deny policy (anything not explicitly specified in the policy is disallowed)

##### Use scenarios

SELinux can potentially control which activities a system allows each user, process, and daemon, with very precise specifications. It is used to confine daemons such as database engines or web servers that have clearly defined data access and activity rights. This limits potential harm from a confined daemon that becomes compromised.
Command-line utilities include: `chcon`, `resorecon`, `runcon`, `secon`, `fixfiles`, `setfiles`, `load_policy`, `booleans`, `getsebool`, `setsebool`, `togglesebool`, `setenforce`, `semodule`, `postfix-nochroot`, `check-selinux-installation`, `semodule_package`, `checkmodule`, `selinux-config-enforcing`, `selinuxenabled`, `selinux-policy-upgrade`

##### Comparison with [AppArmor](AppArmor.md)

SELinux represents one of several possible approaches to the problem of restricting the actions that installed software can take. Another popular alternative is called [AppArmor](AppArmor.md). Because AppArmor and SELinux differ radically from one another, they form distinct alternatives for software control. Whereas SELinux re-invents certain concepts to provide access to a more expressive set of policy choices, AppArmor was designed to be simple by extending the same administrative semantics used for [Discretionary Access Control (DAC)](Discretionary%20Access%20Control%20%28DAC%29.md) up to the mandatory access control level.

There are several key differences:

* AppArmor identifies file system objects by path name instead of inode. This means that, for example, a file that is inaccessible may become accessible under AppArmor when a hard link is created to it, while SELinux would deny acces through the newly created hard link.
  * As a result, AppArmor can be said not to be a *type enforcement* system, as files are not assigned a type; instead, they are merely referenced in a configuration file.
* SELinux and AppArmor also differ significantly in how they are administered and how they integrate into the system
* Since it endeavors to revreate traditional DAC controls with MAC-level enforcement, AppArmor's set of operations is also considerably smaller than those available under most SELinux implementations. For example, AppArmor's set of operations consist of: read, write, append, execute, lock and link. Most SELinux implementations will support numbers of operations orders of magnitude more than that. For example SELinux will usually support sthose same permissions, but also includes controls for mknod, binding to network sockets, implicit use of POSIX capabilities, loading and unloading kernel modules, various means of accessing shared memory, etc.
* There are no controls in AppArmor for categorically bounding POSIX capabilities. Since the current implementation of capabilities contains no notion of a subject for the operation (only the actor and the operation) it is usually the job of the MAC layer to prevent priviliged operations on files outside the actor's enforced realm of control (i.e. "Sandbox). AppArmor can prevent its own policy from being altered, and prevent file systems from being mounted/unmounted, but does nothing to prevent users from stepping outside their approved realms of control
  * For example, it may be deemed beneficial for help desk employees to change ownership or permissions on certain files even if they don't own them (for example, on a departmental file share). The administrator does not want to give the user(s) root access on the box so they give them `CAP_FOWNER` or `CAP_DAC_OVERRIDE`. Under SELinux the administrator (or platform vendor) can configure SELinux to deny all capabilities to otherwise unconfined users, then create confined domains for the employee to be able to transition into after logging in, one that can exercise those capabilities, but only upon files of the appropriate type.
* There is no notion of multilevel security with AppArmor, thus tere is no hard BLP or Biba enforcement available
* AppArmor configuration is done using solely regular flat files. SELinux (by default in most implementations) uses a combination of flat files (used by administrators and developers to write human readable policy before it's compiled) and extended attributes.
* SELinux supports the concept of a "remote policy server" (configurable via /etc/selinux/semanage.conf) as an alternative source for policy configuration. Central management of AppArmor is usually complicated considerably since administrators must decide between configuration deployment tools being run as root (to allow policy updates) or configured manually on each server

Multi-Category Security (MCS) is an enhancement to SELinux for Red Hat Enterprise Linux that allows users to label files with categories, in order to further restrict access through discretionary access control and type enforcement. Categories provide additional compartments within sensitivity levels used by multilevel security (MLS).

srcs:

* https://www.redhat.com/en/topics/linux/what-is-selinux
* https://en.wikipedia.org/wiki/Security-Enhanced_Linux
