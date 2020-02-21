+++
date = "2019-06-11T19:45:58+01:00"
draft = true
title = "TCPdump"
tags        = [ "tcpdump", "networking"]
+++

# What you need

- go installed
- Tcpdump and pcap library `sudo apt install tcpdump libpcap-dev`

# List known interfaces

    $ ip link show
   
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN mode DEFAULT group default qlen 1000
    link/ether e8:6a:64:de:a8:59 brd ff:ff:ff:ff:ff:ff
    3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
    link/ether a4:c3:f0:3e:1f:e8 brd ff:ff:ff:ff:ff:ff
    4: vpn0: <POINTOPOINT,MULTICAST,NOARP> mtu 1406 qdisc fq_codel state DOWN mode DEFAULT group default qlen 500
    link/none 
	
# Dump all traffic from wifi using tcpdump

 *Need sudo

    $ sudo tcpdump -i wlp3s0
