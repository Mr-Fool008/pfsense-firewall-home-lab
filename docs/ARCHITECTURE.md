# Lab Architecture Deep Dive

This document explains how traffic moves through the pfSense home lab and why each network component exists.

## Topology

```text
                         Internet
                            |
                      Home Router
                     192.168.4.1
                            |
              Home/WAN Network 192.168.4.0/22
                   /                    \
                  /                      \
        Kali Linux                     pfSense WAN
        192.168.4.103                  192.168.4.104
        (Attacker)                          |
                                            | Routing / Firewall
                                            |
                                      pfSense LAN
                                      192.168.1.1/24
                                            |
                                      VirtualBox LabNet
                                      192.168.1.0/24
                                            |
                                      Ubuntu Desktop
                                      192.168.1.100
                                      (Protected Host)
```

## Why Two Different Networks?

The WAN and LAN use different IP networks on purpose.

- WAN side: `192.168.4.0/22`
- LAN side: `192.168.1.0/24`

This forces traffic between Kali and Ubuntu to pass through pfSense instead of allowing the hosts to communicate directly on one flat network.

That makes pfSense the control point for:

- routing
- firewall policy
- DHCP
- DNS
- traffic logging

## VirtualBox Network Modes

### Bridged Adapter

The bridged adapter connects a VM to the same network as the physical host.

In this lab, Kali and the pfSense WAN interface are bridged. They appear as separate systems on the home network.

### Internal Network

The `LabNet` internal network exists only inside VirtualBox.

Ubuntu is connected only to this network, so it cannot reach the external network without using pfSense as its gateway.

This is what creates the protected LAN.

## pfSense as the Gateway

Ubuntu uses:

```text
Default gateway: 192.168.1.1
```

That address belongs to the pfSense LAN interface.

When Ubuntu sends traffic to a destination outside `192.168.1.0/24`, it forwards the packet to pfSense. pfSense then decides whether the traffic is allowed and where it should go next.

A simplified outbound flow looks like this:

```text
Ubuntu
192.168.1.100
    |
    v
pfSense LAN
192.168.1.1
    |
    | firewall decision
    | routing / NAT
    v
pfSense WAN
192.168.4.104
    |
    v
Home Router
    |
    v
Internet
```

## DHCP

pfSense provides DHCP service to the internal LAN.

Configured pool:

```text
192.168.1.100 - 192.168.1.199
```

Ubuntu received `192.168.1.100/24` from pfSense.

DHCP also supplied important network settings such as the default gateway and DNS server.

Receiving an IP address confirmed that the Ubuntu-to-pfSense LAN connection and DHCP service were working.

## DNS and Unbound

Ubuntu was configured to use pfSense as its DNS server:

```text
DNS server: 192.168.1.1
```

pfSense uses the Unbound DNS Resolver service.

During the lab, Ubuntu could successfully ping `8.8.8.8` but could not resolve `google.com`.

That difference was important:

```text
Ping gateway         -> working
Ping Internet IP     -> working
Resolve hostname     -> failing
```

Because routing already worked, the problem was narrowed to DNS.

Unbound was stopped because the pfSense Domain field was empty. Setting the domain to `home.arpa` allowed the resolver to start and restored name resolution.

The main troubleshooting lesson was to test network functions separately instead of treating every connectivity problem as the same issue.

## Why Kali Needed a Static Route

Kali was connected to the WAN-side network:

```text
Kali: 192.168.4.103/22
```

Ubuntu was on a different network:

```text
Ubuntu: 192.168.1.100/24
```

Kali did not automatically know that the `192.168.1.0/24` network existed behind pfSense.

The route below explicitly tells Kali where to send packets destined for the protected LAN:

```bash
sudo ip route add 192.168.1.0/24 via 192.168.4.104
```

This means:

```text
To reach 192.168.1.0/24
send packets to 192.168.4.104
```

where `192.168.4.104` is the pfSense WAN interface.

Without that route, Kali would normally send the traffic toward its default gateway instead of pfSense.

## Kali-to-Ubuntu Packet Flow

When Kali sends a packet to Ubuntu:

```text
Kali
192.168.4.103
    |
    | destination: 192.168.1.100
    v
pfSense WAN
192.168.4.104
    |
    | firewall evaluates WAN rules
    | routing table selects LAN
    v
pfSense LAN
192.168.1.1
    |
    v
Ubuntu
192.168.1.100
```

pfSense is therefore both the router and the policy enforcement point.

## Firewall Rule Order

pfSense evaluates interface firewall rules in order.

A specific block rule should appear before a broader allow rule when the goal is to deny that traffic.

Example:

```text
1. BLOCK  Kali -> Ubuntu
2. ALLOW  broader test traffic
```

If the broader allow rule matched first, the later block rule would never be reached.

This is why rule order matters as much as the rule contents.

## Before and After Mitigation

Before blocking:

```text
Kali -- SYN --> pfSense -- SYN --> Ubuntu
                                Wireshark sees packets
```

After blocking:

```text
Kali -- SYN --> pfSense --X       Ubuntu
                  BLOCK           Wireshark sees nothing
```

The mitigation was validated from three places:

1. Kali generated the packets.
2. The pfSense block-rule counter increased.
3. Ubuntu no longer observed those packets in Wireshark.

Using all three gives stronger evidence than relying on only one tool.

## Troubleshooting Model

A useful way to reason about this lab is layer by layer:

```text
1. Does the host have an IP address?
2. Can it reach its gateway?
3. Can it reach another network by IP?
4. Can it resolve DNS names?
5. Is the expected route present?
6. Does the firewall rule match the real traffic?
7. Does packet capture confirm what actually happened?
```

This approach helped isolate both the DNS failure and the firewall-rule mismatch without randomly changing unrelated settings.

## Key Takeaway

The main lesson from the architecture is that routing, DNS, DHCP, firewall rules, and packet captures are separate pieces of the same traffic path.

Understanding where a packet should go at each step makes troubleshooting much faster than changing settings until something works.
