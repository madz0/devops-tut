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

## Maven broken local repository

If `mvn clean dependency:tree` is ok but `mvn clean compile` fails and you beleive you've added all dependencies in the pom,
then chanses are, you have a broken local repository. 

To fix do:

`mvn dependency:purge-local-repository`

If that does not work, try to remove `~/.m2/repository/some/package`

Note: To find out where the local repository is, in eclipse expand Maven Dependencies and right click on one of t
he libraries and go to build path

## Maven java version

```
<properties>
     <java.version>1.8</java.version>
</properties>
```
Is only specific to spring boot.

```
<plugins>
    <plugin>    
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
            <source>1.8</source>
            <target>1.8</target>
        </configuration>
    </plugin>
</plugins>
```
or
```
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>
```
Are standard and equivalent way to specify java vesion for maven compiler plugin

The maven-compiler-plugin 3.6 and later versions provide a new way :
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.0</version>
    <configuration>
        <release>9</release>
    </configuration>
</plugin>
```
You could also declare just :
```
<properties>
    <maven.compiler.release>9</maven.compiler.release>
</properties>
```
The Maven `release` argument conveys  release : a new JVM standard option that we could pass from Java 9 :
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.0</version>
    <configuration>
        <release>9</release>
    </configuration>
</plugin>
```
[From](https://stackoverflow.com/a/38883073/2556354)

## Git discard changes to a file

```bash
git checkout -- <file>
```
For a specificrevision:
```bash
git checkout <commit_hash> -- <file>
```

## See diff after changing a file before committing

```bash
git diff path/to/file
```
