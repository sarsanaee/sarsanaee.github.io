---
layout: post
title: Google's software/hardware data plane at the end-hosts
---



This Figure 1 is just my imagination about google software/hardware date plane at
the end-hosts.

Snap [SOSP'19] is basically the vswitch that forwards the packet to/from the
applications and NIC. Every single component should eventually integrate into
Snap.  Snap does multiple jobs, including packet processing, congestion
control, flow control, connection management, thread scheduling (close to
Shenango [NSDI'19]).

![Figure 1](/images/posts/figure1.png)

The prior system of Snap, I believe to be andromeda [NSDI'18]. Andromeda has
severe problems particularly related to head-of-line (HOL) at the virtual NIC.
They solve HOL blocking problem by building a predictable virtual NIC (PicNIC
[Sigcomm'19]).

PicNIC benefits from a backpressure mechanism from NIC to the virtual machines
to avoid NIC flooding and HOL. The backpressure mechanism comes from a Timing
Wheel - Carousel (Sigcomm'17) at the hypervisor.

Carousel implements packet scheduling so efficiently that it can easily scale
up to thousands of flows (I don't know the CPU overhead of this piece of
software). I think Eiffel (NSDI'19) tries to address CPU overhead of the timing
wheel. Note that Eiffel is not a google paper. However, I believe they have
adopted that in the Snap as well. In summary, Carousel is a traffic shaper and
pacer. I'm guessing Carousel is a very nice idea to be offloaded to NIC. Maybe
a Loom-like Carousel would be a nice thing to have.

Now Pony Express is the beating heart of Snap regarding the networking
services. It includes the congestion control subsystem (Swift [Sigcomm'20]). I
think the main insight in the swift is that end-hosts can be congested and drop
packets. The system must differentiate between two cases. First, congestion at
the end-host and congestion in the network. Congestion in the end-hosts happens
due to improper process scheduling, thread managements, wrong queuing, and load
balancing. Also, the are hardware limitations as well including PCI-e crossing
considerations. There are two different congestion windows for swift, local
congestion window and a remote congestion window which we should always pick
the min of these two.

Swift is an RTT-based congestion control mechanism. It uses RTT as congestion
control signals and **not packet drop**. In my view, packet drop is the worst
option as a congestion control signal, since it recovers when disaster has
already happened, packet loss. Now RTT-based congestion controls require
fine-grained latency measurements to understand where precisely the congestion
originates from. They are using multiple timestamps at different layers to keep
track of latency situations. For instance, Swift probes at local machines
stack, machine NIC, remote machine stack, and remote machine NIC.

In this system, software with general-purpose CPUs handle almost everything.
There is a trade-off between pushing various functionalities into the hardware
or keep everything in the software. However, I think there should be a middle
ground. There are some tasks that hardware is good at, and there are some other
tasks quite suitable for the software. The more complex the functionality, the
more reasonable the software.

On that note, 1RMA [Sigcomm'20] comes to the game. 1RMA is a middle-ground
solution. I think the idea of 1RMA is that we should keep complex
functionalities in the software, like what? Like congestion control and flow
control. However, we might manage to offload encryption into the hardware. RDMA
is an excellent solution for low CPU utilization and low tail latency.
Unfortunately, RDMA also has some limitations which I'm not going to dive into.
1RMA achieves higher performance compared to its counterparts because it can
apply simple policies in the hardware. It can separate latency-sensitive and
throughput-sensitive messages in the hardware. It also benefits from a new
design that manages to scale the number of connections (it is not
connection-oriented anymore).











