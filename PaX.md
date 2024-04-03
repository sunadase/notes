*Kernel Hardening*

PaX is a patch to the Linux kernel that provides hardening in three ways:

1. Judicious enforcement of non-executable memory
1. Address Space Layout Randomization (ASLR)
1. Miscellaneous hardening on stack- and memory handling

##### Judicious enforcement of non-executable memory

Judicious enforcement of non-executable memory prevents a common form of attack where executable code is inserted into the address space of a process by an attacker and then triggered, thus hijacking the process and possibly escalating privileges. The typical vector for the insertion is via user provided data that finds its way into executable memory. By ensuring that "data" only lives in memory which is non-executable, and that only "text" is found in memory which is executable, PaX preemptively protects against this class of attacks.

##### Address Space Layout Randomization (ASLR)

With Address Space Layout Randomization (ASLR), a randomization of the memory map of a process (as reported, for example, by pmap) is provided and thus makes it harder for an attacker to find the exploitable code within that space. Each time a process is spawned from a particular ELF executable, its memory map is different. Thus exploitable code which may live at 0x00007fff5f281000 for one running instance of an executable may find itself at 0x00007f4246b5b000 for another. While the vanilla kernel does provide some ASLR, a PaX patched kernel increases the randomization. Furthermore, when an application is built as a Position Independent Executable (PIE), even the base address is randomized. 

##### Miscellaneous hardening

Finally, the PaX patches provide some miscellaneous hardening: 

* erasing the stack frame when returning from a system call 
* refusing to dereference user-land pointers in some contexts 
* detecting overflows of certain reference counters 
* correcting overflows of some integer counters 
* enforcing the size on copies between kernel and user land 
* providing extra entropy.

More information on PaX can be found on its [official homepage](https://pax.grsecurity.net/).

The first step in working with PaX is to configure and boot a PaX patched kernel. Depending on whether or not one has configured PaX for SOFTMODE or non-SOFTMODE, the kernel will automatically start enforcing memory restrictions and address space randomization on all running processes.

* With **SOFTMODE** enabled, PaX protection will not be enforced by default for those features which can be turned on or off at runtime, so this is the "permit by default" mode. In SOFTMODE the user must explicitly mark executables to enforce PaX protections.
* Without **SOFTMODE** (the **non-SOFTMODE** approach), PaX protections are immediately activate ("forbid by default"). The user must explicitly mark binaries to relax PaX protections selectively.
  Ideally it should not be necessary to do anything else; however, for problematic executables in non-SOFTMODE, a second step is required: relax certain PaX restrictions on a per ELF object basis. This is done by tweaking the PaX flags which are read by the kernel when the ELF is loaded into memory and execution begins. This second step is usually straight forward except when the ELF object that requires the relaxation is a library. In that case, the library's flags have to be back ported to the executable that links against it. When PaX enforces or relaxes a feature, it does so on the basis of the executable's flags, not those of the libraries it links against. Both steps will be discussed in detail below, but first an overview of PaX's features will be presented.

srcs:

* https://pax.grsecurity.net/
* https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project
* https://wiki.gentoo.org/wiki/Hardened/PaX_Quickstart
