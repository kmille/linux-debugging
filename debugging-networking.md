## Networking Stuff

### Wireshark on remote host
ssh server 'tcpdump -ni any -s0 -U -w - udp port 53' | wireshark -k -i -

### Wireshark on remote host over jump server

ssh -J jumpserver server 'tcpdump -ni any -s0 -U -w - udp port 53' | wireshark -k -i -

### Wireshark on remote host over jump server (if the private key for the server is only on the jump host)

on the jump server:  
>ssh server 'tcpdump -ni any -s0 -U -w- udp port 53' > /tmp/packets.pcap  

on our local machine:  
>ssh jump "tail -f -c +0 /tmp/packets.pcap" | wireshark -k -i -  

### Find your traffic in tcpdump
use source ports (NAT does not change the port) for filtering  
Explanation: for outgoing connections Linux uses a port out of `ip_local_port_range`  

>kmille@linbox /proc% cat sys/net/ipv4/ip_local_port_range  
>32768   60999  

This means: if we use a lower outgoing port we can easily find our packets  

>tcpdump -ni any portrange 2000-3000  
>nc -p 2000 -v localhost 8000  
>nc -s 192.168.10.70 -p 2000 -v localhost 8000  
>curl --local-port 2001 -v localhost:8000  
>curl --interface 192.168.10.70 --local-port 2001 -v localhost:8000  

### Find the dropping firewall
use `mtr` for a simple traceroute with a TCP SYN to 443 on top  
shows you where the packet is dropped when there are multiple firewalls and you don't know which is dropping your SYNs  

>mtr --tcp -P 443 server

### Useful tcpdump filter
SYN only (someone trys to connect but the firewall drops => retransmission)
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn'

Another way to find retransmissions: `sar -n ETCP 1` and check the `retrans/s` column  
To find which connection/application use  

>watch -n 0.1 'ss -tpn state syn-sent'  

SYN and SYN-ACK (show only newly established tcp connections)  
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn or tcp[13]=18'  

SYN and RST (connect to a port which is closed)  
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn or tcp[13] & 4!=0'

SYN and ICMP port unreachable (firewall rejects packet)  
>tcpdump -ni any 'tcp[tcpflags] == tcp-syn or icmp[0] = 3'

SYN AND SYN-ACK AND RST AND ICMP port unreachable
>tcpdump -ni any '(tcp[tcpflags] == tcp-syn or tcp[13]=18) or tcp[13] &amp; 4!=0 or icmp[0] = 3'  

For outgoing connections use [tcpconnect](https://github.com/iovisor/bcc/blob/master/tools/tcpconnect_example.txt)to get the process which is sending the packets. Or `ss -tanp` or `netstat -tanp`.

### Debugging Docker network issues
Use `docker run --rm --network=container:<name of a running contianer> -it nicolaka/netshoot` to get a shell with debugging tools in the network namespace of the container, that makes problem. You're not only in the same network as the target container, it also has access to the same ip. So you can just run tcpdump to get the incoming requests. I use these aliases in `~/.bashrc`.

>dg() { # docker grep
>    docker ps --format "{{ .Names }}" | rg $1
>}
>
>de() { #  docker exec
>    docker exec -it $1 bash
>}
>den() { #docker exec network (run debug container with network of $1 container
>    docker run --rm --network=container:$1 -it nicolaka/netshoot
>}


### Debugging iptables
#### Clear and check the counter (pkts/bytes)
>iptables -Z INPUT  
>iptables -Z INPUT 1  
>watch -n 0.1 iptables -vnL INPUT  

#### Log traffic to dmesg 
>iptables -I INPUT --match multiport --sports 2000:3000 -j LOG --log-prefix "our debug traffic"


#### Use the nflog interface
>iptables -A FORWARD -p icmp -j NFLOG --nflog-group 5  
>wireshark -ni nflog:5  


#### Trace the iptables rules linux uses for a package  
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

use `nft monitor trace` instead of dmesg  (on systems using netfilter and not nftables)
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

My mac addresses is redacted on purpose.
</pre>
</details>

for production use the following rules  
>iptables -t raw -I OUTPUT -p tcp -m multiport --sports 2000:3000 -j TRACE  
>iptables -t raw -I OUTPUT -p tcp -m multiport --dports 2000:3000 -j TRACE  
>iptables -t raw -I PREROUTING -p tcp -m multiport --dports 2000:3000 -j TRACE  
>iptables -t raw -I PREROUTING -p tcp -m multiport --sports 2000:3000 -j TRACE  


clear tracing  
>iptables -t raw -F
