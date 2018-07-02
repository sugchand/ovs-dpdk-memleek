# Memory Leak Detection in OVS-DPDK
Here is the summary of my investigation to find a way for identifying memory
leaks in OVS-DPDK.

## Valgrind
Valgrind is the default option to find memory leak in any
application. Its been widely used , but when working with DPDK code base,
I found Valgrind is very limiting. This is due to the memory management
behavior in DPDK. DPDK doesnt really use standard 'glibc' memory management
APIs for its heap allocation, instead all the memory is allocated from the
hugepages at the application init time. Later on any runtime allocations are
made from this mempool. This is being handled by 'rte_*' apis.

Due to this specific DPDK memory model, valgrind cannot be used even for
tracking standard 'glibc' allocations as errors are throwing out at init itself.

To work with DPDK memory, there is a modified Valgrind is
available as below.

```
  https://github.com/bluca/valgrind-dpdk.git
```

Here are the steps that I used to run dpdk-valgrind in my setup

* Compile the dpdk-valgrind with default options

```
  ./autogen.sh

  ./configure --prefix=/home/sugeshch/valgrind-dpdk/install/bin/

  make
```

* Start vswitchd(compiled with DPDK) with modified valgrind

```
  /home/sugeshch/valgrind-dpdk/install/bin/valgrind \
  --soname-synonyms=somalloc=NONE --vgdb=yes --vgdb-error=0 \
  --leak-check=full --show-reachable=yes --error-limit=no \
  --gen-suppressions=all --suppressions=/tmp/ovs_supress.supp \
  --log-file=/tmp/valgrind.log ovs-vswitchd \
  --pidfile unix:/usr/local/var/run/openvswitch/db.sock --log-file

```
The 'vswitchd' application is initialized with valgrind and the instructions
to connect gdb to the vswitchd is dumped to the log file '/tmp/valgrind.log'

```
  $ cat /tmp/valgrind.log
  ==31204== Memcheck, a memory error detector
  ==31204== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
  ==31204== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
  ==31204== Command: /usr/bin/vswitchd/ovs-vswitchd --pidfile unix:/usr/local/var/run/openvswitch/db.sock --log-file
  ==31204== Parent PID: 31203
  ==31204==
  ==31204== (action at startup) vgdb me ...
  ==31204==
  ==31204== TO DEBUG THIS PROCESS USING GDB: start GDB like this
  ==31204==   /path/to/gdb /usr/bin/ovs-vswitchd
  ==31204== and then give GDB the following command
  ==31204==   target remote | /home/sugeshch/valgrind-dpdk/install/lib/valgrind/../../bin/vgdb --pid=31204
  ==31204== --pid is optional if only one valgrind process is running
  ==31204==

```
Note :- '31204' in the above log is the vswitchd pid.
the suppressions file let application to ignore some of valgrind errors.
My suppression file looks something like below.

```
$ cat /tmp/ovs_supress.supp
{
   <ovs_suppress_1>
   Memcheck:Param
   epoll_pwait(sigmask)
   fun:epoll_pwait
   fun:eal_intr_handle_interrupts
   fun:eal_intr_thread_main
   fun:start_thread
   fun:clone
}
{
   <ovs_suppress_2>
   Memcheck:Param
   epoll_ctl(event)
   fun:epoll_ctl
   fun:eal_intr_thread_main
   fun:start_thread
   fun:clone
}
{
   <ovs_suppress_3>
   Memcheck:Cond
   fun:rte_ixgbe_dev_atomic_write_link_status
   fun:ixgbe_dev_link_update_share
   fun:ixgbe_dev_link_update
   fun:rte_eth_dev_start
   fun:dpdk_eth_dev_init
   fun:netdev_dpdk_reconfigure
   fun:netdev_reconfigure
   fun:port_reconfigure
   fun:reconfigure_datapath
   fun:do_add_port
   fun:dpif_netdev_port_add
   fun:dpif_port_add
}
```
More details on the suppressions and its options can be found in the
following link. The suppression file above is just for reference. It may not
be same for every usecase.

```
https://wiki.wxwidgets.org/Valgrind_Suppression_File_Howto
```

* After these updates, I managed to start the vswitchd application without
any error. However it taking forever to complete the init. Even after 12hrs
of waiting the vswitchd still doing the init. Looking for help from people
here, who managed to run it successfully. ;)

## mtrace
After the struggle with vlagrind, I tried to use mtrace, which is part of
glibc itself for tracking alloc/free. mtrace provide a wrapper for allocations
and free operations. The good thing about mtrace is it allows to limit the scope
of tracking. I found it very useful for tracking the leaks introduced by new
feature development.

To avail mtrace support, it need to install glibc-utils(libc6-dev) in the system
before using it.

Steps to use mtrace in OVS-DPDK

* OVS uses wrapper functions for all the memory allocations and free
operations.The wrapper functions are defined in '/lib/util.c'.
mtrace can only track the alloc/free callers. So its necessary to get rid of
these wrappers to get the useful tracking information with the right callers.
Apply the atttached patch for replacing the wrapper functions in OVS-DPDK.

* Mark the start and end of tracking. To track for the entire vswitchd
application, apply the following changes.

```
  $ git diff vswitchd/ovs-vswitchd.c
  diff --git a/vswitchd/ovs-vswitchd.c b/vswitchd/ovs-vswitchd.c
  index 7de6d89..271498f 100644
  --- a/vswitchd/ovs-vswitchd.c
  +++ b/vswitchd/ovs-vswitchd.c
  @@ -48,6 +48,7 @@
   #include "openvswitch/vconn.h"
   #include "openvswitch/vlog.h"
   #include "lib/vswitch-idl.h"
  +#include <mcheck.h>

   VLOG_DEFINE_THIS_MODULE(vswitchd);

  @@ -106,6 +107,7 @@ main(int argc, char *argv[])

       exiting = false;
       cleanup = false;
  +    mtrace();
       while (!exiting) {
           memory_run();
           if (memory_should_report()) {
  @@ -135,7 +137,7 @@ main(int argc, char *argv[])
       bridge_exit(cleanup);
       unixctl_server_destroy(unixctl);
       service_stop();
  -
  +    muntrace();
       return 0;
   }

```

* Compile the OVS-DPDK with gdb option(-g)
```
  ./configure CFLAGS="-g" --with-dpdk=$DPDK_DIR/x86_64-native-linuxapp-gcc

  make -j CFLAGS="-g"
```

* Export trace log file to dump memory allocation traces of OVS-DPDK.
Make sure the log file is present and application has write permission on it.

```
  sudo -E MALLOC_TRACE=/tmp/mtrace.log $OVS_DIR/vswitchd/ovs-vswitchd \
  --pidfile unix:/usr/local/var/run/openvswitch/db.sock --log-file
```

* Exit the vswitchd application and validate trace file to
see the memory allocation logs. The memory validation is done using a perl
script 'mtrace' that provided with glib-utils itself.

The trace file result will be something like below,
```
$ mtrace vswitchd/ovs-vswitchd /tmp/mtrace.log
- 0x0000000002c4d660 Free 18246 was never alloc'd /home/sugeshch/ovs/lib/hmap.c:51
- 0x0000000002c46560 Free 20606 was never alloc'd /home/sugeshch/ovs/lib/daemon-unix.c:502
- 0x0000000002c4d790 Free 44872 was never alloc'd /home/sugeshch/ovs/lib/netlink-notifier.c:170
- 0x0000000002c4ce90 Free 44873 was never alloc'd /home/sugeshch/ovs/lib/if-notifier.c:54
............
............
............

Memory not freed:
-----------------
           Address     Size     Caller
0x0000000002c46560     0x38  at /home/sugeshch/ovs/lib/async-append-aio.c:58
0x0000000002c465a0     0x18  at /home/sugeshch/ovs/lib/netdev.c:226
0x0000000002c465c0     0x18  at /home/sugeshch/ovs/lib/netdev.c:226
0x0000000002c4d660      0xf  at /home/sugeshch/ovs/lib/util.c:145
0x0000000002c4d680     0x20  at /home/sugeshch/ovs/lib/shash.c:109
0x0000000002c4d6b0     0x20  at /home/sugeshch/ovs/lib/unixctl.c:119
0x0000000002c4d6e0      0xf  at /home/sugeshch/ovs/lib/util.c:145
0x0000000002c4d700     0x20  at /home/sugeshch/ovs/lib/shash.c:109
............
............
............

```

Developer can review these allocations to see which one is really a leak and
which one is not. It would be great to limit scope of mtrace to some specific
area of code execution for easy debugging.

