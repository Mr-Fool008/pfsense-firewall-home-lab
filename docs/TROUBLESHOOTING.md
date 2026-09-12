# Troubleshooting Notes

## Ubuntu has an IP but cannot resolve domain names

### Symptoms

```bash
ping -c 4 192.168.1.1   # works
ping -c 4 8.8.8.8       # works
ping -c 4 google.com     # fails
```

Check the DNS server received by Ubuntu:

```bash
resolvectl status
```

Expected in this lab:

```text
DNS Servers: 192.168.1.1
```

Then query pfSense directly:

```bash
nslookup google.com 192.168.1.1
```

If this times out, check **pfSense > Status > Services** and verify `unbound` is running.

In this build, Unbound failed with:

```text
error parsing local-data at 2 '.. A 192.168.1.1': Empty label
error: Bad local-data RR .. A 192.168.1.1
fatal error: Could not set up local zones
```

The pfSense **System > General Setup > Domain** field was empty. Setting it to `home.arpa` allowed Unbound to start and restored DNS resolution.

## Firewall rule does not match traffic

If the rule counter remains at `0 B`, compare the rule against the actual packet flow:

- Correct interface?
- Correct source address?
- Correct destination address?
- Correct protocol and destination port?
- Is a different rule matching first?

A test sent to `192.168.4.104` cannot validate a rule whose destination is `192.168.1.100`.

## Kali cannot reach Ubuntu

Because Kali and Ubuntu are on different networks, Kali needs a route through pfSense:

```bash
sudo ip route add 192.168.1.0/24 via 192.168.4.104
```

Verify:

```bash
ip route
ping -c 4 192.168.1.100
```

Also verify that pfSense has an appropriate WAN rule permitting the test traffic before attempting the before/after mitigation demonstration.
