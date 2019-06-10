# Devops

## To apply `/etc/sysctl.conf` rules without restart system:

```bash
sysctl -p
sysctl --system
```
## Enable forwarding only to specific interface:

```bash
echo 1 > /proc/sys/net/ipv4/conf/eth0/forwarding
```
