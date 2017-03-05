+++
date = "2017-03-05T19:55:34+09:00"
title = "CVE-2017-6074 PoC with SELinux(on Ubuntu)"
draft = false

+++

We found that PoC code for CVE-2017-6074 was published since 2017/02/28. Then we did PoC with SELinux Enabled, and figure out SELinux could mitigate it or not.

(Written by Kazuki Omo:ka-omo@sios.com).

## Prepare for PoC

Here is a description how to reproduce it. **I used Ubuntu 16.04.1 LTS
with 4.4.0-62-generic x86\_64 kernel on VMWare Guest, because this PoC
code is only for that distro/version. Also I assigned only 1 CPU to that
guest.**

  1.  Install 4.4.0-62-generic kernel on Ubuntu. You can find it on any mirror site.

  2.  Prepare SELinux on Ubuntu. I prefer to use "aptitude" instead of "apt".

        root@ubuntu:~# aptitude -y install selinux
        The following NEW packages will be installed:
          checkpolicy{a} libapol4{a} libauparse0{a} libpython-stdlib{a} 
          libpython2.7-minimal{a} libpython2.7-stdlib{a} libqpol1{a} m4{a} make{a} 
          policycoreutils{a} python{a} python-audit{a} python-ipy{a} 
          python-minimal{a} python-selinux{a} python-semanage{a} python-sepolgen{a} 
          python-sepolicy{a} python-setools{a} python2.7{a} python2.7-minimal{a} 
          selinux{b} selinux-policy-default{a} selinux-policy-dev{a} 
          selinux-policy-ubuntu{ab} selinux-utils{a} setools{a} 
        0 packages upgraded, 27 newly installed, 0 to remove and 131 not upgraded.
        --snip--
        Processing triggers for initramfs-tools (0.122ubuntu8.1) ...
        update-initramfs: Generating /boot/initrd.img-4.4.0-62-generic
        W: mdadm: /etc/mdadm/mdadm.conf defines no arrays.
                                                 
        Current status: 125 (-6) upgradable.
        root@ubuntu:~# reboot

  3.  Download and copy the PoC code.

        jsossug@ubuntu:~/CVE-2017-6074$ ls
        poc.c  README.md  trigger.c
        jsossug@ubuntu:~/CVE-2017-6074$ 

  4.  Compile that code on PoC Ubuntu 16.04.1 LTS;

        jsossug@ubuntu:~/CVE-2017-6074$ gcc -o pwn poc.c 
        jsossug@ubuntu:~/CVE-2017-6074$ ls
        poc.c  pwn  README.md  trigger.c
        jsossug@ubuntu:~/CVE-2017-6074$ 

* * *

## PoC with no SELinux(SELinux Permissive)

  1.  Confirm SELinux is Permissive mode;

        jsossug@ubuntu:~/CVE-2017-6074$ getenforce
        Permissive

  2.  Run PoC bindary;

        jsossug@ubuntu:~/CVE-2017-6074$ ./pwn
        [.] namespace sandbox setup successfully
        [.] disabling SMEP & SMAP
        [.] scheduling 0xffffffff81064550(0x406e0)
        [.] waiting for the timer to execute
        [.] done
        [.] SMEP & SMAP should be off now
        [.] getting root
        [.] executing 0x402043
        [.] done
        [.] should be root now
        [.] checking if we got root
        [+] got r00t ^_^
        [!] don't kill the exploit binary, the kernel will crash
        root@ubuntu:/home/jsossug/CVE-2017-6074# ls /root/.bashrc
        /root/.bashrc
        root@ubuntu:/home/jsossug/CVE-2017-6074# id
        uid=0(root) gid=0(root) groups=0(root) context=system_u:system_r:kernel_t:s0
        root@ubuntu:/home/jsossug/CVE-2017-6074# cat /etc/shadow
        root:XXXXXXXXXXXX:13219:0:99999:7:::
        daemon:*:12041:0:99999:7:::
        bin:*:12041:0:99999:7:::
        --snip--

* * *
## PoC with SELinux Enabled(SELinux Enforcing)

  1. Reboot and set SELinux as Enforcing.

    jsossug@ubuntu:~/CVE-2017-6074$ getenforce
    Enforcing

  2. Run PoC code again;

    jsossug@ubuntu:~/CVE-2017-6074$ ./pwn 
    [.] namespace sandbox setup successfully
    [.] disabling SMEP & SMAP
    [.] scheduling 0xffffffff81064550(0x406e0)
    socket(SOCK_DCCP): Permission denied

  3. Check AV log. We can find kernel\_t(pwn command domain) couldn't create dccp\_secket Object Class.;

    Mar  5 19:39:54 ubuntu kernel: [   67.029899] audit: type=1400 audit(1488710394.317:38): avc:  denied  { create } for  pid=4372 comm="pwn" scontext=system_u:system_r:kernel_t:s0 tcontext=system_u:system_r:kernel_t:s0 tclass=dccp_socket permissive=0

* * *
## Conclusion

Now we understand that SELinux can mitigate CVE-2017-6074 through PoC.

So we can say it's better to enable SELinux for keeping your system
secure.
