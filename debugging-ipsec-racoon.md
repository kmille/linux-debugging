## ~~Fighting~~ Debugging IPsec on linux (racoon + setkey)
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
- [ ] Is the secret the same on both endpoints (prevent special characters on really old systems)?
- [ ] We had problems using sha2 (the generated keys for phase2 where truncated at the wrong length) - use sha1 or sha512

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

