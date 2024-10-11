# Write Up on etceterad machine


## Information Gathering And Enumeration
Let's us first start off by firing up `nmap` to discover open ports and running services on our target. <br>

```
mcsam@0x32:~/$ sudo nmap -vvv 10.0.160.122 -p- --min-rate 10000
Nmap scan report for 10.0.160.122
Host is up (0.27s latency).
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 63
1337/tcp open  waste   syn-ack ttl 63
2379/tcp open  etcd-client syn-ack ttl 63
```

From the above scan we see that there are three services running. Nmap gives us an idea of the services running on the port and from that we can see that `etcd` is running on port `2379`. There's another service running on `1337` and after doing a service scan on it we realized it's a web application and hence can be accessed via the browser.<br>

```
mcsam@0x32:~/$ sudo nmap -sV 10.0.160.122 -p 1337
Starting Nmap 7.80 ( https://nmap.org ) at 2024-10-11 09:25 GMT
Nmap scan report for 10.0.160.122
Host is up (0.22s latency).

PORT     STATE SERVICE VERSION
1337/tcp open  http    Node.js (Express middleware)
```
<br>

![etceterad website](https://raw.githubusercontent.com/theMcSam/echoCTF-writeups/refs/heads/main/etceterad/images/1337-website-etcd.png)

After a search for vulnerabilities associated with `etcd` we can find that this version of `etcd` is vulnerable to `CVE-2021-28235`.

PoC for this vulnerability can be found here: https://github.com/lucyxss/etcd-3.4.10-test/blob/master/temp4cj.png

Using this vulnerability we care able to view leaked credetials for authenticating to `etcd`.


