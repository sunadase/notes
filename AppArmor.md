AppArmor ("Application Armor") is a [Linux kernel security module](Linux%20Security%20Modules%20%28LSM%29.md) that allows the system administrator to restrict programs' capabilities with per-program profiles. Profiles can allow capabilities like network access, raw socket access, and the permission to read, write, or execute files on matching paths. [AppArmor](AppArmor.md) supplements the traditional Unix [discretionary access control (DAC)](Discretionary%20Access%20Control%20%28DAC%29.md) model by providing [mandatory access control (MAC)](Mandatory%20Access%20Control%20%28MAC%29.md). It has been partially included in the mainline Linux kernel since version 2.6.36 and its development has been supported by Canonical since 2009.

#### Details

In addition to manually creating profiles, AppArmor includes a learning mode, in which profile violations are logged, but not prevented. This log can then be used for generating an AppArmor profile, based on the program's typical behaviour.

AppArmor is implemented using the [Linux Security Modules (LSM)](Linux%20Security%20Modules%20%28LSM%29.md) kernel interface.

AppArmor is offered in part as an alternative to SELinux, which critics consider dificult for administrators to set up and maintain. Unlike SELinux, which is based on applying labels to files, AppArmor works with file paths. Proponents of AppArmor claim that it is less complex and easier for the average user to learn than SELinux. They  also claim that AppArmor requires fewer modifications to work with existing systems. For example SELinux requires a filesystem that supports "security labels", and thus cannot provide acces control for files mounted via NFS. AppArmor is filesystem-agnostic.

#### Other systems

[SELinux](SELinux.md) system generally takes an approach similar to AppArmor. One important difference: SELinux identifies file system objects by inode number instead of path. ==Under AppArmor an inaccessible file may become accessible if a hard link to it is created.== This difference may be less important than it once was, as Ubuntu 10.10 and later mitigate this with a security module called *Yama*, which is also used in other distributions. SELinux's inode-based model has always inherently denied access through newly created hard links because the hard link would be pointing to an inaccessible inode

SELinux and AppArmor also differ significantly in how they are administered and how they integrate into the system.

Isolation of processes can also be accomplished by mechanisms like virtualization; the One Laptop per Child (OLPC) project, for example, sandboxes individual applications in lightweight Vserver.

In 2007, the Simplified Mandatory Access Control Kernel was introduced.

In 2009, a new solution called *Tomoyo* was included in Linux 2.6.30; like AppArmor, it also uses path-based access control.
