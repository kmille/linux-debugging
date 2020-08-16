## Performance Analysis
This is based (copied) from Brendan Gregg!

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
