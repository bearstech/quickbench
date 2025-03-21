Quickbench
==

This is a bash script which was designed to run with no specific dependencies
(ie. on a barely booted minimal Linux-based server) and still provides a few
useful metrics to assess the basic performance of a given host.

Example usage :

```
$ ./quickbench

Quickbench: 1.0
Kernel    : Linux ovhb2 6.1.0-26-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.112-1 (2024-09-30) x86_64 GNU/Linux
Bash      : GNU bash, version 5.2.15(1)-release (x86_64-pc-linux-gnu)
dd        : dd (coreutils) 9.1

[ CPU info ]
Model name:                           Intel Core Processor (Haswell, no TSX)
BIOS Model name:                      pc-i440fx-8.2  CPU @ 2.0GHz
Thread(s) per core:                   1
Core(s) per socket:                   1
Socket(s):                            4

[ CPU loop ]
 1 // jobs :     478140 loop/job/s
 2 // jobs :     476692 loop/job/s
 4 // jobs :     458565 loop/job/s

[ Memory ]
Total    : 14641 MB
Bandwidth: 22.0 GB/s

[ Storage info ]
Name      Bus     Size(GB)  QDepth  Model
sda       scsi        100      128  QEMU HARDDISK   

[ Storage bandwidth - 4k blocks ]
Writes ( 1 thread[s]):     50 MB/s   12314 IO/s       0 µs avg. lat.
Writes ( 4 thread[s]):    151 MB/s   36998 IO/s       0 µs avg. lat.
Writes (16 thread[s]):    156 MB/s   38115 IO/s       0 µs avg. lat.
Reads  ( 1 thread[s]):     50 MB/s   12313 IO/s       0 µs avg. lat.
Reads  ( 4 thread[s]):    136 MB/s   33262 IO/s       0 µs avg. lat.
Reads  (16 thread[s]):    156 MB/s   38188 IO/s       0 µs avg. lat.

[ Storage bandwidth - 1M blocks ]
Writes ( 1 thread[s]):   1450 MB/s    1384 IO/s       0 µs avg. lat.
Writes ( 4 thread[s]):   1483 MB/s    1415 IO/s       2 µs avg. lat.
Writes (16 thread[s]):   1480 MB/s    1611 IO/s       7 µs avg. lat.
Reads  ( 1 thread[s]):   1450 MB/s    1383 IO/s       0 µs avg. lat.
Reads  ( 4 thread[s]):   5202 MB/s    4961 IO/s       0 µs avg. lat.
Reads  (16 thread[s]):   6187 MB/s    5900 IO/s       1 µs avg. lat.
```

Parameters
===

The script accepts only **1 parameter** which is the **loop delay** : every
loop in the benchmark will not run more than the given figure in seconds. Using
less than 10 might give you unreliable results, use at least 30 seconds for
statistically meaningful results.


Header/info
===

Some measures are not absolute nor calibrated (mostly: the CPU loop), thus if
you want to compare some host, you have to make sure those figures are the same
(**Quickbench version**, **kernel version**, **bash version** and **dd version**).


CPU info
===

This block tries to sum up as much as possible the CPU configuration. The
**Model name** is usually from the CPUID instruction, but also most of the time
edited by hypervisors. The **BIOS Model name*ù might be complementary and often
helps to identify the underlying architecture (which cloud, which generation,
...).

CPUs are hierarchically considered as :
* **Sockets** : actually means a physical package (only populated 'sockets' are reported)
* **Cores** : this is a whole independent CPU unit, often there are many of these within a socket
* **Threads** : there are core sub-units, not full cores, and allow some limited form of instruction parellelism within a core


CPU loop
===

This is a busy-loop. Obviously it only exercices a few CPU instructions, but it
still helps to compare them (if there's a large difference, don't
over-interpret +/-5% results).

You should observe that the loop rate is the same as long as you don't run more
jobs than you have cores. This is useful to debunk cloud coperators which
sell you 'cores' which are actually 'threads' : most of the time two threads
won't manage to run two loops at nominal speed.


Memory info
===

The **Total** reported is what Linux could identy as allocatable memory
(`/proc/meminfo`'s `MemTotal`). It will never be the real/VM memory, since is
necessarily deduced the kernel's executable own footprint and a few hardware
quircks (like DMA windows, video memory, etc). It should still be within
100-200 MB of what's installed in your host.

The **Bandwidth** is a simple test asking the CPU to move a block of memory
around. Right now it might only hit the L1 cache and reflect its performance,
it's thus rather a "L1/L2 cache memory bandwidth test".


Storage info
===

Interesting block devices are listed here, with some informations about them if
they are found. Quickbench will not benchmark all of them, see below.


Storage benchmark
===

**This is not a thorough benchmark**, nothing beats
[Fio](https://github.com/axboe/fio) but this one is quite involved to run and
to interpret its results. But it's still possible to have a pretty good
approximation of an IO benchmark with the venerable **dd** (along with some of
its more modern features/flags).

The benchmark runs **on the filesystem of your current working directory**. If
you want to benchmark another device, you need to have a filesystem mounted on
it, and change your current working directory to this mount point.

The first part generates small IOs (**4k blocks**), and you should expect this
part to be IOPS-bound. This should give you an idea of how much IOs can be
handled by the backing device.

The second part generates large IOs (**1M blocks**), and you should expect this
part to be bandwidth-limited - since a single IO moves so much bytes. You
should see here the maximum transfer rates of your backing device.

Every line of the benchmark runs the same dd transfer loop, but with that many
loops in parallel. Most modern storages (SSDs, NVMes, network-distributed-based
like Ceph, etc) exhibit their best performance when sollicited with many jobs
in parallel.

Note that the **latency** is the **IO submission latency** which is not really
relevant. The **IO completion latency** would be much more interesting but
right now Quickbench cannot measure it (patch welcome).

