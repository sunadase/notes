# My Document


bazen bazi makienlerde network cok onemli oluyor. diyelim ki bu makine
de bir adet network karti var. ve biz tum bu network kartinin bir
prosese adanmasini istiyoruz. ve diger yavaslik kaynagi olan her seyden
kurtulmak istiyoruz.performans icin hangi metodlari kullanilir. ve
detalaylari nelerdir(detaylarina inmeden hangi methodlar oldugunu
yazarsan dogru kategorinin mi detaylarina bakiyorsun baska seylerin mi
erkenden haberdar oluruz)

kw: context switch, userspace networking

[Fast and Compatible User-Space Container Networking with Programmable
NIC (Bojie Li,
2017)](https://www.sigops.org/src/srcsosp2017/sosp17src-final39.pdf)

“Each component is wrapped into a container and they communicate via a
virtual network, e.g., Linux bridge”

“Due to extensive communication among containers, currently the OS
kernel is the performance bottleneck for most containerized
applications.”

[XDP: The eXpress Data Path: Fast Programmable Packet Processing in the
Operating System Kernel (Red
Hat+Cilium)](https://www.diva-portal.org/smash/get/diva2:1256181/FULLTEXT01.pdf)

“Programmable packet processing is increasingly implemented using kernel
bypass techniques, where a userspace application takes complete control
of the networking hardware to avoid expensive context switches between
kernel and userspace. However, as the operating system is bypassed, so
are its application isolation and security mechanisms; and well-tested
configuration, deployment and management tools cease to function.”

[Why you should use io_uring for network
I/O](https://developers.redhat.com/articles/2023/04/12/why-you-should-use-iouring-network-io)

[On Sockets and System Calls Minimizing Context Switches for the Socket
API](https://www.usenix.org/system/files/conference/trios14/trios14-paper-hruby.pdf)
“The socket API is well understood and simple to use. However, its
simplicity has also limited its efficiency in existing implementations.
Specifically, the socket API requires the application to execute many
system calls like select, accept, read, and write. Each of these calls
crosses the protection boundary between user space and the operating
system, which is expensive. Moreover, the system calls themselves were
not designed for high concurrency and have become bottlenecks in modern
systems where processing simultaneous tasks is key to performance. We
show that we can retain the original socket API without the current
limitations.”

[Kernel- vs. User-Level Networking: A Ballad of Interrupts and How to
Mitigate Them (Peter Cai,
2023)](https://uwspace.uwaterloo.ca/bitstream/handle/10012/19598/Cai_Peter.pdf?sequence=1&isAllowed=y)
notes :\[\[Kernel- vs. User-Level Networking (Peter Cai, 2023)\]\]

https://cilium.io/blog/2018/11/20/fb-bpf-firewall/

fb’s katran also use xdp

[VIDEO: P99 CONF: High-Performance Networking Using eBPF, XDP, and
io_uring](https://www.youtube.com/watch?v=dWfA5460HYU)

what are our options: - a - normal, blocking sockets - 1 thread blocking
on every i/o syscall - Async sockets (epoll + nonblocking sockers,
io_uring) - Epoll: register sockets and get notified when they are
ready - io_uring:sdsddds - Low Level: AF_PACKET - Kernel Bypass: -
DPDK - Data Plane Development Kit (Intel) - Netmap - “the fast packet
I/O framework”

[Kernel-bypass techniques for high-speed network packet
processing](https://www.youtube.com/watch?v=MpjlWt7fvrw)

Kernel-bypass techniques: - User-space packet processing - Data Plane
Development Kit (DPDK) - Netmap - User-space network stack - mTCP

Typical Packet Flow

    | TX             |↓       ↑| RX             |
    | APPLICATION    |↓       ↑| APPLICATION    |
    | TRANSPORT (L4) |↓       ↑| TRANSPORT (L4) |
    | NETWORK (L3)   |↓       ↑| NETWORK (L3)   |
    | DATA LINK (L2) |↓       ↑| DATA LINK (L3) |
    | NIC DRIVER     |↓       ↑| NIC DRIVER     |
    | NIC HARDWARE   |↓       ↑| NIC HARDWARE   |
                      . → → → .  

Typical Packet

                                          ┌────────┬─────────┬───┬────────┬───┐  
                                          │src port│dest port│...│checksum│...│  
                                          └────────┴─┬───────┴───┴────────┴───┘  
                                                     │                           
                     ┌───────────────┬─────────┬─────┴────┬───────┬─────────────┐
                     │Ethernet header│IP Header│TCP Header│Payload│FrameChecksum│
                     └──┬────────────┴────┬────┴──────────┴───────┴─────────────┘
                        │                 │                                      
                        │                 │                                      
    ┌────────┬───────┬──┴─┐               │                                      
    │dest MAC│src MAC│type│               │                                      
    └────────┴───────┴────┘   ┌───┬──────┬┴──┬───────┬───────────┬──────┬───────┐
                              │...│length│...│IP type│header csum│src IP│dest IP│
                              └───┴──────┴───┴───────┴───────────┴──────┴───────┘

TX/RX rings - circular queue - shared between NIC and NIC driver -
content: Length + Packer Buffer Pointer

RX path:

    ┌───── NIC recieves the packet ────┐               
    │ - Match destination MAC address  │
    │ - Verify Ethernet checksum (FCS) │                   
    └──────────────────────────────────┘
    ↓
    ┌───── Packets accepted at the NIC ──────────────────────────┐               
    │ - Direct Memory Access (DMA) the packet to RX ring buffer  │
    │ - NIC Triggers an interrupt                                │                   
    └────────────────────────────────────────────────────────────┘
    ↓ Interrupt processing in the linux kernel (Top-half(minimal), Bottom-half(rest))
    ↓ - CPU interrupts the process in execution
    ↓ ! switch from user space to kernel space
    ┌───── Top-half interrupt proccessing ─────────────────┐               
    │ - Lookup IDT (Interrupt Descriptor Table)            │
    │ - Call corresponding ISR (Interrupt Service Routine) │
    │   - Acknowledge the interrupt                        │
    │   - Schedule bottom-half processing                  │
    │ ! switch back to user space                          │
    └──────────────────────────────────────────────────────┘
    ↓ CPU initiates the bottom-half when it is free (soft-irq(interrupt))
    ↓ ! switch from user space to kernel space
    ↓ Driver dynamically allocates an sk-buff (skb)
    ┌───── NIC driver proccessing ───────────────┐               
    │ 1. Driver dynamically allocates an sk-buff │ ← ← ↑ repeat for all
    │ 2. Update sk-buff with packet metadata     │     ↑ packets in buffer
    │ 3. Remove the Ethernet headder             │     ↑
    │ 4. Pass sk-buff to the network stack       │ → → →
    └────────────────────────────────────────────┘
    ↓ Call L3 protocol handler
    ┌───── L3 processing ──────────┐               
    │ Match destination IP/socket  │
    │ Verify checksum              │
    │ Remove header                │
    │ - Route lookup               │
    │ - Combine fragmented packets │
    │ - Call L4 protocol handler   │
    └──────────────────────────────┘
    ┌───── L4 processing ────────────┐               
    │ Match destination IP/socket    │
    │ Verify checksum                │
    │ Remove header                  │
    │ - Handle TCP state machine     │
    │ - Enqueue to socket read queue │
    │ - Signal the socket            │
    └────────────────────────────────┘
    ↓ Application processing
    ┌───── On socket read ──────────────────────────────────────┐               
    │ ! user space to kernel space                              │
    │ - Dequeue packet from socket recieve queue (kernel space) │
    │ - Copy packet to application buffer (user space)          │
    │ - Release sk-buff                                         │
    │ - Return back to the application                          │
    │ ! kernel space to user space                              │
    └───────────────────────────────────────────────────────────┘

    ┌───── On socket write ──────────────────────────┐               
    │ ! user space to kernel space                   │
    │ - Writes the packet to the kernel buffer       │
    │ - Calls socket's send function (e.g., sendmsg) │
    └────────────────────────────────────────────────┘
    ┌───── L4-specific processing ─┐               
    │ - Route lookup               │
    │ - Combine fragmented packets │
    │ - Call L4 protocol handler   │
    └──────────────────────────────┘
    ┌───── Common processing ──────┐
    │ Build header                 │
    │ Add header to packet buffer  │
    │ Update sk-buff               │
    └──────────────────────────────┘
    ┌───── L3-specific processing ─┐               
    │ - Fragment, if needed        │
    │ - Call L2 protocol handler   │
    └──────────────────────────────┘
    ┌───── L2 processing ───────────────────────────────┐               
    │ Enqueue packet to queue discipline (qdisc)        │
    │ - Hold packets in a queue                         │
    │ - Apply scheduling policies (e.g. FIFO, priority) │
    └───────────────────────────────────────────────────┘
    ┌───── qdisc ─────────────────────────────────┐               
    │ - Dequeue sk-buff (if NIC has free buffers) │
    │ - Post process sk-buff                      │
    │   - Calculate IP/TCP checksum               │
    │   - ... (tasks that h/w cannot do)          │
    │ - Call NIC driver's send function           │
    └─────────────────────────────────────────────┘
    ┌───── NIC driver ──────────────────┐               
    │ - If hardware transmit queue full │
    │   - stop qdisc queue              │ 
    │ - Otherwise:                      │
    │   - Map packet data for DMA       │
    │   - Tell NIC to send the packet   │
    └───────────────────────────────────┘
    ┌───── NIC ──────────────────────────────────────────────────────────┐
    │ - Calculates ethernet frame checksum (FCS)                         │
    │ - Sends packet to the wire                                         │ 
    │ ! Sends an interrupt "Packet is sent" : kernel space to user space │
    │ - Driver frees the sk-buff, starts the qdisc queue                 │
    └────────────────────────────────────────────────────────────────────┘

Packet processing overheads in the kernel:

- Too many context switches
  - pollutes CPU cache
- Per-packet interrupt overhead
- Dynamic allocation of sk-buff
- Packet copy between kernel and user space
- Shared data structures

-\> Cannot achieve line-rate for recent high speed NICs (40Gbps/100Gbps)

Optimizations to accelerate kernel packet processing:

- NAPI (New API) (2001)
- GRO (Generic Receive Offload) / GSO (Generic Segmentation Offload)
  - The default Ethernet maximum transfer unit (MTU) is 1500 bytes,
    which is the largest frame size that can usually be transmitted.
    This can cause system resources to be underutilized, for example, if
    there are 3200 bytes of data for transmission, it would mean the
    generation of three smaller packets. There are several options,
    called offloads, which allow the relevant protocol stack to transmit
    packets that are larger than the normal MTU. Packets as large as the
    maximum allowable 64KiB can be created, with options for both
    transmitting (Tx) and receiving (Rx). When sending or receiving
    large amounts of data this can mean handling one large packet as
    opposed to multiple smaller ones for every 64KiB of data sent or
    received. This means there are fewer interrupt requests generated,
    less processing overhead is spent on splitting or combining traffic,
    and more opportunities for transmission, leading to an overall
    increase in throughput.
- Use of multiple hardware queues

#### Overcome Overheads in Kernel: Bypass the Kernel

Do L2-L3-L4 packet processing in user-space

Interrupt Mode : Netmap

    ┌─────┐        ┌─────┐               
    │ CPU │ <===== │ NIC │
    └─────┘        └─────┘

- NIC notifies it needs servicing
- Interrupt is a hardware mechanism
- Handled using interrupt handler
- Interrupt overhead for high speed traffic
- Interrupt for a batch of packets

Poll Mode : DPDK

    ┌─────┐        ┌─────┐               
    │ CPU │ =====> │ NIC │
    └─────┘        └─────┘

- CPU keeps checking the NIC
- Polling is done with help of control bits (Command-ready bit)
- Handled by the CPU
- Consumes CPU cycles but handles high speed traffic
- Reserve processors/cores just for polling
- DPDK has many different polling sources/methods: Poll Mode Drivers
  (PMD)

DPDK(?), Netmap only manage processing till L2

What about L3-L7? Overheads with L3-L7 processing in kernel - Shared
data structure We need a userspace network stack over netmap or DPDK
that also handle shared data structure based overheads -\> mTCP:
multicore TCP

Modern NICs have Multiqueues: every incoming packet get distributed to
one of the RX queues, done using Receieve Side Scaling (RSS). Basically
hash of 4 headers (src_ip, dst_ip, src_port, dst_port) of any packet,
hash returns the queue id into which this packet will be allocated. Once
allocated it can be entirely processed in that core without needing any
intercore communication

mTCP - Designed for multicore scalable application - Per core TCP data
structures - e.g. accept queue, socket list - lock free - connection
locality - leverages multiqueue support of NIC - has multiple rings one
ring per core

##### What’s trending

- Offload application processing to the kernel
  - BPF (Berkeley Packet Filter)
  - eBPF (Extended BPF)
- Offload application processing to the NIC driver
  - XDP (eXpress DataPath)
- Offload application processing to programmable hardware
  - Programmable SmartNICs (NPU/DPU)
    - Netronome, Mellanox, Bluefield, Pensando
  - Programmable FPGAs
    - Xilinx, Altera
  - Programmable hardware ASICs
    - Barefoot Tofine, Cisco’s Doppler, Intel Flexpipe, Cavium’s Xpliant

(https://talawah.io/blog/extreme-http-performance-tuning-one-point-two-million/)
(https://github.com/aya-rs/aya/)
(https://konghq.com/blog/engineering/writing-an-ebpf-xdp-load-balancer-in-rust)

[Linux Networking - eBPF, XDP, DPDK, VPP - What does all that mean? (by
Andree Toonk)](https://www.youtube.com/@virtualnog)

(https://toonk.io/building-a-high-performance-linux-based-traffic-generator-with-dpdk/index.html)
Millions of packets/s Need a cheap ideally free, simple, high performant
traffic generator -\> DPDK pktgen: A DPDK based application that allows
you to define traffic patterns using CLI, LUA or PCAP (There’s also
Cisco TRex)

now that we have a reliable way to test performance, let’s start
experimenting

#### Regular Linux Kernel Networking

(https://toonk.io/linux-kernel-and-measuring-network-throughput/index.html)
![](/media/Pasted%20image%2020240402042020.png) In the results above,
you can see that one flow can go as high as 1.4Mpps. At that point, the
core serving that queue is maxed out (running 100%), and can not process
any more packets and will start dropping When doing the same test with
10,000 flows, I get to 14 Mpps, full 10g line rate at the smallest
possible packet size (64B) This is expected and is due to the hashing of
flows over different queues. Looking at the CPU usage, I estimate that
you’d need roughly 16 cores at 100% usage to serve this amount of
packets (interrupts).

In this test we’re adding two simple iptables rules to the DUT to see
what the impact is. The hypothesis here is that since we’re now going to
ask the system to invoke conntrack and do stateful session mapping,
we’re starting to execute more code, which could impact the performance
and system load.

    #added Iptables rules
    iptables -I FORWARD -d 10.10.11.1 -m conntrack — ctstate RELATED,ESTABLISHED -j ACCEPT
    iptables -I FORWARD -d 10.10.12.1 -m conntrack — ctstate RELATED,ESTABLISHED -j ACCEPT

![](/media/Pasted%20image%2020240402042741.png) The results for the
single flow performance test look exactly the same, that’s good. The
results for the 10,000 flows test, look the same as well when it comes
to packet per second. However, we do need a fair amount of extra CPU’s
to do the work.

In this test, we’re starting from scratch as we did in test 1 and I’m
adding a simple nat rule which causes all packets going through the DUT
to be rewritten to a new source IP. These are the two rules:
![](/media/Pasted%20image%2020240402042853.png) The results show that
rewriting the packets is quite a bit more expensive than just allowing
or dropping a packet. For example, if we look at the unidirectional test
with 10,000 flows, we see that we dropped from 14M pps (test 1) to 3.2
Mpps, we also needed 13 cores more to do this! One of the questions I
had starting this experiment was: **can Linux route at line-rate between
two network interfaces?** The answer is yes, we saw 14Mpps
(unidirectional), as long as there are sufficient flows, and you have
enough cores (~16). The bidirectional test made it to 12Mpps (24Mpps
total per NIC) with 26cores at 100%. We also saw that with the addition
of two stateful Iptables rules, I was still able to get the same
throughput, but needed extra CPU to do the work. So at least it scales
horizontally. Finally, we saw the rather dramatic drop in performance
when adding SNAT rules to test. With SNAT the maximum I was able to get
out of the system was 5.9Mpps; this was for 20k sessions (10k per
direction). So yes, you can build a close to line rate router in Linux,
as long as you have sufficient cores and don’t do too much packet
manipulations.

#### Userland Networking

(https://toonk.io/kernel-bypass-networking-with-fd-io-and-vpp/index.html)

#### DPDK

Networking stack is no longer handled by the kernel. DPDK is a userland
NIC driver, downside: you lose that NIC, ie no longer visible to the
kernel(if its the only nic that provides connection to the machine, can
virtiualize to not lose connection) + some cores are dedicated to
polling the NIC: constantly checking for packets Fast for sending and
receieving packets, great for packet generator. But you need to do
*everything* in userland.. ie, sockets, packet processing, firewalling,
forwarding…

So DPDK provides us with the ability to efficiently and extremely fast,
send and receive packets. But that’s also it! Since you’re not using the
kernel, we now need a program that takes the packets from DPDK and does
something with it. Like for example, a virtual switch or router.

#### VPP: Vector Packet Processing

VPP is part/heart of FD.io, an open-source software dataplane developed
by Cisco The VPP platform is an extensible framework that provides
switching and routing functionality. VPP is built on a ‘packet
processing graph.’ This modular approach means that anyone can ‘plugin’
new graph nodes. This makes extensibility rather simple, and it means
that plugins can be customized for specific purposes. FD.io can use DPDK
as the drivers for the NIC and can then process the packets at a high
performant rate that can run on commodity CPU. It’s important to
remember that it is not a fully-featured router, ie. it doesn’t really
have a control plane; instead, it’s a forwarding engine. Think of it as
a router line-card, with the NIC and the DPDK drivers as the ports. VPP
allows us to take a packet from one NIC to another, transform it if
needed, do table lookups, and send it out again. There are API’s that
allow you to manipulate the forwarding tables. Or you can use the CLI
to, for example, configure static routes, VLAN, vrf’s etc. To start, I
configured VPP with “vppctl” like this, note that I need to set static
ARP entries since the packet generator doesn’t respond to ARP.

    set int ip address TenGigabitEthernet19/0/1 10.10.10.2/24  
    set int ip address TenGigabitEthernet19/0/3 10.10.11.2/24  
    set int state TenGigabitEthernet19/0/1 up  
    set int state TenGigabitEthernet19/0/3 up  
    set ip neighbor TenGigabitEthernet19/0/1 10.10.10.3 e4:43:4b:2e:b1:d1  
    set ip neighbor TenGigabitEthernet19/0/3 10.10.11.3 e4:43:4b:2e:b1:d3

![](/media/Pasted%20image%2020240402045403.png) Those are some
remarkable numbers! With a single flow, VPP can process and forward
about 8Mpps, not bad. The perhaps more realistic test with 10,000 flows,
shows us that it can handle 14Mpps with just two cores. To get to a full
bi-directional scenario where both NICs are sending and receiving at
line rate (28 Mpps per NIC) we need three cores and three receive queues
on the NIC which needed approx. 26 cores with Linux Network Stack To
enable nat on VPP, I used the following commands:

    nat44 add interface address TenGigabitEthernet19/0/3
    nat addr-port-assignment-alg default
    set interface nat44 in TenGigabitEthernet19/0/1 out TenGigabitEthernet19/0/3 output-feature

![](media/Pasted%20image%2020240402052251.png)

with one flow only in one direction we’re able to get 4.3 which is
exactly half of what we got without nat. A single flow for nat isn’t
super representative of a real-life nat example where you’d be
translating many sources. So for the next measurements, I’m using 255
different source IP addresses and 255 destination IP addresses as well
as different port numbers; with this setup, the nat code is seeing about
16k sessions. I can now see the numbers go to 3.2Mpps; more flows mean
more nat work. Interestingly, this number is exactly the same as I saw
with iptables. There is however one big difference, with iptables the
system was using about 29 cores. In this test, I’m only using two cores.
note that I [isolated the
cores](https://www.linuxtopia.org/online_books/linux_kernel/kernel_configuration/re46.html) I
allocated to VPP so that the kernel wouldn’t schedule anything else on
it Pro’s - Super fast -\> cheap - VPP is plugin driven, could build your
own packet/dataplanne processor, witout the need for kernel changes. For
example, your custom tunnel protocol Con’s - Need dedicated NIC for DPDK
(or Single-root input/output virtualization (SR-IOV)) - You lose all
Kernel network functionality you’re familiar with (Sockets,
Iptables/netfilter, etc) - VPP is forwarding plane, doesn’t really have
a control plane (like BGP(Border Gateway Protocol)/OSPF(Open Shortest
Path First)), relies on “something” to program the FIB

![](media/Pasted%20image%2020240402052800.png)

#### XDP

Recent~ addition to Linux Kernel as an alternative to userland
networking, XDP utilize eBPF in kernel to connect to Linux kernel hook
points and verify and run its code. XDP is a special eBPF hook in the RX
path of the kernel and lets a user-supplied eBPF program to decide the
fate of the packet It sits very early on the recieve path of the kernel,
typically as soon as it comes out of the driver but *before* an
skb(sk-buff) is created \![\[Pasted image 20240402051504.png\]\] after
processing the packet, need to finalize it by returning a XDP action to
tell the bpf program what to do next - XDP_DROP This does exactly what
you think it does; it drops the packet and is often used for XDP based
firewalls and DDOS mitigation scenarios. - XDP_ABORTED Similar to DROP,
but indicates something went wrong when processing. This action is not
something a functional program should ever use as a return code. -
XDP_PASS This will release the packet and send it up to the kernel
network stack for regular processing. This could be the original packet
or a modified version of it. - XDP_TX This action results in bouncing
the received packet back out the same NIC it arrived on. This is usually
combined with modifying the packet contents, like for example, rewriting
the IP and Mac address, such as for a one-legged load balancer. -
XDP_REDIRECT The redirect action allows a BPF program to redirect the
packet somewhere else, either a different CPU or different NIC. We’ll
use this function later to build our router. It is also used to
implement AF_XDP, a new socket family that solves the highspeed packet
acquisition problem often faced by virtual network functions. AF_XDP is,
for example, used by IDS’ and now also supported by Open vSwitch.

![](media/Pasted%20image%2020240402051504.png)

#### eBPF

#### Open vSwitch

#### io_uring

(https://despairlabs.com/blog/posts/2021-06-16-io-uring-is-not-an-event-system/)

(https://ryanseipp.com/post/iouring-vs-epoll/)

Has two ring buffers one Submission Queue Buffer and one Completion
Queue Buffer that are shared between the kernel and the userspace.
Instead of requesting an operation, issueing a syscall, and waiting for
the result. Requests are put into a buffer and syscalls are made in
batches and async, and when the operation is resulted the result is put
into the Completion Queue Buffer, where the issuer can find it.

“Additionally, io_uring has added the ability for the kernel to poll the
SQ for additional work. This is a busy loop, and effectively wastes CPU
cycles and electricity, but means the application may be able to
eliminate syscalls entirely for its core logic.”

In June 2023, Google’s security team reported that 60% of Linux
kernel exploits submitted to their bug bounty program in 2022 were
exploits of io_uring vulnerabilities. As a result, io_uring was disabled
for apps in Android, and disabled entirely in ChromeOS as well as Google
servers. Docker also consequently disabled io_uring from their
default seccomp profile.

#### Snabb

#### PF_RING

#### LUNA

(https://www.usenix.org/system/files/atc23-zhu-lingjun.pdf) Why Not Just
Use Existing Solutions?

High packet processing overhead. The microsecond-scale Service Level
Objectives (SLOs) place significant pressures on packet processing speed
within the network stack. Several existing user-space TCP solutions
(e.g., mTCP and IX) delegate TCP protocol processing and application
logic to separate threads for better portability. Meanwhile, others
(e.g., VPP and TAS ) assign network and application processing to
different cores for better scaling. Such partitioning could slow the
processing speed due to context switch overhead or inter-core
communication. For example, in Figure 2(a), we further profile the mTCP
and VPP with the microbenchmark in §3.1. The results indicate that VPP
suffers high latency, mainly introduced by the CPU cycles waste, and
inter-core communication overheard.

Expensive memory copying. Data movement contributes a large proportion
of datacenter tax. In a typical cloud storage service test on the 50Gbps
network, memory copy can consume up to 12.5% CPU cycles, severely
impacting the end-to-end latency. When the bandwidth grows to 100Gbps or
more, the memory copy will take more than 40% CPU cycles, and further
incur the memory bandwidth bottleneck problem. However, for user-space
network stacks with a traditional IO path like mTCP and VPP, there are
two copy operations on the both receive and send paths (i.e., from user
to TCP receive/send buffer and between TCP buffer to the packet). Figure
2(b) shows that, mTCP could not fully utilize the 50Gbps network
bandwidth even with 16 cores. Our further analysis concludes that the
memory copying does consequence with excessive overhead. **One reason
that most user-space TCP solutions do not support zero-copy buffer is
that the traditional BSD-like socket interface would introduce
inevitable memory copy between the application and TCP buffer for
isolation.**

Supporting kernel traffic. Our storage services are deployed across
multiple clusters consisting of several generations of machines. For
many servers, their hardware are not capable of running user-space
networks (e.g., lack of hugepages support). Moreover, many applications
(such as monitoring agents) still rely on the kernel TCP. However, many
user-space TCP stacks demand to exclusively own the entire NIC, thus
could not collaborate with applications relies on kernel network stack
on the same machine.

Implementation quality. Many existing user-space TCP works are
research-oriented and thus can have various compatibility or performance
issues. For instance, VPP is not well-compatible with Mellanox NICs when
applying flow director filters. IX requires a particular Linux kernel
version to run as it relying on the Dune kernel module, and also only
provides drivers to the Intel NICs of outdated versions. Hacking into
these problems will take an unexpected amount of engineering effort with
rather limited community support. Hence, building a new user-space TCP
from scratch can be actually more time-saving.

LibOS mode. LUNA operates in a LibOS mode, similar to the mTCP \[20\]
and F-stack \[1\]. In this setup, the application and LUNA run in the
same process and share the memory address space.

we build LUNA with DPDK for its rich development kits and active
community support. LUNA leverages DPDK’s PMDs (Poll Mode Drivers) to
directly access packets from the NIC queues. We also utilize DPDK’s
hugepage management, and data structures like hash map and mbuf.

Share-nothing architecture. To exploit the parallel processing
capability of multi-core systems, like many previous designs \[7, 20\],
LUNA runs the threads in a share-nothing mode. Each core processes its
own traffic divided by the NIC’s multi-queue technique, and finishes
related application-layer processing on the same core. LUNA does not use
a dispatcher mode (like TAS \[23\]) or load balancing (like
task-stealing in Shenango \[32\]) due to cache efficiency and
synchronization overhead (e.g., from lock and atomic operations)
concerns

(https://www.youtube.com/watch?v=68Oq6XBNYFI)

![](media/Pasted%20image%2020240402070651.png)

![](media/Pasted%20image%2020240402072138.png)

![](media/Pasted%20image%2020240402070919.png)

#### gVisor?

(https://blog.cloudflare.com/why-we-use-the-linux-kernels-tcp-stack)
