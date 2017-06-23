+++
date = "2017-06-08T09:53:10+09:00"
title = "CVE-2017-5638(Struts2) PoC with SELinux"
draft = false

+++

We did "Famous" Struts2 vulnerability(CVE-2017-5638) PoC to check how SELinux can mitigate that vulnerability.
During the PoC, we found current policy problem, then reported it on bugzilla.
So, I would write the information for that PoC and SELinux policy problem.

(Written by Kazuki Omo:ka-omo@sios.com).

## Prepare for PoC

Here is a description how to reproduce it. **I used CentOS 7.3(CentOS-7-x86_64-DVD-1611.iso)image for the PoC.
I used VMWare Guest(CPU: 1, Memory: 2GB) for the PoC.**

  1.  Install tomcat and related packages for working Struts2.

  2.  Download and install vulnerable version of Struts2. I used both of struts-2.5.10. Copy struts2-showcase.war under /var/lib/tomcat/webapps
        
        root@cent7enc:~# ls /var/ls /var/lib/tomcat/webapps/*war
        /var/lib/tomcat/webapps/struts2-showcase.war

  3.  Download and copy the PoC code on remote. There are many sample site for the PoC, then I'm not explaining it in here. 

  4.  To avoid normal Unix permission check, I changed /etc/shadow permission to 755. 
	
        root@cent7enc:~# ls -lZ /etc/shadow
        -rw-r--r--. root root system_u:object_r:shadow_t:s0        /etc/shadow

* * *

## PoC with no SELinux(SELinux Permissive)

  1.  Confirm SELinux is Permissive mode;
        
        root@cent7enc:~# getenforce
        Permissive
        

  2.  Run PoC from remote host(jssosug@vmhost);
        
        
         jsossug@vmhost:~$ python attack.py http://172.16.148.130:8080/struts2-showcase/showcase.action "cat /etc/shadow"
         CVE: 2017-5638 - Apache Struts2 S2-045
         cmd: cat /etc/shadow
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

        root@cent7enc:~# getenforce
        Permissive

  2. Run PoC code again. Even if SELinux is Enforcing, still you can see /etc/shadow(so bad...);
	
        jsossug@vmhost:~$ python attack.py http://172.16.148.130:8080/struts2-showcase/showcase.action "cat /etc/shadow"
        CVE: 2017-5638 - Apache Struts2 S2-045
        cmd: cat /etc/shadow
        
        root:XXXXXX.::0:99999:7:::
        bin:*:17110:0:99999:7:::
        daemon:*:17110:0:99999:7:::
        --snip--
        sshd:!!:17247::::::
        jssosug:XXXXXXXXXXXX::0:99999:7:::
        
        jsossug@vmhost:~$
	
* * *
## Check SELinux Policy

Now we understand that SELinux can't mitigate CVE-2017-6074. Why it is happened?

1. For understanding it, attack from remote(vmhost) by using below command;

        jsossug@vmhost:~$ python attack.py http://172.16.148.130:8080/struts2-showcase/showcase.action "vi /tmp/abcd" 
        CVE: 2017-5638 - Apache Struts2 S2-045
        cmd: vi /tmp/abcd

2. On Struts PC, check domain who is running "vi /tmp/abcd";
	
        root@cent7enc:~# ps axZ|grep abcd
        system_u:system_r:tomcat_t:s0         3251 ?                S          0:00 vi /tmp/abcd

3. Check "tomcat_t" inheriented domain with seinfo;

	
        root@cent7enc:~# seinfo -ttomcat_t -x
        tomcat_t
          can_change_object_identity
          can_load_kernmodule
          can_setbool
          can_setenforce
          corenet_unconfined_type
          corenet_unlabeled_type
          devices_unconfined_type
          domain
          files_unconfined_type
          filesystem_unconfined_type
          kern_unconfined
          kernel_system_state_reader
          process_uncond_exempt
          selinux_unconfined_type
          storage_unconfined_type
          unconfined_domain_type
          dbusd_unconfined
          daemon
          syslog_client_type
          sepgsql_unconfined_type
          tomcat_domain
          userdom_filetrans_type
          x_domain
          xserver_unconfined_type
        

So we found tomcat_t is in several "unconfined" domain.
I reported it on bugzilla(https://bugzilla.redhat.com/show_bug.cgi?id=1432083), then fixed version of policy(selinux-policy-3.13.1-145.el7.noarch.rpm)


* * *
## Check SELinux Updated Policy

1. Fixed version of policy is in RHEL-7.4Beta image. Just for confirmation, I installed those policy on CentOS7(PoC).

        root@cent7enc:~# rpm -Fvh selinux-policy-3.13.1-145.el7.noarch.rpm selinux-policy-targeted-3.13.1-145.el7.noarch.rpm 
        
        root@cent7enc:~# getenforce
        Enforcing

2. Run PoC from remote same as before;

        jsossug@vmhost:~$ python attack.py http://172.16.148.130:8080/struts2-showcase/showcase.action "cat /etc/shadow" 
        CVE: 2017-5638 - Apache Struts2 S2-045
        cmd: cat /etc/shadow
        
        cat: /etc/shadow: Permission denied

3. Check AVC log on Struts PC;

        type=AVC msg=audit(1496882036.860:219): avc:  denied  { read } for  pid=4413 comm="cat" name="shadow" dev="dm-1" ino=34456196 scontext=system_u:system_r:tomcat_t:s0 tcontext=system_u:object_r:shadow_t:s0 tclass=file
        
4. Check "tomcat_t" inheriented domain with seinfo;
   
        root@cent7enc:~# seinfo -ttomcat_t -x
        tomcat_t
          nsswitch_domain
          corenet_unlabeled_type
          domain
          kernel_system_state_reader
          netlabel_peer_type
          daemon
          syslog_client_type
          pcmcia_typeattr_7
          pcmcia_typeattr_6
          pcmcia_typeattr_5
          pcmcia_typeattr_4
          pcmcia_typeattr_3
          pcmcia_typeattr_2
          pcmcia_typeattr_1
          tomcat_domain
So now we can see no "Unconfined" domain in there.


* * *
## Conclusion

From this PoC we can say

1. SELinux can mitigate Struts2 vulnerability if "Policy is good.";
2. Last SELinux Policy is treating "tomcat_t" as "unconfined domain".
3. Latest version of SELinux Policy will solve the problem.
