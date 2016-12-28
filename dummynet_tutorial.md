Using [dummynet(4)][1] for poor network conditions emulation in transparent bridge mode under FreeBSD 11.0-RELEASE
=======

1 Introduction and start
-------------
<br>
#### 1.1 Purpose of this tutorial
This tutorial will show you how configure a fresh FreeBSD 11-RELEASE installation on a machine with two or three nics to be a transparent Ethernet bridge, which can simulate poor network conditions using [dummynet(4)][1]. The [dummynet(4)][1] is a flexible FreeBSD kernel functionality, which may be used for network testing, traffic shaping and specific network conditions emulation.

#### 1.2 Obtaining FreeBSD 11.0-RELEASE
You can install FreeBSD booting your machine from [.iso][13] or [.memstick][14] image (provided links are for AMD64 arch, see the [FreeBSD ftp site][12] for more options). Burn ISO image to the CD or DVD disk or write down memstick image on USB drive using *dd* or similiar utility. Boot from selected media and install FreeBSD following the guidelines on the screen. 

You can leave all options set to default or modify them according to your needs. See [Handbook][15] for details.

It's highly recommended to update your fresh installed FreeBSD system with recent updates using [freebsd-update(8)][16]:

```
# freebsd-update fetch
# freebsd-update install 
# reboot
```

#### 1.3 Enabling dummynet.
The recommended way to enable [dummynet(4)][1] is to load it at a boot time by adding these options to the [rc.conf(5)][7]:

```
dummynet_enable="YES"
firewall_enable="YES"
firewall_type="open"
```

You need to reboot machine for options to take effect.

>Note: Loading [dummynet(4)][1] kernel module will load [ipfw(4)][2] as dependence. You can lose you network connection to the FreeBSD machine due to default denial [ipfw(4)][2] policy. To prevent that use firewall_type="open" [rc.conf(5)][7] option or add appropriate *allow* rules to your custom ruleset. See [firewall(7)][8] for details.

#### 1.4 Enabling transparent bridge mode
It's handy to use [dummynet(4)][1] along with transparent bridge configuration. You can bridge two interfaces with the following commands (the interfaces are *em0* and *em1* in this example):

```
# ifconfig bridge create
bridge0
# ifconfig bridge0 addm em0 addm em1 up
# ifconfig em0 up
# ifconfig em1 up
```

To make this bridge permanent and loadable at a boot time you can add these lines to rc.conf:

```
cloned_interfaces="bridge0"
ifconfig_bridge0="addm em0 addm em1 up"
ifconfig_em0="up"
ifconfig_em1="up"
```

> Note: this configuration doesn't assign any IP addresses to your network interfaces, so you'll not able to manage it via network. To avoid this use proper [ifconfig(8)][17] options in your [rc.conf(5)][7]. See working examples below.

Example 1, one bridge, two interfaces, static ip configuration for bridge:

```
cloned_interfaces="bridge0"
ifconfig_bridge0="inet 192.168.10.10/24 addm em0 addm em1 up"
ifconfig_em0="up"
ifconfig_em1="up"
```

Example 2, three interfaces, one bridge, two bridged interfaces, one interface for management configured staticly:

```
ifconfig_em0="inet 192.168.10.10/24"
cloned_interfaces="bridge0"
ifconfig_bridge0="addm em1 addm em2 up"
ifconfig_em1="up"
ifconfig_em2="up"
```

Example 3, two interfaces, one bridge, management IP on the bridge, [STP][18] enabled:

```
cloned_interfaces="bridge0"
ifconfig_bridge0="inet 192.168.10.10/24 addm em0 addm em1 stp em0 stp em1 up"
ifconfig_em0="up"
ifconfig_em1="up"
```

> Note: use your interface names instead of *em\** from examples. You can use options like `ifconfig_em0="DHCP"` to configure interface via DHCP.

#### 1.5 Related sysctl variables
This section contains sysctl variables which affect dummynet usage. You should save them to [sysctl.conf(5)][6] to make them permanent.
<br>
Enable ipfw to process L2 traffic (Ethernet frames):

```
# sysctl net.link.ether.ipfw=1
net.link.ether.ipfw: 0-> 1
```

Enable ipfw for bridge interface:

```
# sysctl net.link.bridge.ipfw=1
net.link.bridge.ipfw: 0 -> 1
```

In advanced cases you may want to process ARP traffic with ipfw as well:

```
# sysctl net.link.bridge.ipfw_arp=1
net.link.bridge.ipfw_arp: 0 -> 1
```

Make sure the *fast* mode of [dummynet(4)][1] is disabled (default, see details at [ipfw(8)][3]):

```
# sysctl net.inet.ip.dummynet.io_fast=0
net.inet.ip.dummynet.io_fast: 0 -> 0
```

In order to collect pipe statistics you should disable autodeletion of unused dummynet queues:

```
# sysctl net.inet.ip.dummynet.expire=0
net.inet.ip.dummynet.expire 1 -> 0
```


2 Usage
-------------
<br>
#### 2.1 Firewall ruleset
You can utilize dummynet functionality using [ipfw(8)][3]. It's a good practice to test any new firewall rules by adding them from command prompt first, save them after testing to the file */etc/firewall.rules* and load them at a boot time with 

```
firewall_type="/etc/firewall.rules"
```

in rc.conf. The file should contain ipfw rules without ipfw command, e.g.:

```
pipe 1 config bw 100Kbit/s delay 100 plr .01
pipe 2 config bw 500Kbit/s delay 250 plr .03
add 1000 prob 0.33 pipe 1 ip from any to any out
add 1100 prob 0.5 pipe 1 ip from any to any out
add 65000 allow ip from any to any
```

See the corresponding sections of [ipfw(4)][2], [ipfw(8)][3] and [firewall(7)][8] for details. 

There are three dummynet ipfw subcommands: *pipe*, *queue* and *sched*. We will use *pipes* in this tutorial. Read the [TRAFFIC SHAPER (DUMMYNET) CONFIGURATION][9] section of [ipfw(8)][3] for detailes.

#### 2.2 Pipes
Pipe is the powerfull way to apply specific network conditions to the traffic flow. You can define *pipe N* with [ipfw(8)][3]:

```
# ipfw pipe 1 config delay 100ms plr .05
# ipfw pipe 2 config bw 200KB/s delay 200ms
```

Each pipe should have one or more config option, defining bandwidth, network latency, packet loss and so on. The *bw* option defines bandwidth in Kbit/s, Mbit/s, KByte/s, MByte/s:

```
# ipfw pipe 1 config bw 1MByte/s
# ipfw pipe 2 config bw 200Kbit/s
```

The *delay* option defines network latency in milliseconds:

```
# ipfw pipe 1 config delay 400
# ipfw pipe 2 config delay 100
```

The *plr* (packet-loss-rate) option defines packet loss (.08=8%, .33=33% and so on):

```
# ipfw pipe 1 config plr .01
# ipfw pipe 2 config plr .15
```

You can combine options in one pipe:

```
# ipfw pipe 1 config bw 1Mbit/s delay 100ms plr .05
# ipfw pipe 2 config bw 200KByte/s delay 200ms plr .02
```

#### 2.3 Route traffic to the pipes and multipath simulation
You can route traffic to the pipe on any stage of going through your dummynet-enabled server, but the good practice is to apply pipe rules on leaving the system:

```
# ipfw add 1000 pipe 1 ip from any to any out
```

You can implement multipath simulation with probability *prob* setting (.3=30% and so on). Here we have 20% of probability to hit *pipe 1*, then the 50% from rest 80%, which is 40% to hit *pipe 2* and the rest 40% - *pipe 3*:

```
# ipfw add 1000 prob .2 pipe 1 ip from any to any out
# ipfw add 1100 prob .5 pipe 2 ip from any to any out
# ipfw add 1200 pipe 3 ip from any to any out
```

#### 2.4 Sample config
So, let's put it all together. You should have these lines in */etc/sysctl.conf*:

```
net.link.bridge.ipfw=1
net.link.ether.ipfw=1
net.inet.ip.dummynet.expire=0
```

Something like this in */etc/rc.conf* (three NICs case):

```
ifconfig_em0="DHCP"
cloned_interfaces="bridge0"
ifconfig_bridge0="addm em1 addm em2 up"
ifconfig_em1="up"
ifconfig_em2="up"
dummynet_enable="YES"
firewall_enable="YES"
firewall_type="/etc/firewall.rules"
```

And something like this in */etc/firewall.rules*:

```
add 500 allow ip from any to any via em0
pipe 1 config bw 100Kbit/s delay 300ms plr .05
pipe 2 config bw 200KByte/s delay 200ms plr .02
pipe 3 config bw 1Mbit/s delay 100ms plr .05
add 1000 prob .2 pipe 1 ip from any to any out
add 1100 prob .5 pipe 2 ip from any to any out
add 1200 pipe 3 ip from any to any out
add 65000 allow ip from any to any
```

#### 2.5 Testing

Let's say we have the configuration from 2.4, the 192.168.33.0/24 network divided into two segments by our FreeBSD-based transparent bridge powered by dummynet. Let's do the ping (consider that the pipe rules apply for both echo request and echo reply packets):

```
# ping -c 100 192.168.33.2
PING 192.168.33.2 (192.168.33.2): 56 data bytes
64 bytes from 192.168.33.2: icmp_seq=0 ttl=64 time=509.361 ms
64 bytes from 192.168.33.2: icmp_seq=1 ttl=64 time=401.652 ms
64 bytes from 192.168.33.2: icmp_seq=2 ttl=64 time=402.065 ms
64 bytes from 192.168.33.2: icmp_seq=3 ttl=64 time=301.228 ms
64 bytes from 192.168.33.2: icmp_seq=4 ttl=64 time=407.986 ms
64 bytes from 192.168.33.2: icmp_seq=5 ttl=64 time=410.086 ms
64 bytes from 192.168.33.2: icmp_seq=6 ttl=64 time=202.868 ms
64 bytes from 192.168.33.2: icmp_seq=7 ttl=64 time=402.346 ms
64 bytes from 192.168.33.2: icmp_seq=8 ttl=64 time=400.921 ms
64 bytes from 192.168.33.2: icmp_seq=10 ttl=64 time=301.975 ms
[...skip...]
64 bytes from 192.168.33.2: icmp_seq=90 ttl=64 time=201.869 ms
64 bytes from 192.168.33.2: icmp_seq=91 ttl=64 time=617.660 ms
64 bytes from 192.168.33.2: icmp_seq=92 ttl=64 time=202.019 ms
64 bytes from 192.168.33.2: icmp_seq=93 ttl=64 time=507.822 ms
64 bytes from 192.168.33.2: icmp_seq=94 ttl=64 time=302.496 ms
64 bytes from 192.168.33.2: icmp_seq=95 ttl=64 time=300.705 ms
64 bytes from 192.168.33.2: icmp_seq=96 ttl=64 time=302.309 ms
64 bytes from 192.168.33.2: icmp_seq=97 ttl=64 time=301.775 ms
64 bytes from 192.168.33.2: icmp_seq=98 ttl=64 time=402.769 ms
64 bytes from 192.168.33.2: icmp_seq=99 ttl=64 time=510.121 ms

--- 192.168.33.2 ping statistics ---
100 packets transmitted, 93 packets received, 7.0% packet loss
round-trip min/avg/max/stddev = 201.118/360.326/617.660/106.197 ms
```

Starting [iperf3][19] server on 192.168.33.2 machine (could be installed with `pkg install iperf3`):

```
# iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

Gathering network stats with iperf3 client on 192.168.33.1:

```
# iperf3 -c 192.168.33.2
Connecting to host 192.168.33.2, port 5201
[  4] local 192.168.33.1 port 50901 connected to 192.168.33.2 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.01   sec  38.4 KBytes   313 Kbits/sec    1   2.02 MBytes       
[  4]   1.01-2.01   sec  15.6 KBytes   126 Kbits/sec    2   10.1 MBytes       
[  4]   2.01-3.02   sec  15.6 KBytes   126 Kbits/sec    0   12.1 MBytes       
[  4]   3.02-4.03   sec  11.3 KBytes  92.2 Kbits/sec    3   4.05 MBytes       
[  4]   4.03-5.01   sec  8.48 KBytes  70.8 Kbits/sec    0   8.10 MBytes       
[  4]   5.01-6.02   sec  9.90 KBytes  80.8 Kbits/sec    0   10.1 MBytes       
[  4]   6.02-7.03   sec  9.90 KBytes  80.3 Kbits/sec    1   4.05 MBytes       
[  4]   7.03-8.02   sec  8.48 KBytes  69.7 Kbits/sec    1   2.02 MBytes       
[  4]   8.02-9.04   sec  7.07 KBytes  57.0 Kbits/sec    0   2.03 MBytes       
[  4]   9.04-10.06  sec  0.00 Bytes  0.00 bits/sec    1   4.03 MBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.06  sec   125 KBytes   102 Kbits/sec    9             sender
[  4]   0.00-10.06  sec  93.3 KBytes  76.0 Kbits/sec                  receiver

iperf Done.
```


3 Statistics and graphs
-------------
<br>
#### 3.1 Statistics

You can get current pipe config and stats with `ipfw pipe show` or `ipfw pipe N show` where N is a number of pipe:

```
# ipfw pipe 1 show
00001: 100.000 Kbit/s  300 ms burst 0 
q131073  50 sl.plr 0.050000 0 flows (1 buckets) sched 65537 weight 0 lmax 0 pri 0 droptail
 sched 65537 type FIFO flags 0x0 0 buckets 1 active
BKT Prot ___Source IP/port____ ____Dest. IP/port____ Tot_pkt/bytes Pkt/Byte Drp
  0 ip           0.0.0.0/0             0.0.0.0/0      130    68642  0    0   6
```

A little more human friendly stats with simple shell script:

```
#!/bin/sh

tmp_file=`mktemp`

for pipe_ in `ipfw pipe show | grep \: | cut -d ':' -f 1`; do
        ipfw pipe show ${pipe_} > ${tmp_file}
        total_packets=`tail -1 ${tmp_file} | awk '{ print $5 }'`
        total_bytes=`tail -1 ${tmp_file} | awk '{ print $6 }'`
        dropped_packets=`tail -1 ${tmp_file} | awk '{ print $9 }'`
        echo "Pipe ${pipe_} stats: "
        echo "---------------------"
        echo "Total packets: ${total_packets}"
        echo "Total bytes: ${total_bytes}"
        echo "Dropped packets: ${dropped_packets}"
        echo
done

rm -f ${tmp_file}
```

#### 3.2 RRDtool

You can create Round-robin database and gather stats there with [rrdtool][11]. Let's say we are creating DB for three pipes to store the number of total bytes passed through:

```
# mkdir -p /var/db/rrd && \
rrdtool create /var/db/rrd/pipes.rrd --step 60 \
DS:totalbytes1:COUNTER:600:0:999999999 \
DS:totalbytes2:COUNTER:600:0:999999999 \
DS:totalbytes3:COUNTER:600:0:999999999 \
RRA:AVERAGE:0.5:1:525600
```

Update script:

```
#!/bin/sh

totalbytes1=`ipfw pipe show 1 | tail -1 | awk '{ print $6 }'`
totalbytes2=`ipfw pipe show 2 | tail -1 | awk '{ print $6 }'`
totalbytes3=`ipfw pipe show 3 | tail -1 | awk '{ print $6 }'`

rrdtool update /var/db/rrd/pipes.rrd N:${totalbytes1}:${totalbytes2}:${totalbytes3}
```

Let's save it to */usr/local/bin/rrdupdate_script.sh* and add to the [crontab(5)][10]:

```
*/1	*	*	*	*	root	/usr/local/bin/rrdupdate_script.sh
```


[1]: https://www.freebsd.org/cgi/man.cgi?query=dummynet&sektion=4&manpath=FreeBSD+11.0-RELEASE
[2]: https://www.freebsd.org/cgi/man.cgi?query=ipfw&sektion=4&manpath=FreeBSD+11.0-RELEASE
[3]: https://www.freebsd.org/cgi/man.cgi?query=ipfw&sektion=8&manpath=FreeBSD+11.0-RELEASE
[4]: https://www.freebsd.org/cgi/man.cgi?query=if_bridge&sektion=4&manpath=FreeBSD+11.0-RELEASE
[5]: https://www.freebsd.org/cgi/man.cgi?query=loader.conf&sektion=5&manpath=FreeBSD+11.0-RELEASE
[6]: https://www.freebsd.org/cgi/man.cgi?query=sysctl.conf&sektion=5&manpath=FreeBSD+11.0-RELEASE
[7]: https://www.freebsd.org/cgi/man.cgi?query=rc.conf&sektion=5&manpath=FreeBSD+11.0-RELEASE
[8]: https://www.freebsd.org/cgi/man.cgi?query=firewall&sektion=7&manpath=FreeBSD+11.0-RELEASE
[9]: https://www.freebsd.org/cgi/man.cgi?query=ipfw&sektion=8&manpath=FreeBSD+11.0-RELEASE#TRAFFIC_SHAPER_(DUMMYNET)_CONFIGURATION
[10]: https://www.freebsd.org/cgi/man.cgi?query=crontab&sektion=5&manpath=FreeBSD+11.0-RELEASE
[11]: http://oss.oetiker.ch/rrdtool/
[12]: http://ftp.freebsd.org/pub/FreeBSD/releases/ISO-IMAGES/11.0
[13]: http://ftp.freebsd.org/pub/FreeBSD/releases/ISO-IMAGES/11.0/FreeBSD-11.0-RELEASE-amd64-disc1.iso
[14]: http://ftp.freebsd.org/pub/FreeBSD/releases/ISO-IMAGES/11.0/FreeBSD-11.0-RELEASE-amd64-memstick.img
[15]: https://www.freebsd.org/doc/handbook/
[16]: https://www.freebsd.org/cgi/man.cgi?query=freebsd-update&sektion=8&manpath=FreeBSD+11.0-RELEASE
[17]: https://www.freebsd.org/cgi/man.cgi?query=ifconfig&sektion=8&manpath=FreeBSD+11.0-RELEASE
[18]: https://www.freebsd.org/cgi/man.cgi?query=if_bridge&manpath=FreeBSD+11.0-RELEASE#SPANNING_TREE
[19]: https://iperf.fr/

