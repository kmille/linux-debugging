# Linux Debugging Cheat Sheet
Troubleshooting, Performance Analysis



## Networking Stuff

### Wireshark on remote host
ssh server 'tcpdump -ni any -s0 -U -w - udp port 53' | wireshark -k -i -

### Wireshark on remote host over jump server
on the jump server:  
>ssh server 'tcpdump -ni any -s0 -U -w- udp port 53' > /tmp/packets.pcap  

on our machine:  
>ssh jump "tail -f -b +0 /tmp/packets.pcap" | wireshark -k -i -  

### Finding traffic in tcpdump
use source ports (NAT does not change the port) for filtering  
for outgoing connections Linux uses a port out of `ip_local_port_range`  
>kmille@linbox /proc% cat sys/net/ipv4/ip_local_port_range  
>32768   60999  

This means: if we use a lower outgoing port we can easily find our packets  

>tcpdump -ni any portrange 2000-3000  
>nc -s 192.168.10.70 -4 -p 2000 -v localhost 8000  
>curl --interface 192.168.10.70 --local-port 2001 -4 -v localhost:8000  

### Useful tcpdump filter
SYN only (someone trys to connect but the firewall drops => retransmission)
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn'

Another way to find retransmissions: `sar -n ETCP 1` and check the `retrans/s` column  
To find which connection use  
>watch -n 0.1 'ss -tn state syn-sent'  

SYN and SYN-ACK (show new tcp connections)  
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn or tcp[13]=18'  

SYN and RST (connect to a port which is closed)  
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn or tcp[13] & 4!=0'

SYN and ICMP port unreachable (firewall rejects packet)  
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn or icmp[0] = 3'

SYN AND SYN-ACK AND RST AND ICMP port unreachable
>tcpdump -ni any '(tcp[tcpflags] == tcp-syn or tcp[13]=18) or tcp[13] &amp; 4!=0 or icmp[0] = 3'  

For outgoing connections use `tcpconnect` to get the process which is sending the packets. Or `ss tanp` or `netstat -tanp`.


### Debugging iptables
#### Clear and check the counter (pkts bytes)
>iptables -Z INPUT  
>iptables -Z INPUT 1  
>watch -n 0.1 iptables -vnL INPUT  

#### Log traffic to dmesg 
>iptables -I INPUT --match multiport --sports 2000:3000 -j LOG --log-prefix "our debug traffic"


#### Use the nflog interface
>iptables -A FORWARD -p icmp -j NFLOG --nflog-group 5  
>wireshark -ni nflog:5  


#### Trace the rules iptables applied for a package  
on older systems  
>modprobe ipt_LOG  
>echo ipt_LOG >/proc/sys/net/netfilter/nf_log/2  

on newer systems  
>modprobe nf_log_ipv4  
>sysctl net.netfilter.nf_log.2=nf_log_ipv4  

use the `raw` table  
use the `OUTPUT` chain for outgoing traffic  
use the `PREROUTING` chain for incoming traffic  

for testing
>iptables -t raw -I OUTPUT -p icmp -j TRACE  
>iptables -t raw -I PREROUTING -p icmp -j TRACE  

check `dmesg` for some output. If there is no output:  
1) check the `used by` column of lsmod (should be equal to the number of rules int the `raw` table)
>kmille@linbox ~% lsmod | grep nf_log_ipv4  
>nf_log_ipv4            16384  2   

2) check if you use nftables (like Debian buster)    
>root@buster:~# lsmod | grep nf_tables  
>nf_tables             143360  0  
>nfnetlink              16384  1 nf_tables  


use `nft monitor trace` instead of dmesg  
other fix: you can use `/usr/sbin/iptables-legacy` instead of the new default `/usr/sbin/iptables-nft`

<details>
<summary>sample output of -j TRACE</summary>
<pre>
kmille@linbox ~% dmesg -wHT
...
[Thu Apr 23 21:55:11 2020] TRACE: raw:OUTPUT:policy:2 IN= OUT=wlp3s0 SRC=192.168.10.70 DST=1.1.1.1 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=31519 DF PROTO=ICMP TYPE=8 	CODE=0 ID=11 SEQ=1 UID=1000 GID=100 
[Thu Apr 23 21:55:11 2020] TRACE: nat:OUTPUT:policy:1 IN= OUT=wlp3s0 SRC=192.168.10.70 DST=1.1.1.1 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=31519 DF PROTO=ICMP TYPE=8 CODE=0 ID=11 SEQ=1 UID=1000 GID=100 
[Thu Apr 23 21:55:11 2020] TRACE: filter:OUTPUT:policy:2 IN= OUT=wlp3s0 SRC=192.168.10.70 DST=1.1.1.1 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=31519 DF PROTO=ICMP TYPE=8 CODE=0 ID=11 SEQ=1 UID=1000 GID=100 
[Thu Apr 23 21:55:11 2020] TRACE: nat:POSTROUTING:rule:1 IN= OUT=wlp3s0 SRC=192.168.10.70 DST=1.1.1.1 LEN=84 TOS=0x00 PREC=0x00 TTL=64 ID=31519 DF PROTO=ICMP TYPE=8 CODE=0 ID=11 SEQ=1 UID=1000 GID=100 
[Thu Apr 23 21:55:11 2020] TRACE: raw:PREROUTING:policy:2 IN=wlp3s0 OUT= MAC=00:00:00:00:00:00:00:00:00:00:00:00:00:00 SRC=1.1.1.1 DST=192.168.10.70 LEN=84 TOS=0x00 PREC=0x00 TTL=59 ID=44374 PROTO=ICMP TYPE=0 CODE=0 ID=11 SEQ=1 
[Thu Apr 23 21:55:11 2020] TRACE: filter:INPUT:rule:3 IN=wlp3s0 OUT= MAC=00:00:00:00:00:00:00:00:00:00:00:00:00:00 SRC=1.1.1.1 DST=192.168.10.70 LEN=84 TOS=0x00 PREC=0x00 TTL=59 ID=44374 PROTO=ICMP TYPE=0 CODE=0 ID=11 SEQ=1 

For prettier output

kmille@linbox ~% dmesg  | grep TRACE: | egrep -v 'security|raw' | cut -d ' ' -f 3-8,14-17,21-22 | column -t
nat:OUTPUT:policy:1     IN=        OUT=wlp3s0  SRC=192.168.10.70                              DST=193.99.144.80  LEN=84             PROTO=ICMP  TYPE=8  CODE=0  ID=13
filter:OUTPUT:policy:2  IN=        OUT=wlp3s0  SRC=192.168.10.70                              DST=193.99.144.80  LEN=84             PROTO=ICMP  TYPE=8  CODE=0  ID=13
nat:POSTROUTING:rule:1  IN=        OUT=wlp3s0  SRC=192.168.10.70                              DST=193.99.144.80  LEN=84             PROTO=ICMP  TYPE=8  CODE=0  ID=13
filter:INPUT:rule:3     IN=wlp3s0  OUT=        MAC=00:00:00:00:00:00:00:00:00:00:00:00:00:00  SRC=193.99.144.80  DST=192.168.10.70  PROTO=ICMP  TYPE=0  CODE=0  ID=13

My mac adresses is redacted on purpose.
</pre>
</details>

for production use the following rules  
>iptables -t raw -I OUTPUT -p tcp -m multiport --sports 2000:3000 -j TRACE  
>iptables -t raw -I OUTPUT -p tcp -m multiport --dports 2000:3000 -j TRACE  
>iptables -t raw -I PREROUTING -p tcp -m multiport --dports 2000:3000 -j TRACE  
>iptables -t raw -I PREROUTING -p tcp -m multiport --sports 2000:3000 -j TRACE  


clear tracing  
>iptables -t raw -F

---


### ~~Fighting~~ Debugging IPsec
[Why IPsec is hard to debug:](https://libreswan.org/wiki/Linux_IPsec_Summit_2018_wishlist#Fixup_XFRM_and_tcpdump)
>The fact that you see some plaintext, but not all plaintext, is the most confusing aspect of IPsec to system administrators, who now believe hey are leaking plaintext. 

The better you know how a system works the better you can debug it. So before debugging IPsec read this:  

- https://wiki.strongswan.org/projects/strongswan/wiki/CorrectTrafficDump
- https://devcentral.f5.com/s/articles/understanding-ikev1-negotiation-on-wireshark-34187


#### Phase 1
- [ ] Firewall allows `udp port 500`? Outgoing traffic allowed?
- [ ] Is there communication between the both IPsec gateways? `tcpdump -ni any host <remote ipsec endpoint>`
	- [ ] Are we sending packets to the remote endpoint?
	- [ ] Is the remote endpoint talking to us?
	- [ ] Are they both talking to each other? We had the problem: We speak IKEv1. They speak IKEv2. Our software was too old to recognize IKEv2
- [ ] Are both using the same proposals? Use Wireshark for dissecting packets and compare parameters.
- [ ] Any errors in `tail -f /var/log/syslog`?
- [ ] Is the secret the same on both endpoints?
- [ ] We had problems using sha2 (the generated keys for phase2 where truncated at the wrong lenth) - use sha1 or sha512

You think it works?  

- check `racoocnctl show-sa esp` (look for state=mature)
- but: tunnel will only be established if there is some traffic
- `ping -I <src ip that goes through the the tunnel> <ip that goes through the tunnel>`


#### Phase 2

Check if the tunnel works:  

- You can ping a remote host? Great! Also check TCP in case of ugly NAT rules!
- You send packets through the tunnel but no esp packets are sent?
	- Check it with `tcpdump -ni any host <remote ipsec endpoint>`
- [ ] Firewall allows `esp` packets in both directions?
- [ ] Both endpoints have phase2 configured properly (ip ranges, crypto, ...)?
	- phase2 is encrypted. Wireshark is not that helpful here
- [ ] You read the logs carefully?
- [ ] You increased the debug level (`log notify, debug,and debug2`)?
- [ ] You verified with tcpdump that the traffic that should go into the tunnel looks like you expect?
- [ ] The firewall works as expected?
	- [ ] Use `-j TRACE`, `-j NFLOG` or `-j LOG` to see what iptables is trying to match
	- [ ] There is a rule in the FORWARD chain that allows the traffic? Rule is used?
	- [ ] There is _no_ rule in the `-t nat -L POSTROUTING` chain that NATs your traffic because of a rule like `-t nat -A POSTROUTING -o wan_interface -j MASQUERDE`? This would prevent it from going into the tunnel because source ip does not match anymore
- [ ] Is there someone who can help you?
- [ ] You took a break after the first hour of debugging? It probably won't get better!
- [ ] You yelled at IPsec and asked `Why not Wireguard????`?
- [ ] Traffic goes into the tunnel but there is no response (only outgoing `esp` packages but no incoming ones)
	- problem lays on the other side (ping not allowed? Firewall drops packages?)


#### Debug commands
`ip xfrm policy` shows the established phase2 connections  
`ip xfrm state` shows the keys and bytes used for a phase2 connection  
`ip xfrm policy` shows changes   

For debugging with tcpdump/iptables this packet flow overview can help:
![flow is great](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)


#### TODOS
- It should be possible to decrypt the phase2 handshakes like described [here](https://wiki.strongswan.org/projects/strongswan/wiki/CorrectTrafficDump) by reading the debug2 output of racoon


#### Decrypt IPsec traffic
1. Use [this script](https://gist.github.com/rectalogic/ee2a48e47584fc0825dad9ffe571ec92) to generate a esp_sa file
	- the script puts the output of `ip xfrm state` in a format wireshark likes
2. Put the file to the right place `scp server:esp_sa ~/.config/wireshark/esp_sa`
3. Wireshark will automatically use the file. Use the Wireshark filter `!isakmp`
	- proof: Wireshark -> Edit -> Preferences -> Protocols -> ESP -> ESP SAs

### Resources
- SYN/Accept-Queue (Recv-Q/Send-Q values in ss) 
	- https://blog.cloudflare.com/syn-packet-handling-in-the-wild/
	- https://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html
- Bind before connect (why we can't reuse some ports for outgoing connections sometimes)
	- https://idea.popcount.org/2014-04-03-bind-before-connect/
	- `watch -n 0.1 ss -4 -tanop` and take look at the timers

- Tracing iptables rules 
	- https://www.opsist.com/blog/2015/08/11/how-do-i-see-what-iptables-is-doing.html
	- https://backreference.org/2010/06/11/iptables-debugging/
	- https://blog.sleeplessbeastie.eu/2018/08/01/how-to-log-dropped-connections-from-iptables-firewall-using-netfilter-userspace-logging-daemon/



## Performance Analysis

uptime
- load sum averages over 1 minute, 5 minute, and 15 minute
- How is the load changing?

dmesg | tail
- check for
	- oom-killer (Out of memory: Kill process 18694 (perl) score 246 or sacrifice child)
	- dropped network packets (TCP: Possible SYN flooding on port 7001. Dropping request.  Check SNMP counters.)
	- broken disks (ata3.00: failed command: READ FPDMA QUEUED)

vmstat -w 1
- Package (procps, also includes kill/ps)
- CPU: 
	- is there idle time (100 - sy - us)?
	- high system time? System time is necessary for I/O processing. A high system time average, over 20%, can be interesting to explore further: perhaps the kernel is processing the I/O inefficiently.
	- r: # of processes running on CPU and waiting for a turn (without I/O)
		- to interpret: an "r" value greater than the CPU count is saturation
- RAM
	- free memory in kb. If there are too many digits to count, you have enough free memory
	- si,so: swap-ins and swap-outs. If these are non-zero, you’re out of memory
- Disk:
	-  wa: constant degree of wait I/O points to a disk bottleneck; this is where the CPUs are idle, because tasks are blocked waiting for pending disk I/O

mpstat -P ALL 1
- Package: sysstat
- CPU time breakdowns per CPU
- check imbalance
- single hot CPU? Single threaded application?

pidstat 1
- Package: sysstat
- who uses the CPU?
- sys? user? wait?
- %CPU column is the total across all CPUs

iostat -xz 1
- Package: sysstat
- r/s, w/s, rkB/s, wkB/s: These are the delivered reads, writes, read Kbytes, and write Kbytes per second to the device. 
- await: The average time for the I/O in milliseconds. This is the time that the application suffers, as it includes both time queued and time being serviced. Larger than expected average times can be an indicator of device saturation, or device problems.
- %util: Device utilization. This is really a busy percent, showing the time each second that the device was doing work. Values greater than 60% typically lead to poor performance (which should be seen in await), although it depends on the device. Values close to 100% usually indicate saturation.


free -h
- Package: procps
- buffers: For the buffer cache, used for block device I/O.
- cached: For the page cache, used by file systems
- free: how  much  memory  is  available  for starting new applications, without swapping

sar -n DEV 1
- Package: sysstat

sar -n TCP,ETCP 1
active/s: Number of locally-initiated TCP connections per second (e.g., via connect()).
passive/s: Number of remotely-initiated TCP connections per second (e.g., via accept()).
retrans/s: Number of TCP retransmits per second.

Retransmits are a sign of a network or server issue; it may be an unreliable network (e.g., the public Internet), or it may be due a server being overloaded and dropping packets. 



top



CPU flame graph

## Good Reeds
- Linux Performance Analysis in 60s (video)
	- http://www.brendangregg.com/blog/2015-12-03/linux-perf-60s-video.html
	- https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55
	- https://www.youtube.com/watch?v=FJW8nGV4jxY&list=PLhhdIMVi0o5RNrf8E2dUijvGpqKLB9TCR
	- https://www.slideshare.net/brendangregg/velocity-2015-linux-perf-tools
	- http://www.brendangregg.com/Slides/Velocity2015_LinuxPerfTools.pdf

- Networking
    - https://blog.cloudflare.com/syn-packet-handling-in-the-wild/
    - https://blog.nelhage.com/2010/08/write-yourself-an-strace-in-70-lines-of-code/
    - https://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html

- else:
    - https://veithen.io/2013/11/18/iowait-linux.html
    - https://blog.heckel.io/2015/10/18/how-to-create-debian-package-and-debian-repository/

todos
Dieser Graph ist ganz nice
Was ist in was fürm Paket

Use method
Workload	Characteriza<on	Method (was den Kunden fragen) - http://www.brendangregg.com/Slides/Velocity2015_LinuxPerfTools.pdf 15/14x

memstat: voher #CPUs rausfinden

df -h
check: remote disks?

M13: load?

Was sagt er zu remote storage?

todo: check log files
	- welche sind offen: lsof/opensnoop


nicstat -z 1 -x
	- geht was an lo?
	- kann auch bits anstatt bytes
	Was sind die ERROR Werte
Paket: nicstat






Google: idelnde TPC Verbindung?


was lief nicht so dolle:
mehr auf logfiles achten
vmstat -w => falsche Spalate gesehen
Nicht auf die Events vom Monitoring geachtet

Linux process states => S was heißt das genau
	bremen-web3: php70fpm[bremen] max children nearly reached mal genauer anschaen. Was bedeutet das? wie wirds gemonitort?


​	
​	
	https://www.dynatrace.com/news/blog/7-minute-workout-does-your-apache-web-server-need-love/
	https://serverfault.com/questions/516373/what-is-the-meaning-of-ah00485-scoreboard-is-full-not-at-maxrequestworkers

```bash
#SCRIPT_NAME=/status SCRIPT_FILENAME=/status REQUEST_METHOD=GET cgi-fcgi -bind -connect /var/run/php/php7.2-fpm.sock
export SCRIPT_NAME=/status 
export SCRIPT_FILENAME=/status 
export REQUEST_METHOD=GET 
cgi-fcgi -bind -connect /var/run/php/php7.2-fpm.sock
https://easyengine.io/tutorials/php/fpm-status-page/
```
