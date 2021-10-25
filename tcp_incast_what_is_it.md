# TCP incast: What is it? How can it affect Erlang applications?

Thu, Jan 5, 2012

So, what’s the deal with the “TCP incast” pattern?  Never heard of it?  Join the club.

I’ve been wearing a developer hat for too long and not wearing my sysadmin and network manager hats.  And the publications about Ethernet microbursts and the TCP incast pattern have been hiding in conference proceedings & journals that I don’t follow.  (Note to self: change reading habits.)

If you’d rather read a paper about the problem, go read one or more of these:

```
 * [http://www.cs.cmu.edu/~vrv/papers/PDSI07-Incast.pdf](http://www.cs.cmu.edu/~vrv/papers/PDSI07-Incast.pdf)
 * [http://www.eecs.berkeley.edu/~ychen2/professional/TCPIncastWREN2009.pdf](http://www.eecs.berkeley.edu/~ychen2/professional/TCPIncastWREN2009.pdf)
```

TL;DR: you can’t pour two buckets of manure into one bucket.  (Credit: my grandfather.)  I’m sure that’s very helpful … now keep on reading.

### Assumptions

If you have a recipe using typical, modern, commodity computing hardware like this:

```
 * Computers of fast CPU cores with lots of memory.
 * Very efficient network interface cards in those computers, and a reasonably modern OS to take advantage of them.
 * Gigabit Ethernet (or 10 Gbit/sec) Ethernet connecting those computers together.  One switch or a tree/fabric/whatever of switches, it doesn't matter too much which, though multiple switches might be a bit easier.
 * An application that has lots of many-to-one or many-to-many communication patterns.  The [Riak](http://basho.com/products/riak-overview/) database by [Basho Technologies](http://basho.com/) happens to do a large amount of many-to-one operations, so I'll use that as my example below.  The same principle applies to Hadoop and many other data-intensive distributed applications.
```

### The Easy Bandwidth Problem

Say you’ve got five machines using “scp” to copy data to a single destination.  Each source machine is capable of outputting 900+ Mbit/sec .  If your five machines each send 900+ Mbit/sec of traffic to a single recipient, your 1Gbit/sec Ethernet switch will soon have no choice but to drop packets.

But the “5 machines sending lots bulk data to 1 machine” is easy to understand: all machines cannot simultanously send 900+ Mbit/sec because the receiver has only 1,000 Mbit/sec of (theoretical) bandwidth.  The switch will drop some packets, then each TCP connection will react and slow down.  Eventually, each sender will reach a steady-state of sending roughly 200 Mbit/sec.  So, 5 * 200Mbit/sec = 1,000 Mbit/sec.  Easy, right?

### The More Difficult Bandwidth Problem

What if you have your Riak cluster of 20 machines, and each machine is using (on average) 250 Mbit/sec of both transmit & receive bandwidth on their 1 Gbit/sec Ethernet interfaces?  You’re only using 25% of the theoretical peak rate on each machine’s interface.  You’ve got _plenty of room_ to grow, right?  No, not really.  For clusters of data-intensive applications like Riak or [Hadoop](http://hadoop.apache.org/), that 25% rate may be halfway to your practical bandwidth limit.  Why?

Because you’re likely to have “microbursts” of traffic sent to a single cluster member.  The switch can’t buffer all of the frames in the microburst, so it drops some.  Then the TCP mechanisms intervene to slow things down.  If your cluster has average utilization of 25-35% and uses Ethernet switches with small buffers, you may already be dropping 40-100 frames per second per machine.  The frame drop rate will rise very sharply as utilization increases.  At 45-55% utilization, you’ll start seeing 100-500 dropped frames per second per machine.  Will TCP be able to run full-speed with frame drop rates that high?  No.  Instead, you’ll be stuck with a cluster that can barely run above 50-60% average utilization.

It sounds a bit crazy, but it’s a real phenomenon.  And it’s finally something that has happened to me.  It took a long time to realize what was really happening … because I never considered a network that was 45-55% utilized was really near its peak usable bandwidth.

### An Illustration Using Riak

The Riak database manages its data replication by storing several copies of any single piece of data on multiple machines.  Typically, the number of copies is three.  When your app tries to fetch a key from Riak, Riak will create a new process to coordinate the operation.  That new process will send ‘get’ requests to three different nodes and await their replies.  Each server that receives a ‘get’ request will send a copy of the key, its value, and its metadata dictionary back to the coordinator.

So, what if both of the following were true?

```
 1. All three 'get' results are sent to the coordinator at exactly the same time.  (Here, "exactly" means something like "within the same millisecond or perhaps even the same microsecond.)
 2. All three 'get' results are big, for example, 100 kilobytes each.
```

Then you have three different computers trying to send their 100 kilobytes of data to a single machine at the exact same time.  The resulting microburst of data creates a problem for the Ethernet switch(es) that these four computers are using.  (For the sake of simplicity, assume that all four computers are using the same Ethernet switch.)

If all three ‘get’ results arrive at the same instant, then the switch must do one of two things:

```
 1. Buffer all of the 300KB of packets.
 2. Drop some packets.
```

If you’re using a typical, low-cost commodity Ethernet switch such as a Cisco Catalyst 3750, you don’t have large amounts of buffer space (compared to other switches on the market).  See a [table from a Tolly Enterprises, LLC report number 211127](http://www.advizex.com/assets/1/7/Tolly211127HP3800SeriesTCOVsCisco3750XandJuniperEX4200.pdf):

But if your Ethernet switch has a lot of buffer space (like the HP or Juniper switches in the Tolly figure), you do not make it impossible to overrun your buffer space: large buffers only make buffer overruns less likely.  Remember, these are fast machines in this cluster.  And your switch may only have buffer space for a few dozen or a few hundred frames, depending on the frame size.

So, now imagine that the Riak cluster is much busier.  Each cluster member is taking in many thousands of queries per second, thus starting several thousand coordinators processes per second.  Each coordinator is getting (typically) 3 replies.  Each reply is big enough to use many Ethernet frames.  And it’s quite likely that enough frames will arrive from all over the cluster to overrun the Ethernet switch’s buffers for the coordinator machine’s Ethernet port.  Boom, “TCP incast” bites you, hard.

### Riak in Production at Voxer: a case study

I’ve seen a [customer’s cluster (Voxer)](http://www.voxer.com) do exactly this.  When utilization on a GigE network (fed by Cisco Catalyst 3750 switches) uses more than 50% average utilization (sampled @ 1 second intervals), TCP throughput collapses regularly.  The 1-second utilization rates will see-saw between 900 Mbit/sec down to 200 Mbit/sec.  To make matters worse, the 200 Mbit/sec rate will happen much more frequently than the 900 Mbit/sec rate.

To show that we’re seeing packet drops, we use a couple of methods.  First, using “tcpdump” and “tcptrace”.

```
root# sh -c 'FILE=/tmp/slf.capt`date +%s` ; echo Putting tcpdump output in $FILE ; time tcpdump -i eth0 -c 200000 -w $FILE ; tcptrace -l $FILE | grep "rexmt data pkts" | head -5 ; echo Pseudo-count of TCP flows with retransmitted packets: ; tcptrace -l $FILE | grep "rexmt data pkts" | grep -v "rexmt data pkts: 0 rexmt data pkts: 0" | wc -l ; echo Total retransmitted packets and bytes: ; tcptrace -l $FILE | awk " /rexmt data pkts/ { p += \$4 + \$8 } /rexmt data bytes/ { b += \$4 + \$8 } END { print p, b }" '
Putting tcpdump output in /tmp/slf.capt1325142154
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 96 bytes
200000 packets captured
200011 packets received by filter
0 packets dropped by kernel
0.22user 0.43system 0:07.50elapsed 8%CPU (0avgtext+0avgdata 17648maxresident)k
136inputs+36960outputs (3major+738minor)pagefaults 0swaps
 rexmt data pkts: 0 rexmt data pkts: 0
 rexmt data pkts: 0 rexmt data pkts: 0
 rexmt data pkts: 0 rexmt data pkts: 0
 rexmt data pkts: 0 rexmt data pkts: 0
 rexmt data pkts: 0 rexmt data pkts: 0
Pseudo-count of TCP flows with retransmitted packets:
99
Total retransmitted packets and bytes:
837 5608410
root# for i in `seq 1 30`; do sh -c 'echo -n `date` " " ; ifconfig eth0 ; sleep 1 ; ifconfig eth0' | grep bytes | sed -e 's/.. bytes://g' | awk 'NR == 1 { rx = $1; tx = $4} NR == 2 { printf "rx %.1f Mbit/sec tx %.1f Mbit/sec ", ($1 - rx) * 8 / (1024*1024) / 1, ($4 - tx) * 8 / (1024*1024) / 1 }' ; date ; donerx 226.3 Mbit/sec tx 293.7 Mbit/sec Wed Dec 28 23:03:17 PST 2011rx 242.8 Mbit/sec tx 265.3 Mbit/sec Wed Dec 28 23:03:18 PST 2011
rx 260.4 Mbit/sec tx 302.1 Mbit/sec Wed Dec 28 23:03:25 PST 2011
rx 226.4 Mbit/sec tx 241.4 Mbit/sec Wed Dec 28 23:03:26 PST 2011
rx 238.1 Mbit/sec tx 277.9 Mbit/sec Wed Dec 28 23:03:27 PST 2011
rx 270.5 Mbit/sec tx 315.9 Mbit/sec Wed Dec 28 23:03:28 PST 2011
rx 257.4 Mbit/sec tx 287.5 Mbit/sec Wed Dec 28 23:03:29 PST 2011
^C
```

That’s 837 dropped packets in 7.5 seconds of sampling, at an average 1-second throughput of 240-315 Mbit/sec.

Looking at this in a more Linux-specific manner, and a bit more exact measurement of transmit bandwidth during the time period that we’re measuring transmissions:

```
root# netstat -s | egrep 'segments retransmited|OutOctets|requests sent out' ; sleep 1; netstat -s | egrep 'segments retransmited|OutOctets|requests sent out'
    1542387771 requests sent out
    521663296 segments retransmited
    OutOctets: 1222021825
    1542398375 requests sent out
    521663388 segments retransmited
    OutOctets: 1248437047
```

That’s 10604 packets sent, 92 packets retransmitted, for 26415222 octets sent (201 Mbit/sec).  The packet retransmission rate is 0.9.  (Yes, all other machines are using roughly the same 1-second bandwidth.)  At 201 Mbit/sec, we’re at 20% of allegedly full bandwidth.  If we double the average utilization of all machines to 400Mbit/sec, the packet rate moves up to the 4-6% range.  And if all machines go up to 500Mbit/sec, things get ugly.

```
rx 364.4 Mbit/sec tx 916 Mbit/sec Wed Dec 21 15:43:31 PST 2011
rx 339.1 Mbit/sec tx 951 Mbit/sec Wed Dec 21 15:43:32 PST 2011
rx 361.6 Mbit/sec tx 952 Mbit/sec Wed Dec 21 15:43:33 PST 2011
rx 475.4 Mbit/sec tx 491 Mbit/sec Wed Dec 21 15:43:34 PST 2011
rx 529.2 Mbit/sec tx 415 Mbit/sec Wed Dec 21 15:43:36 PST 2011
rx 472.9 Mbit/sec tx 267 Mbit/sec Wed Dec 21 15:43:37 PST 2011
rx 505.3 Mbit/sec tx 269 Mbit/sec Wed Dec 21 15:43:38 PST 2011
rx 393.5 Mbit/sec tx 191 Mbit/sec Wed Dec 21 15:43:39 PST 2011
rx 175.1 Mbit/sec tx 8 Mbit/sec Wed Dec 21 15:43:40 PST 2011
rx 436.8 Mbit/sec tx 246 Mbit/sec Wed Dec 21 15:43:41 PST 2011
rx 487.6 Mbit/sec tx 246 Mbit/sec Wed Dec 21 15:43:42 PST 2011
rx 524.2 Mbit/sec tx 194 Mbit/sec Wed Dec 21 15:43:43 PST 2011
rx 441.9 Mbit/sec tx 699 Mbit/sec Wed Dec 21 15:43:44 PST 2011
rx 382.8 Mbit/sec tx 952 Mbit/sec Wed Dec 21 15:43:45 PST 2011
rx 331.5 Mbit/sec tx 951 Mbit/sec Wed Dec 21 15:43:46 PST 2011
rx 391.3 Mbit/sec tx 949 Mbit/sec Wed Dec 21 15:43:47 PST 2011
rx 538.6 Mbit/sec tx 396 Mbit/sec Wed Dec 21 15:43:48 PST 2011
```

Bummer.  During about 20 seconds, our transmit bandwidth ranges from a high of 952 Mbit/sec all the way down to 8 Mbit/sec.  EIGHT!  For an entire second!  And it took four more seconds before the transmit rate rises above 500Mbit/sec.

So, one last question about the whole microburst and Ethernet switch buffering and “TCP incast” problem is … are the microbursts really happening?  Are the switch ports really being pushed to full line rate some of the time?

As far as I can tell, the answer is “yes”.  Using an Erlang program (Escript, actually), I have good timer resolution down to about 4 milliseconds, perhaps even 2 milliseconds.  To get polling accurate below that, I’d have to write a demo program in another language.  (Or write the Erlang program so that avoids using timers and instead uses busy-wait loops: Erlang’s time-of-day clock has microsecond resulution.)

Here’s a [link to the escript that I used](http://www.snookles.com/scotttmp/poll-value-mbit.escript), and here’s some output, taken from an off-peak period last night. The first argument is the number of milliseconds between polling periods, and the second is the path to Linux’s /sys file system file for incoming network octets. (The {22,x,y} stuff is a timestamp, e.g. 22:29:21 Pacific time.  The “ratio” figure divides the maximum bandwidth observed during that second by the average bandwidth for the second.)

```
root# ./poll-value-mbit.escript 500 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 184.7 Mbit/s Avg 170.3 Mbit/s Ratio 1.1 @ {22,29,21}
Max 206.6 Mbit/s Avg 185.7 Mbit/s Ratio 1.1 @ {22,29,22}
Max 230.1 Mbit/s Avg 206.0 Mbit/s Ratio 1.1 @ {22,29,23}
Max 192.1 Mbit/s Avg 185.4 Mbit/s Ratio 1.0 @ {22,29,24}
Max 183.7 Mbit/s Avg 167.4 Mbit/s Ratio 1.1 @ {22,29,25}
Max 212.5 Mbit/s Avg 191.8 Mbit/s Ratio 1.1 @ {22,29,26}
^C
root# ./poll-value-mbit.escript 250 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 174.0 Mbit/s Avg 157.8 Mbit/s Ratio 1.1 @ {22,29,32}
Max 225.1 Mbit/s Avg 180.2 Mbit/s Ratio 1.2 @ {22,29,33}
Max 210.7 Mbit/s Avg 182.6 Mbit/s Ratio 1.2 @ {22,29,34}
Max 188.3 Mbit/s Avg 172.3 Mbit/s Ratio 1.1 @ {22,29,35}
Max 193.5 Mbit/s Avg 177.5 Mbit/s Ratio 1.1 @ {22,29,36}
^C
root# ./poll-value-mbit.escript 125 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 256.2 Mbit/s Avg 200.5 Mbit/s Ratio 1.3 @ {22,29,44}
Max 245.3 Mbit/s Avg 209.4 Mbit/s Ratio 1.2 @ {22,29,45}
Max 271.8 Mbit/s Avg 212.4 Mbit/s Ratio 1.3 @ {22,29,46}
Max 214.6 Mbit/s Avg 189.4 Mbit/s Ratio 1.1 @ {22,29,47}
Max 261.8 Mbit/s Avg 199.5 Mbit/s Ratio 1.3 @ {22,29,48}
^C
root# ./poll-value-mbit.escript 64 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 458.6 Mbit/s Avg 311.9 Mbit/s Ratio 1.5 @ {22,30,0}
Max 389.1 Mbit/s Avg 236.0 Mbit/s Ratio 1.6 @ {22,30,1}
Max 267.2 Mbit/s Avg 162.8 Mbit/s Ratio 1.6 @ {22,30,2}
Max 276.8 Mbit/s Avg 167.6 Mbit/s Ratio 1.7 @ {22,30,3}
Max 229.3 Mbit/s Avg 172.3 Mbit/s Ratio 1.3 @ {22,30,4}
Max 346.6 Mbit/s Avg 193.1 Mbit/s Ratio 1.8 @ {22,30,5}
^C
root# ./poll-value-mbit.escript 32 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 372.7 Mbit/s Avg 204.2 Mbit/s Ratio 1.8 @ {22,30,9}
Max 305.6 Mbit/s Avg 166.9 Mbit/s Ratio 1.8 @ {22,30,10}
Max 356.0 Mbit/s Avg 192.0 Mbit/s Ratio 1.9 @ {22,30,11}
Max 410.0 Mbit/s Avg 174.4 Mbit/s Ratio 2.4 @ {22,30,12}
Max 349.7 Mbit/s Avg 187.7 Mbit/s Ratio 1.9 @ {22,30,13}
^C
root# ./poll-value-mbit.escript 16 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 441.1 Mbit/s Avg 171.8 Mbit/s Ratio 2.6 @ {22,30,19}
Max 451.1 Mbit/s Avg 179.5 Mbit/s Ratio 2.5 @ {22,30,20}
Max 352.6 Mbit/s Avg 166.3 Mbit/s Ratio 2.1 @ {22,30,21}
Max 424.7 Mbit/s Avg 177.1 Mbit/s Ratio 2.4 @ {22,30,22}
^C
root# ./poll-value-mbit.escript 8 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 598.1 Mbit/s Avg 199.4 Mbit/s Ratio 3.0 @ {22,30,30}
Max 724.1 Mbit/s Avg 205.9 Mbit/s Ratio 3.5 @ {22,30,31}
Max 637.2 Mbit/s Avg 158.8 Mbit/s Ratio 4.0 @ {22,30,32}
Max 727.2 Mbit/s Avg 187.3 Mbit/s Ratio 3.9 @ {22,30,33}
Max 832.3 Mbit/s Avg 221.8 Mbit/s Ratio 3.8 @ {22,30,34}
Max 436.6 Mbit/s Avg 162.6 Mbit/s Ratio 2.7 @ {22,30,35}
^C
root# ./poll-value-mbit.escript 4 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 826.5 Mbit/s Avg 200.2 Mbit/s Ratio 4.1 @ {22,30,41}
Max 858.9 Mbit/s Avg 166.1 Mbit/s Ratio 5.2 @ {22,30,42}
Max 746.8 Mbit/s Avg 176.0 Mbit/s Ratio 4.2 @ {22,30,43}
Max 838.7 Mbit/s Avg 190.0 Mbit/s Ratio 4.4 @ {22,30,44}
Max 674.0 Mbit/s Avg 173.5 Mbit/s Ratio 3.9 @ {22,30,45}
^C
root# ./poll-value-mbit.escript 2 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 950.2 Mbit/s Avg 188.9 Mbit/s Ratio 5.0 @ {22,30,51}
Max 960.0 Mbit/s Avg 197.2 Mbit/s Ratio 4.9 @ {22,30,52}
Max 992.8 Mbit/s Avg 236.1 Mbit/s Ratio 4.2 @ {22,30,53}
Max 997.5 Mbit/s Avg 191.7 Mbit/s Ratio 5.2 @ {22,30,54}
Max 962.5 Mbit/s Avg 172.5 Mbit/s Ratio 5.6 @ {22,30,55}
Max 957.5 Mbit/s Avg 187.6 Mbit/s Ratio 5.1 @ {22,30,56}
^C
root# ./poll-value-mbit.escript 1 /sys/devices/pci0000:00/0000:00:01.0/0000:0a:00.0/net/eth0/statistics/rx_bytes
Max 1645.6 Mbit/s Avg 182.2 Mbit/s Ratio 9.0 @ {22,31,2}
Max 1610.0 Mbit/s Avg 212.0 Mbit/s Ratio 7.6 @ {22,31,3}
Max 1623.1 Mbit/s Avg 206.3 Mbit/s Ratio 7.9 @ {22,31,4}
Max 1522.3 Mbit/s Avg 173.5 Mbit/s Ratio 8.8 @ {22,31,5}
Max 1607.7 Mbit/s Avg 199.0 Mbit/s Ratio 8.1 @ {22,31,6}
```

It’s clear that as the polling period gets smaller, the ratio between maximum observed incoming rate and average rate gets larger.  It’s also clear that we don’t have good enough timer resolution for 1 millisecond resolution.  2 millisecond resolution might be iffy, **however**, a separate experiment to test Erlang timer resolution at 2 milliseconds shows that the accuracy is within roughly 10%.

### What happens to Erlang programs when TCP incast strikes?

When a few packets are dropped, TCP does a good job of figuring out which packet(s) needs retransmission.  The TCP stack in Linux 2.6.32 seems to do well enough.  At low packet loss rates, throughput isn’t harmed much, and latency penalties are minimal.

At higher packet loss rates, however, you can hit “[TCP slow start](http://en.wikipedia.org/wiki/Slow-start)”.  In the example above, where throughput fluctuates from 952 Mbit/sec down to 8 Mbit/sec, that’s what I believe happened.  (I don’t have proof, alas: I didn’t have a packet capture running at that time.)

Between any two Erlang nodes **A** and **B**, there is a single TCP connection that carries all Erlang messages between **A** and **B**.  If there is severe congestion between **A** and **B**, and TCP slow start happens on the A-to-B TCP connection, messaging between **A** and node **C** will not be impacted.

However, there’s a very strong backpressure/feedback mechanism built in to Erlang that is related to output to “ports”.  An Erlang port is a gateway to the outside world, e.g. file descriptors to TCP sockets and to local file systems.  Each port has a buffer associated with it.  If an Erlang process writes data to the port until the buffer is full, then that process will be descheduled: the process cannot run until the buffer is no longer full.  If Erlang system monitoring for busy ports is enabled, a busy_port message will be sent to the system monitor process.

If an Erlang process **P** on node **A** attempts to send a message to process **R** on node **B**, and if the A-to-B port’s buffer is full, then process **P** will be descheduled. If Erlang system monitoring for busy network distribution ports is enabled, a busy_dist_port message will be sent to the system monitor process.

In an application like Riak, however, the process scheduling problem caused by a busy_dist_port event is very costly.

```
 * Assume process **P** on node **A** is a Riak KV vnode process.  This process is, for discussion purposes, an Erlang/OTP gen_server process.  Like all gen_server processes, it handles all requests serially.
 * Assume process **R** (the soon-to-be-message-receiver in this example) on node **B** is a Riak client 'get' operation FSM (finite state machine) process.
 * Assume that the TCP connection between **A** and **B** is congested.  In fact, it's so congested that the buffer for the port for A-to-B messaging is completely full.
 * When process **P** gets **R**'s 'get' request, it does its normal computation: pass the request to the riak_kv vnode handler, then down to (for example) the Bitcask local storage manager.  Eventually, a reply is calculated and ready to send back to the client, **R**.
 * Our server process **P** eventually uses the Erlang ! operator to send the reply to **R**.
 * The A-to-B network distribution port buffer is 100% full.  The VM sends a busy_dist_port message to the system monitor process (which will write the event to the Riak application log file), and process **P** is descheduled.
```

Given this set of events, then our Riak vnode process **P** cannot answer any more queries until the A-to-B network link becomes uncongested.   If the cluster has 12 machines in it, _ P cannot process any queries from the other 10 machines in the cluster_ … even if the Ethernet ports to those other 10 machines are 0% utilized.  Service to those other 10 machines will be delayed for as long as it takes for the A-to-B TCP connection’s data to start flowing again.  If TCP slow start is triggered, the Linux TCP stack’s default timeout before starting TCP slow start is 200 milliseconds.  And it can take another fraction of a second longer (at least) before full bandwidth can be utilized … which also assumes that there are no more packets dropped while TCP climbs out of slow start mode … which we now know is a faulty assumption.

(Careful readers will probably draw a connection from the process descheduling problem to a more general [head-of-line blocking](http://en.wikipedia.org/wiki/Head-of-line_blocking) problem.)

This causes Riak really big headaches.  Suddenly Riak query latencies become extremely unpredictable.  Overall memory usage can rise dramatically, 25% or more, for short periods of time and then fall back to normal as queues eventually drain.

### What’s the remedy?

We don’t have a good remedy for this yet.  Options include:

```
 * Using the [DCTCP](http://research.microsoft.com/pubs/121386/dctcp-public.pdf) protocol isn't an option for most [Basho](http://www.basho.com/) customers.
 * Enabling support for [Ethernet "pause frames"](http://en.wikipedia.org/wiki/Ethernet_flow_control) assumes that your switches support them correctly -- transmission between switches is apparently not well supported.
 * Changing the 200ms TCP "RTO" timer (the retransmission timeout timer) may have a beneficial effect, but it's difficult to measure because the Linux 2.6.32 kernel's implemention for changing the RTO timer is buggy.
 * Changing other Linux TCP and NIC configuration knobs (e.g. TX queue, firmware ring size & interrupt rates, MTU 9000) have negligible effect.
 * Placing limits on all cluster members outgoing bandwidth to 80% of line rate (which might reduce the ability of any single cluster member to cause TCP incast packet drops on any single receiver) appears difficult. Documentation for the Linux tc utility and the "token bucket filter" mechanism suggests that TBF might be 100% accurate up to 10 Mbits/sec.  Um, I need something that can handle a couple orders of magnitude more traffic than that.  And I'd like to be able to configure it without using units like packets/jiffy.
```

We also have [Bug 1309](https://issues.basho.com/show_bug.cgi?id=1309) open, to work around the worst of the head-of-line blocking problem.

### Postscript

Many thanks to Matt Ranney at Voxer and his great staff for assisting in troubleshooting this networking problem.  The amount of time that he spent on the phone with various data center support staff is … I don’t want to try to sum it all up.  When we finally stumbled across the theory of the TCP incast pattern, my initial reaction was (paraphrasing), “That’s bullshit.”  But, indeed, I was wrong.  All subsequent research points to TCP incast as our main problem.

### Update: 2012-Jan-22

There’s also a good collection of papers about the TCP incast pattern at the [CMU PDL Projects](http://www.pdl.cmu.edu/Incast/) page.

---
1. 见书 P340
2. 原文链接 http://www.snookles.com/slf-blog/slf-blog/2012/01/05/tcp-incast-what-is-it/
