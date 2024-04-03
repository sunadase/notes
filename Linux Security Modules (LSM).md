The Linux Security Module (LSM) framework provieds a mechanism for various security checks to be hooked by new kernel extensions. The name "module" is a bit of a misnomer since these extensions are not actually loadable kernel modules. Instead, they are selecetable at build-time via CONFIG_DEFAULT_SECURITY and can be overridden at boot-time via the "security=..." kernel command line argument, in the case where multiple [LSM](Linux%20Security%20Modules%20%28LSM%29.md)s were built into a given kernel.

The primary users of the [LSM](Linux%20Security%20Modules%20%28LSM%29.md) interface are [Mandatory Access Control (MAC)](Mandatory%20Access%20Control%20%28MAC%29.md) extensions which provide a comprehensive security policy. Examples include [SELinux](SELinux.md), *Smack*, *Tomoyo*, and [AppArmor](AppArmor.md). In addition to the larger MAC extensions, other extensions can be built using the [LSM](Linux%20Security%20Modules%20%28LSM%29.md) to provide specific changes to system operation when these tweaks are not available in the core functionality of Linux itself.

The Linux capabilities modules will always be included. This may be followed by ***any number of "minor"*** and ****at most one "major" module****. For more details on capabilities, see ==`capabilities(7)`==

A list of the active security modules can be found by reading ==/sys/kernel/security/lsm==. This is a comma seperated list, and will always include the capability module. The list reflects the order in which checks are made. The capability module will always be first, followed by any "minor" modules (e.g. *Yama*) and then the one "major" module (e.g. [SELinux](SELinux.md)) if there is one configured. 

Process attributes associated with "major" security modules should be accessed and maintained using the special files in ==/proc/.../attr==. A security module may maintain a module specific subdirectory there, named after the module. The files directly in /proc/.../attr remain as legacy interfaces for modules that provide subdirectories

* [AppArmor](AppArmor.md)
* *LoadPin*
* [SELinux](SELinux.md)
* *Smack*
* *TOMOYO*
* *Yama*
* *SafeSetID*

srcs:

* https://docs.kernel.org/admin-guide/LSM/index.html
