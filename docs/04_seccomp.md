
# Adding seccomp system call filtering

Docker offers the ability to start a container with a set of seccomp filters. This way,
the container's process is not able to execute arbitrary system calls but only those
that have been allowed.

For the linux kernel there are many system calls, ranging from "read from/write to file descriptor"
and "bind to socket", "create a child process" up to "turn swap off" and "reboot the host".
Some are architecture-specific. There is a nice [table list](http://syscalls.kernelgrok.com/) with links
to man pages.

This can become important for containers, because we do not want to allow all syscalls for
all containers.

## add seccomp filtering ..

First, seccomp is currently only available to docker if the lxc driver is used (instead of libcontainer):

```bash
[root@localhost ~]# docker info | grep "^Execution"
Execution Driver: native-0.2
```

We're going to change that:

```bash
# yum install lxc
(...)

# vi /etc/sysconfig/docker
OPTIONS='--selinux-enabled -e lxc'

# systemctl restart docker.service

# docker info | grep "^Execution"
Execution Driver: lxc-1.0.7
```

One thing we can observe is that containers are confined in a different domain, and without categories:

```bash
# docker run -ti --rm fedora:21 /bin/bash
bash-4.3# ps -Z
LABEL                             PID TTY          TIME CMD
system_u:system_r:spc_t:s0          1 ?        00:00:00 bash
system_u:system_r:spc_t:s0         11 ?        00:00:00 ps
```

So it's `spc_t` and not `svirt_lcx_net_t`. Not sure why this is the case, but it's [well-defined](https://github.com/fedora-cloud/docker-selinux/blob/master/docker.te) and
working.

How do we get that "seccomp"-thing into our containers? The contrib section of docker's github repository has
the answer:

```bash
# git clone https://github.com/docker/docker
# ls -al docker/contrib/*seccomp*
-rwxr-xr-x. 1 root root 2199 17. Apr 08:02 docker/contrib/mkseccomp.pl
-rw-r--r--. 1 root root 7369 17. Apr 08:02 docker/contrib/mkseccomp.sample
```

`mkseccomp.sample` is a file with all system calls listed by their names, and fortunately grouped into
categories such as Filesystem, Networking etc. Plus, the last part is the "Admin System calls", and they're
commented out. So this file could be used out of the box to improve security.

The docker daemon / seccomp does not interpret this file directly, because system call names may be different
across architectures. So it needs to be compiled into numeric form, by `mkseccomp.pl`. This perl script contains
a very nice documentation of what to do it, keeping it quite simple. It needs cpp as a sort of preprocessor to
translate syscall names into numbers:

```bash
# yum install cpp glibc-headers

# cd docker/contrib
# ./mkseccomp.pl <mkseccomp.sample >/root/seccomp-sample
# ls -al /root/seccomp-sample
-rw-r--r--. 1 root root 938 17. Apr 08:15 /root/seccomp-sample
```

This file contains only system call numbers for our architecture, plus a "whitelist" header generated by
the perl script.

This file is activated with an additional `--lcx-conf` setting:

```
# docker run -tdi --lxc-conf="lxc.seccomp=/root/seccomp-sample" fedora:21 /bin/bash
09f7e3b24ff4bafc2bbe2e6b05a9ed7181e3637810570c0bea4cac5976ad6c23

# docker inspect 09f
(...)
"HostConfig": {
(...)
    "LxcConf": [
        {
            "Key": "lxc.seccomp",
            "Value": "/root/seccomp-sample"
        }
    ],
```

Let's do a test and try to chroot into a new shell:

```bash
# docker run -ti --lxc-conf="lxc.seccomp=/root/seccomp-sample" fedora:21 /bin/bash
bash-4.3# chroot / /bin/bash
Bad system call
```

So we're not allowed to chroot any more, this system call is rejected. Even if we run `--privileged`.

Another example for changing file modes. Let's copy `mkseccomp.sample` into `no-chmod-seccomp` and comment out
chmod, fchmod and fchmodat. Compile and test:

```bash
# grep chmod no-chmod.sample
//chmod
//fchmod
//fchmodat

# ./mkseccomp.pl <no-chmod.sample >/root/no-chmod-seccomp

# docker run -ti --lxc-conf="lxc.seccomp=/root/no-chmod-seccomp" fedora:21 /bin/bash
bash-4.3# touch x
bash-4.3# ls -al x
-rw-r--r--. 1 root root 0 Apr 17 04:52 x
bash-4.3# chmod a+rwx x
Bad system call
```

Similar things exist in the space of linux capabilities. But capabilities are checked inside the kernel,
seccomp protects the kernel from ever receiving such system calls.


# Detecting the required system call surface using `strace`

From the list of system calls it's easy to remove single calls, but it's not easy to tell what calls
are actually needed by an application. Fortunately, there's support for this. Long-known `strace` traces
all system calls a process makes, and is also able to produce a nice summary. The `mkseccomp.pl` file contains
a documentation for this, so let's try it out.

We'll pull a docker image for an owncloud installation, [l3iggs/owncloud](https://registry.hub.docker.com/u/l3iggs/owncloud/)

```bash
# docker pull l3iggs/owncloud
```

Then we combine a number of steps. We start the owncloud container, detached in background. After querying the container PID,
we let strace collect all systems calls and write the summary to a file, pushing this to background as well.
Ideally, an automated testing tool simulates a walk-through over the whole application, with a very high line coverage, so
that all system calls are at least called once. Since i don't have that for owncloud i just hit the default page, but you'll get the idea.

```bash
# docker run --detach -p 80:80 -p 443:443 \
  --name oc l3iggs/owncloud ;
  PID=`docker inspect -f '{{ .State.Pid }}' oc` ;
  strace -f -q -c -o trace.out -p $PID &

a4de97dab43c98ea5cf419d84340183216ac1eb94c43766ea17f1307656bcab0
[1] 18066
```

Container started, strace process pushed to background, with
* `-f`: trace child processes as well
* `-q`: be quiet
* `-c`: Count times and calls
* `-o`: Write to output file
* `-p`: Attach to process by PID

Then, make a sample https call. This is just for demoing the process of course, nothing fully functional.

```bash
# curl --insecure https://127.0.0.1/owncloud/
```

When finished working with it, we stop the container. strace stops as well since the process is gone.
We have our output trace file which contains a nicely formatted table.

```bash
# docker stop oc
oc
[1]+  Fertig                  strace -f -q -c -o trace.out -p $PID

# ls -al trace.out
-rw-r--r--. 1 root root 6655 17. Apr 09:55 trace.out

# cat trace.out
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 49.93    0.350418        1718       204           io_getevents
 21.75    0.152618         170       897        28 futex
 12.19    0.085562        1097        78           select
  5.02    0.035245           0     73015     11057 lstat
  4.24    0.029762         140       213        85 wait4
  1.37    0.009604           2      6029        10 read
  1.23    0.008606           2      5495       126 open
  0.86    0.006042           7       925       282 stat
  0.51    0.003600           3      1394           fstat
  0.38    0.002691           0      5980        66 close
  0.37    0.002598           2      1660           mmap
  0.36    0.002526          15       171           pread
  0.33    0.002312           4       524           munmap
  0.19    0.001364           1      1579           rt_sigaction
  0.15    0.001069           1      1002           rt_sigprocmask
  0.14    0.000991           1       907           mprotect
(...)
```

The last column can be extracted into a file, served to mkseccomp.pl and we're ready. This however is a bit simplified because we start strace some time after the container has started, which might miss out syscalls that are not called again afterwards and thus don't appear in the trace file.