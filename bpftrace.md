# How to use bpftrace 

We can trace arbitrary kernel and userspace functions.

## Which kernel functions trace?

`bpftrace -l | grep kprobe`

Functions, the kernel knows right now: 

`root@buster:/sys/kernel/debug/tracing# cat available_filter_functions `

We have the `ac`  kernel module loaded:

`root@buster:/sys/kernel/debug/tracing# lsmod | grep ac`
`ac   16384  0`

Functions of the kernel module `ac`:

`root@buster:/sys/kernel/debug/tracing# cat available_filter_functions | grep '\[ac\]'
acpi_ac_get_state [ac]
acpi_ac_resume [ac]
acpi_ac_notify [ac]
acpi_ac_remove [ac]
acpi_ac_battery_notify [ac]
get_ac_property [ac]
acpi_ac_add [ac]`



## Example

1) Let's use `strace` to find some interesting system calls.

```bash
root@buster:/home/vagrant# strace -e network iptables-legacy -nL
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 4
getsockopt(4, SOL_IP, IPT_SO_GET_INFO, "filter\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., [84]) = 0
getsockopt(4, SOL_IP, IPT_SO_GET_ENTRIES, "filter\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., [672]) = 0
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
+++ exited with 0 +++
```

2) Let's find out which kernel we use. Get the kernel source code and checkout the right version.

```bash
root@buster:~/mod/linux# uname -a
Linux buster 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1 (2020-01-26) x86_64 GNU/Linux

root@buster:~/mod/linux# git status
HEAD detached at v4.19-rc8
nothing to commit, working tree clean
```

3) grep for some function names, constants or variable names

```c
root@buster:~/mod/linux# rg IPT_SO_GET_INFO
net/ipv4/ip_sockglue.c
1563:   if (optname >= BPFILTER_IPT_SO_GET_INFO &&
1600:   if (optname >= BPFILTER_IPT_SO_GET_INFO &&

net/ipv4/netfilter/ip_tables.c
1652:   case IPT_SO_GET_INFO:
1698:   case IPT_SO_GET_INFO:

include/uapi/linux/bpfilter.h
14:     BPFILTER_IPT_SO_GET_INFO = 64,

include/uapi/linux/netfilter_ipv4/ip_tables.h
140:#define IPT_SO_GET_INFO                     (IPT_BASE_CTL)
156:/* The argument to IPT_SO_GET_INFO */

```

In `net/ipv4/netfilter/ip_tables.c looks promising` there are two interesting functions:  `compat_do_ipt_get_ctl` and `do_ipt_get_ctl`. Let's find out if they are called.

```bash
root@buster:~# bpftrace -e 'kprobe:do_ipt_get_ctl { printf("function was called!\n"); }'
Attaching 1 probe...
function was called!
function was called!
```

Let's get some more details. This is how the signature of the `compat_do_ipt_get_ctl` function looks like:

```c
static int compat_do_ipt_get_ctl(struct sock *sk, int cmd, void __user *user, int *len)
```

Let's write some code.

```c
#include <net/sock.h>

kprobe:do_ipt_get_ctl
{
    printf("called by %s (pid: %d). and: %d\n", comm, pid, ((sock *)arg0)->__sk_common.skc_family);
}
```

And execute it.

```c
root@buster:~# bpftrace mypbraceprogram.bpf
/bpftrace/include/stdarg.h:52:1: warning: null character ignored [-Wnull-character]       
/lib/modules/4.19.0-8-amd64/source/arch/x86/include/asm/bitops.h:209:2: error: 'asm goto' constructs are not supported yet
/lib/modules/4.19.0-8-amd64/source/arch/x86/include/asm/bitops.h:256:2: error: 'asm goto' constructs are not supported yet
/lib/modules/4.19.0-8-amd64/source/arch/x86/include/asm/bitops.h:310:2: error: 'asm goto' constructs are not supported yet
/lib/modules/4.19.0-8-amd64/source/arch/x86/include/asm/jump_label.h:23:2: error: 'asm goto' constructs are not supported yet
/lib/modules/4.19.0-8-amd64/source/arch/x86/include/asm/signal.h:24:2: note: array 'sig' declared here
Attaching 1 probe...
called by iptables-legacy (pid: 2981). and: 2
called by iptables-legacy (pid: 2981). and: 2
```

What does the `2` mean?

```c
/usr/src/linux-headers-4.19.0-8-common/include/linux/socket.h
160 /* Supported address families. */
161 #define AF_UNSPEC   0
162 #define AF_UNIX     1   /* Unix domain sockets      */
163 #define AF_LOCAL    1   /* POSIX name for AF_UNIX   */
164 #define AF_INET     2   /* Internet IP Protocol     */
165 #define AF_AX25     3   /* Amateur Radio AX.25      */
166 #define AF_IPX      4   /* Novell IPX           */
```

So it's just IPv4. Some explanation of `((sock *)arg0)->__sk_common.skc_family)`. The first parameter of `compat_do_ipt_get_ctl` is a pointer to a `socket`.  A `socket` has another data structure of type `sock_common` called `__sk_common`. And  ``__sk_common`` has the attribute `skc_family` . You can (have to) find it out by grepping through the code.

```c
root@buster:/usr/src/linux-headers-4.19.0-8-common/include# rg '^struct sock \{'
net/sock.h
327:struct sock {
    ....
239 /**
 240   * struct sock - network layer representation of sockets
 241   * @__sk_common: shared layout with inet_timewait_sock
 242   * @sk_shutdown: mask of %SEND_SHUTDOWN and/or %RCV_SHUTDOWN
 243   * @sk_userlocks: %SO_SNDBUF and %SO_RCVBUF settings
 244   * @sk_lock:   synchronizer
 245   * @sk_kern_sock: True if sock is using kernel lock classes
 ...
 322   * @sk_rcu: used during RCU grace period
 323   * @sk_clockid: clockid used by time-based scheduling (SO_TXTIME)
 324   * @sk_txtime_deadline_mode: set deadline mode for SO_TXTIME
 325   * @sk_txtime_unused: unused txtime flags
 326   */
 327 struct sock {
 328     /*
 329      * Now struct inet_timewait_sock also uses sock_common, so please just
 330      * don't add nothing before this first member (__sk_common) --acme
 331      */
 332     struct sock_common  __sk_common;
 333 #define sk_node         __sk_common.skc_node
 
     ...
 
 123  *  struct sock_common - minimal network layer representation of sockets
 124  *  @skc_daddr: Foreign IPv4 addr
 125  *  @skc_rcv_saddr: Bound local IPv4 addr
 126  *  @skc_hash: hash value used with various protocol lookup tables
 127  *  @skc_u16hashes: two u16 hash values used by UDP lookup tables
 128  *  @skc_dport: placeholder for inet_dport/tw_dport
 129  *  @skc_num: placeholder for inet_num/tw_num
 130  *  @skc_family: network address family                          <----- here it is
 131  *  @skc_state: Connection state
...
 143  *  @skc_flags: place holder for sk_flags
 144  *      %SO_LINGER (l_onoff), %SO_BROADCAST, %SO_KEEPALIVE,
 145  *      %SO_OOBINLINE settings, %SO_TIMESTAMPING settings
 146  *  @skc_incoming_cpu: record/match cpu processing incoming packets
 147  *  @skc_refcnt: reference count
 148  *
 149  *  This is the minimal network layer representation of sockets, the header
 150  *  for struct sock and struct inet_timewait_sock.
 151  */
 152 struct sock_common {

     
     
```

Another example: `bpftrace -e 'kprobe:vfs_open { printf("open path: %s\n", str(((path *)arg0)->dentry->d_name.name)); }'`



# Useful links

- https://www.joyfulbikeshedding.com/blog/2019-01-31-full-system-dynamic-tracing-on-linux-using-ebpf-and-bpftrace.html

- https://lwn.net/Articles/793749/

- http://www.brendangregg.com/BPF/bpftrace-cheat-sheet.html
- https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md
- https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#6-tracepoint-static-tracing-kernel-level-arguments
- https://jvns.ca/blog/2017/07/05/linux-tracing-systems/
- https://piware.de/post/2020-02-28-bpftrace/