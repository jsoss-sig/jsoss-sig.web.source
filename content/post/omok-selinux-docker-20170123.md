+++
title = "Docker vulnerability (CVE-2016-9962) PoC with SELinux (Again)"
date = "2017-01-23T13:07:34+09:00"
draft = false
+++

This blog is for following up to reproduce CVE-2016-9962(vulnerability for
Docker) and how can we mitigate it by using SELinux.

(Written by Kazuki Omo:ka-omo@sios.com).

  

* * *

## Reference

#### [Docker vulnerability (CVE-2016-9962) PoC with SELinux](https://jsoss-sig.github.io/post/omok-selinux-docker-20170118/)

#### <https://bugzilla.redhat.com/show_bug.cgi?id=1409531>

#### <https://bugzilla.suse.com/show_bug.cgi?id=1012568#c2>

## Mistake in Previous PoC

I sent previous PoC result to SELinux , I got result I did mistake in Previous
PoC. **Actually, I didn't use SELinux Access Control in Previous PoC**

In Previous PoC, "[PoC] run container(sh) in shell1;"

    
    
    [root@localhost ~]# runc run ctr
    
    / #
    

But Finally I found the "runc" program is working in "unconfined_t" domain.
From another terminal, I checked runc domain;

    
    
    [root@fedora25 ~]# ps axZ|grep runc
    unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 1578 pts/0 Sl+
    0:00 runc run ctr
    

So this means runc is working in unconfined_t domain, then that runc is having
lots of permissions(actually un-confined) from SELinux.

This is a reason why SELinux couldn't mitigate my previous PoC.

## Assign "container_t" domain on runc.

When I checked Fedora25 SELinux Policy, I found that container_t domain is
switched from container_runtime_t(which is domain for docker process, etc.).
And container_runtime_t is transited from initrc domain by exec
container_runtime_exec_t file(/usr/bin/runc), such as;

    
    
    [root@fedora25 ~]# ls -lZ /usr/bin/runc
    -rwxr-xr-x. 1 root root system_u:object_r:container_runtime_exec_t:s0 5016704 Jan 20 19:26 /usr/bin/runc
    
    
    
    ./container.cil:(typetransition initrc_domain container_runtime_exec_t process container_runtime_t)
    

Then, for doing PoC more close to existance situation, we need to run "runc"
as "container_t" domain.

For running "runc" as "container_t" domain, we need to add several policy
(typetransition rule and more allow rule) to transit from unconfined_t to
container_t domain. Also I changed PoC directory from /root to /tmp.

## Changed PoC directory.

For more easy to write Policy for this PoC, I changed PoC directory to /tmp
/PoC-CVE-2016-9962;

    
    
    [root@fedora25 PoC-CVE-2016-9962]# pwd
    /tmp/PoC-CVE-2016-9962
    [root@fedora25 PoC-CVE-2016-9962]# ls -l
    total 4
    -rw-r--r--.  1 root root 2364 Jan 20 09:04 config.json
    drwxr-xr-x. 18 root root  380 Jan 23 12:33 rootfs
    

## Additional Policy

Just for PoC, I made below policy rule file as "/root/custom_policy/runc.cil";

    
    
    (typetransition unconfined_usertype container_runtime_exec_t process container_t
    )
    (roletransition unconfined_r container_runtime_exec_t process system_r)
    
    (allow container_t user_tmp_t (file (open read execute execute_no_trans)))
    (allow container_t var_run_t (dir (write add_name create setattr remove_name rmd
    ir)))
    (allow container_t var_run_t (fifo_file (create setattr unlink read open)))
    (allow container_t ptmx_t (chr_file (read write open ioctl)))
    (allow container_t devpts_t (chr_file (setattr read write open ioctl getattr)))
    (allow container_t root_t (dir (mounton)))
    (allow container_t user_tmp_t (dir (mounton write add_name create remove_name rm
    dir)))
    (allow container_t user_tmp_t (lnk_file (read)))
    (allow container_t proc_t (filesystem (mount remount)))
    (allow container_t tmpfs_t (filesystem (mount remount)))
    (allow container_t tmpfs_t (dir (setattr write add_name create mounton)))
    (allow container_t devpts_t (filesystem (mount)))
    (allow container_t sysfs_t (filesystem (mount)))
    (allow container_t cgroup_t (filesystem (remount)))
    (allow container_t tmpfs_t (lnk_file (create)))
    (allow container_t tmpfs_t (chr_file (create setattr read write open getattr ioc
    tl append)))
    (allow container_t tmpfs_t (file (open create mounton)))
    (allow container_t proc_t (dir (mounton)))
    (allow container_t proc_t (file (mounton)))
    (allow container_t sysctl_irq_t (dir (mounton)))
    (allow container_t sysctl_t (dir (mounton)))
    (allow container_t sysctl_t (file (mounton)))
    (allow container_t proc_kcore_t (file (mounton)))
    (allow container_t nsfs_t (file (getattr read open)))
    (allow container_t var_run_t (file (create read write open unlink)))
    (allow container_t sysfs_t (dir (mounton)))
    (allow container_t kernel_t (unix_stream_socket (read write)))
    (allow init_t kernel_t (unix_stream_socket (read write)))
    (allow container_t init_t (unix_stream_socket (read write)))
    [root@fedora25 custom_policy]# cat runc.cil
    (typetransition unconfined_usertype container_runtime_exec_t process container_t)
    (roletransition unconfined_r container_runtime_exec_t process system_r)
    
    (allow container_t user_tmp_t (file (open read execute execute_no_trans)))
    (allow container_t var_run_t (dir (write add_name create setattr remove_name rmdir)))
    (allow container_t var_run_t (fifo_file (create setattr unlink read open)))
    (allow container_t ptmx_t (chr_file (read write open ioctl)))
    (allow container_t devpts_t (chr_file (setattr read write open ioctl getattr)))
    (allow container_t root_t (dir (mounton)))
    (allow container_t user_tmp_t (dir (mounton write add_name create remove_name rmdir)))
    (allow container_t user_tmp_t (lnk_file (read)))
    (allow container_t proc_t (filesystem (mount remount)))
    (allow container_t tmpfs_t (filesystem (mount remount)))
    (allow container_t tmpfs_t (dir (setattr write add_name create mounton)))
    (allow container_t devpts_t (filesystem (mount)))
    (allow container_t sysfs_t (filesystem (mount)))
    (allow container_t cgroup_t (filesystem (remount)))
    (allow container_t tmpfs_t (lnk_file (create)))
    (allow container_t tmpfs_t (chr_file (create setattr read write open getattr ioctl append)))
    (allow container_t tmpfs_t (file (open create mounton)))
    (allow container_t proc_t (dir (mounton)))
    (allow container_t proc_t (file (mounton)))
    (allow container_t sysctl_irq_t (dir (mounton)))
    (allow container_t sysctl_t (dir (mounton)))
    (allow container_t sysctl_t (file (mounton)))
    (allow container_t proc_kcore_t (file (mounton)))
    (allow container_t nsfs_t (file (getattr read open)))
    (allow container_t var_run_t (file (create read write open unlink)))
    (allow container_t sysfs_t (dir (mounton)))
    (allow container_t kernel_t (unix_stream_socket (read write)))
    (allow init_t kernel_t (unix_stream_socket (read write)))
    (allow container_t init_t (unix_stream_socket (read write)))
    

## Load Additional Policy to PoC system

On the PoC system

    
    
    [root@fedora25 ~]# semodule -i /root/custom_policy/runc.cil
    

## Check custom policy is working or not.

After load runc.cil, run "runc run ctr" in a terminal;

    
    
    [root@fedora25 PoC-CVE-2016-9962]# runc run ctr
    / #
    

Then open another terminal and check this "runc" program is working as
"container_t" domain;

    
    
    [root@fedora25 ~]# ps axZ|grep runc
    unconfined_u:system_r:container_t:s0-s0:c0.c1023 6799 pts/1 Sl+   0:00 runc run ctr
    

If the "runc" is not working on container_t domain, chack
/var/log/audit/audit.log and maybe add several another rules to runc.cil.

## PoC(Again!)

  * open 2 terminals(shell1, shell2).

  * Check SELinux is enabled;
    
        [root@localhost ~]# getenforce
        Enforcing
    

  * run container(sh) in shell1;
    
        [root@localhost ~]# runc run ctr
        / # 
    

  * run new container in shell2 with "runc exec" command. It is pausing 500sec;
    
        [root@localhost ~]# runc run ctr
    
    

  * run "ps ax" in shell1. You can see shell2 process;
    
        [root@localhost ~]# runc run ctr
        / # ps ax
        PID   USER     TIME   COMMAND
            1 root       0:00 sh
            6 root       0:00 /proc/self/exe init
           11 root       0:00 ps ax
        / # 
    

  * In above case, check /proc/6/fd by ls command. You see path as fd/4;
    
        / # ls -la /proc/6/fd/
        total 0
        dr-x------    2 root     root             0 Jan 16 06:43 .
        dr-xr-xr-x    9 root     root             0 Jan 16 06:43 ..
        lrwx------    1 root     root            64 Jan 16 06:43 0 -> /dev/pts/4
        lrwx------    1 root     root            64 Jan 16 06:43 1 -> /dev/pts/4
        lrwx------    1 root     root            64 Jan 16 06:43 2 -> /dev/pts/4
        lrwx------    1 root     root            64 Jan 16 06:43 3 -> socket:[40487]
        lr-x------    1 root     root            64 Jan 16 06:43 4 -> /run/runc/ctr
        lrwx------    1 root     root            64 Jan 16 06:43 5 -> /dev/pts/4
        lr-x------    1 root     root            64 Jan 16 06:43 6 -> pipe:[40496]
        l-wx------    1 root     root            64 Jan 16 06:43 7 -> /dev/null
        / # 
    

  * Do ls /etc/shadow file by using "/proc/6/fd/4/../../../etc/shadow" path;
    
        / # ls -l /proc/6/fd/4/../../../etc/shadow
        ls: /proc/6/fd/4/../../../etc/shadow: Permission denied
    

Check /var/log/audit/audit.log;

    
        type=AVC msg=audit(1485143271.659:3107): avc:  denied  { getattr } for  pid=8847 comm="ls" path="/etc/shadow" dev="dm-0" ino=785423 scontext=unconfined_u:system_r:container_t:s0-s0:c0.c1023 tcontext=system_u:object_r:shadow_t:s0 tclass=file permissive=0
    

Fine, Now SELinux is protecting to search /etc/shadow file.

  * Also you can't read /etc/shadow file because the file permission is "000".
    
        / # cat /proc/6/fd/4/../../../etc/shadow
        cat: can't open '/proc/6/fd/4/../../../etc/shadow': Permission denied
    

If I change /etc/shadow file as "755" permission, the DAC control will permit
to read the file.

    
        [root@fedora25 ~]# chmod 755 /etc/shadow
        [root@fedora25 ~]# ls -lh /etc/shadow
        -rwxr-xr-x. 1 root root 1.3K Jan 20 08:30 /etc/shadow
    

But still I can't read /etc/shadow file ;

    
        / # cat /proc/6/fd/4/../../../etc/shadow
        cat: can't open '/proc/6/fd/4/../../../etc/shadow': Permission denied
    

I could see in the /var/log/audit/audit.log file that the"read" action is
denied by SELinux;

    
        type=AVC msg=audit(1485143502.183:3311): avc:  denied  { read } for  pid=9418 comm="cat" name="shadow" dev="dm-0" ino=785423 scontext=unconfined_u:system_r:container_t:s0-s0:c0.c1023 tcontext=system_u:object_r:shadow_t:s0 tclass=file permissive=0
    

* * *

## Conclusion

Now we know from PoC that we could mitigate CVE-2016-9962 by enabling SELinux.

So we should enable SELinux in container environment also and making more
safety(and keeping to have mitigate way even if in 0-day situation)
environment.

