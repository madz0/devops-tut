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

### Redirect incomming traffic to local port
```bash
iptables -t nat -I PREROUTING --src 0/0 -p udp --dport 514 -j REDIRECT --to-ports 1516
```
### Redirecting locally generated packets
```bash
iptables -t nat -I OUTPUT --src 0/0 --dst 127.0.0.1. -p tcp --dport 80 -j REDIRECT --to-ports 8080
```
*you need CONFIG_IP_NF_NAT_LOCAL=y in your kernel. Without it this rule can be inserted, but will have no effect*

## Moving lattest 3 commit to a new branch

```bash
git checkout -b newbranch # switch to a new branch
git branch -f master HEAD~3 # make master point to some older commit
```
for previous commit you can only use `HEAD~`

## git cherry-pick

```bash
git checkout master
git cherry-pick commit-1 commit-2 ...
```
if there where any confilicts, resolve them and then do:
```bash
git cherry-pick --continue
```
[from](https://www.previousnext.com.au/blog/intro-cherry-picking-git)

## git rebase

```bash
git checkout source
git rebase master
git checkout master
git merge source
```
or without needing to check out source target first:
```bash
git rebase master server
git checkout master
git merge source
```
