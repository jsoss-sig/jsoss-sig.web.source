+++
date = "2017-08-09T16:48:48+09:00"
title = "S2-048: CVE-2017-9791(Struts2) PoC with SELinux"
draft = false

+++

We did another "Famous" Struts2 vulnerability(CVE-2017-9791) PoC to check how SELinux can mitigate that vulnerability.

(Written by Kazuki Omo:ka-omo@sios.com).

## Prepare for PoC

Here is a description how to reproduce it. **I used Fedora25 image for the PoC.
I used VMWare Guest(CPU: 1, Memory: 2GB) for the PoC.**
Actually, this PoC environment is almost same as Previous vulnerability (CVE-2017-5638 which we did on June.).
Also, I used selinux-policy-targeted-3.13.1-225.11.fc25.noarch because previous policy had un-confined tomcat_t policy(See http://www.secureoss.jp/post/omok-selinux-struts2-20170607/).

  1.  Install tomcat and related packages for working Struts2.

  2.  Download and install vulnerable version of Struts2. I used both of struts-2.5.10. Copy struts2-showcase.war under /var/lib/tomcat/webapps
        
        root@fedora25:~# ls /var/ls /var/lib/tomcat/webapps/*war
        /var/lib/tomcat/webapps/struts2-showcase.war

  3.  Download and copy the PoC code on remote. There are many sample site for the PoC, then I'm not explaining it in here. 

  4.  To avoid normal Unix permission check for the PoC, I changed /etc/shadow permission to 755. 
	
        root@fedora25:~# ls -lZ /etc/shadow
        -rw-r--r--. root root system_u:object_r:shadow_t:s0        /etc/shadow


* * *

## PoC with no SELinux(SELinux Permissive)

  1.  Confirm SELinux is Permissive mode;
        
        root@fedora25:~# getenforce
        Permissive
        

  2.  Run PoC from remote host(jssosug@vmhost);
        
        
         jsossug@vmhost:~$ python Struts048.py http://172.16.148.147:8080/struts2-showcase/integration/saveGangster.action "cat /etc/shadow"

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

        root@fedora25:~# getenforce
        Permissive

  2. Run PoC from remote same as before;

        jsossug@vmhost:~$ python Struts048.py http://172.16.148.147:8080/struts2-showcase/integration/saveGangster.action "cat /etc/shadow"
        cmd: cat /etc/shadow
        
        cat: /etc/shadow: Permission denied

   3. Check AVC log on Struts PC;

        type=AVC msg=audit(1598882036.160:219): avc:  denied  { read } for  pid=4413 comm="cat" name="shadow" dev="dm-1" ino=34456196 scontext=system_u:system_r:tomcat_t:s0 tcontext=system_u:object_r:shadow_t:s0 tclass=file
        
* * *
## Conclusion

From this PoC we can say

1. SELinux can mitigate Struts2 vulnerability "if Policy is updated".;
2. Last SELinux Policy is treating "tomcat_t" as "unconfined domain".
3. Latest version of SELinux Policy will solve the problem.
