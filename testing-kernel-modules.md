# Debug a Linux Kernel Module (easy)

## First test
Try the Hello World exmpale from [cyberciti.biz](https://www.cyberciti.biz/tips/build-linux-kernel-module-against-installed-kernel-source-tree.html)  

hello.c
```c
#include <linux/module.h>
#include <linux/kernel.h>

int init_module(void)
{
                printk(KERN_INFO "init_module() called\n");
                        return 0;
}

void cleanup_module(void)
{
                printk(KERN_INFO "cleanup_module() called\n");
}

MODULE_LICENSE("GPL");
```

Makefile

```Makefile
obj-m := hello.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
default:
        $(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules
clean:
        rm -rf *.ko
        rm -rf *.mod.c
        rm -rf *mod.o
        rm -rf *.o

```

>make  
>insmod hello.ko  
>rmmod hello.ko  
>dmesg:  
>[  512.003876] init_module() called  
>[  512.003876] init_module() called  


## Debug our module
for example `nf_log_ipv4` (used for -j TRACE in iptables)

1. copy/download the source code
    - download it from Github
    - git clone https://github.com/torvalds/linux
    - obviously: code must match the kernel version
2. use multiple print statements (also print variables)
    - `printk(KERN_INFO "nf_log_ipv4: Passed %s %d \n",__FUNCTION__,__LINE__);`
3. in the Makefile replace `hello.o` with `nf_log_ipv4.o`
4. `insmod nf_log_ipv4` 
    - does not load dependencies => `symbol not found` error
    - modprobe loads dependencies (needs a `modprobe` before)
    - modprobe loads from /lib/modules/$(uname -r)/ ... (insmod from the current dir)

### Automate the process 
I was testing why `iptables -j TRACE` didn't log something to dmesg. Solution: Debian buster uses nf_tables per default and you get the logs via `nft monitor trace`.  But using the old iptalbes (`/usr/sbin/iptables-legacy`) it logged to dmesg.  

```bash
root@buster:~/mod/hello-example# cat rebuild.sh 
#/bin/bash
#iptables=/usr/sbin/iptables-legacy
iptables=iptables

make clean
make
rmmod nf_log_ipv4
modprobe nf_log_common
insmod nf_log_ipv4.ko
echo nf_log_ipv4 > /proc/sys/net/netfilter/nf_log/2
cat /proc/net/netfilter/nf_log
$iptables -t raw -F
$iptables -t raw -I OUTPUT -p icmp -j TRACE
$iptables -t raw -I PREROUTING -p icmp -j TRACE
lsmod | grep nf_log
ping -c 1 1.1.1.1
$iptables -t raw -vnL
```


## Find the source code/which module to look at?
It's very easy to grep the kernel code. Use `strace` to get the user space part:
```bash
root@linbox ~ # strace -e network iptables -vnL
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 4
getsockopt(4, SOL_IP, IPT_SO_GET_INFO, "filter\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., [84]) = 0
getsockopt(4, SOL_IP, IPT_SO_GET_ENTRIES, "filter\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., [2984]) = 0
Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, 0x7fff31ce9a90, [30]) = -1 EPROTONOSUPPORT (Protocol not supported)
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "conntrack\0\202\365d\177\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\1", [30]) = 0
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, 0x7fff31ce9a40, [30]) = -1 EPROTONOSUPPORT (Protocol not supported)
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "conntrack\0\214\365d\177\0\0\1\0\0\0\0\0\0\0\3111\200\365d\2", [30]) = 0
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, 0x7fff31ce99f0, [30]) = -1 EPROTONOSUPPORT (Protocol not supported)
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "conntrack\0\227\342\332y\347X\1\0\0\0\0\0\0\0\3111\200\365d\3", [30]) = 0
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, 0x7fff31ce99a0, [30]) = -1 EPROTONOSUPPORT (Protocol not supported)
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "conntrack\0\214\365d\177\0\0\0\0\0\0\0\0\0\0\3111\200\365\1\3", [30]) = 0
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "conntrack\0\214\365d\177\0\0\0\0\0\0\0\0\0\0\0\0\0\0\1\3", [30]) = 0
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "conntrack\0\214\365d\177\0\0\0\0\0\0\0\0\0\0\0\0\0\0\1\2", [30]) = 0
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            state INVALID
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, 0x7fff31ce9d00, [30]) = -1 EPROTONOSUPPORT (Protocol not supported)
1472K 1056M ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
 1619 97140 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_TARGET, "REJECT\0\0s\371\202\365d\177\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", [30]) = 0
    0     0 REJECT     all  --  cdark.net *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "tcp\0d\177\0\0s\371\202\365d\177\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", [30]) = 0
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22000
socket(AF_INET, SOCK_RAW, IPPROTO_RAW)  = 5
getsockopt(5, SOL_IP, IPT_SO_GET_REVISION_MATCH, "udp\0d\177\0\0s\371\202\365d\177\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", [30]) = 0
    0     0 ACCEPT     udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:21027
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:5001
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080
   63 17777 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 398K packets, 4393M bytes)
 pkts bytes target     prot opt in     out     source               destination
+++ exited with 0 +++
root@linbox ~ #
```

to get the kernel space code:  

- Google: `torvald/linux IPT_SO_GET_REVISION_MATCH`
    - you will find https://github.com/torvalds/linux/blob/master/include/uapi/linux/netfilter_ipv4/ip_tables.h  
- use https://elixir.bootlin.com/linux/latest/ident/IPT_SO_GET_REVISION_MATCH as search engine



