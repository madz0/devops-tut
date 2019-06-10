# Devops

## To apply `/etc/sysctl.conf` rules without restart system:

```bash
sysctl -p
sysctl --system
```
## Enable forwarding only to specific interface:

```bash
echo "1" > /proc/sys/net/ipv4/conf/eth0/forwarding
```
## Iptables rules:

* To redirect incomming traffic means inserting rules into PREROUTING chain of the nat table
* Use the REDIRECT target, which allows you to specify destination port(s) (--to-ports)
* you need CONFIG_IP_NF_NAT_LOCAL=y in your kernel. Without it this rule can be inserted, but will have no effect
