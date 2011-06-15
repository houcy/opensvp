============================================
Opensvp and application layer gateway attack
============================================

Introduction
============

Opensvp is a security tool implementing some new attack. Its aim is to provide
a tools to test the resistance of firewall to protocol level attack. It
implements some new kind of attack against application layer gateway.

For example, opensvp is able under some conditions (see explanation
below for details) to open a pin hole in a firewall protecting a
ftp server: even if the filtering policy garantee that only the 21
port is open to the server, you can open 'any' port on the server
by using opensvp.

Lets have 192.168.2.3 a server running ftp, placed behind a firewall.
If the user, as root, runs::

 ./opensvp.py --server 192.168.2.3 --helper ftp --port 23 -v -i eth0

Then he will have a temporary access on port 23 of the server independantly
of the firewall rules.


Attack Description
==================
Principle
---------

Some network protocols are using multiple connections  for the exchange
between a client and a server. The most known example is ftp where command
goes through a connection on port 21 and where data exchange are done with
two different mode (connection from port 20 or dynamic connection).

Some firewall implementation implement application layer gateway (ALG) to be
able to detect this parallel connection and be able to autorize them dynamically.
Other solutions are to use application relay (transparent proxy) or to open
all the possible flow (read almost everything).

The ALG analyse the traffic and detect and parse the command sent between the
peers to declare the parameters of the parallel connections. Once done they
open temporary pin hole in the firewall to let the probable traffic goes through.

The idea of this attack is to forge this type of messages to open pin hole in
the firewall but pin hole that should not have been open.


Condition:
 * Attacker computer is on a network directly connected to the firewall.
 * Firewall is sensible to the attack (for example, Netfilter with rp_filter
   set to 0)
 * Attacker is able to sniff data packet (or by pcap sniffing or by running
   himself a data connection)

The cinematic is the following :
 1. Sniffer on the attacker network capture one packet from the protocol flow

     * it reverse the ethernet dst and src
     * it increase id in IP and seq for TCP
     * it set payload to the wanted command (with selected
       port)

 2. The forged packet is sent on the interface connected to the firewall
 3. Firewall transmit the packet back to the client and is now expecting
    a packet with caracteristic based on attacker input

Attacking IRC
-------------

This attack is a direct application of the described principle. Once data packet
is received, the attacker send a forged DCC command.

Attacking FTP
-------------

In this attack, the client connection is open by the attacker. He connect to the
ftp server behind a firewall and initiate a real connection. Once the session is
setup, he launch the attack by sending a forged 227 command.

If IPv6 is used, the same attack is done with a forged 229 command.

Impact of the attack
====================
Possible target
---------------

The main contraint about these attack is that the attacker has to be on a network
directly connected to the firewall.

Thus, the main possibilities are:
 * Attack from a user LAN
 * Attack in a hosting farm

Both case can lead to severe information exposure by giving the attacker access to
unprotected services.

Linux
-----

This attack is known to work on IPv4 Netfilter firewall if rp_filter is set to
0 (this is hopefully not the default value).

There is currently no reverse path filtering implementation for IPv6, the firewall
is thus not protected and the protection has to be setup in the firewall rules (see
next chapter).

Some firewall software are known to be vulnerable:
 * fwbuilder: a specific policy has to be set up
 * shorewall: vulnerable, developpement is in progress to fix it
 * edenwall: vulnerable

The attack works for both gateway and local firewall. On a local firewall, FORWARD
filtering has to be activated and a ESTABLISHED ACCEPT rules has to be set up on
this chain. This could be the case of system running virtual machine.

Defense against the attack
==========================
Linux
-----

rp_filter is enough for protection in IPv4. Check that you have rp_filter set
to 1 on all the interfaces that can be subject to the attack. For IPv6, the
situation is more complicated, the solution is to block the transmission of
the attack on the firewall (the expectation is created when the packet get
out of the box). Thus it is possible to use something like the following rule
for each interface ::

 ip6tables -I FORWARD -i eth0 -o eth0 -j DROP

This rule has to be put before any accept all packet from ESTABLISHED connection.

A similar but less intrusive way to do is the work on the FORWARD mangle table ::

 iptables -N ANTISPOOF
 iptables -A ANTISPOOF -j NFLOG --nflog-prefix "Spoofing attempt"
 iptables -A ANTISPOOF -j DROP
 for IFACE in IFACE_LIST; do
   iptables -A ANTISPOOF -i IFACE ! -o IFACE -j RETURN
 done
 iptables -I FORWARD  -t mangle -j ANTISPOOF

Another solution is to use standard antispoofing rules : if DMZ is a network
where we have vulnerable protocol/server running, we can add ::

 ip6tables -I FORWARD -i ! $NET_IFACE -s $IP_NET -j DROP

This last solution is harder to setup because you need to know the network
topology but this is the only bullet proof solution.