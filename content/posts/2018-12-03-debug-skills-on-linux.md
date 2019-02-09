---
title: "Debug Skills on Linux"
date: 2018-12-03T22:47:51+08:00
tags:
    - linux
categories:
    - tech
---
This post will show several commands used for debugging on linux server, all examples are tested on ubuntu 18.04,
some tools are not installed by default, you can installl by `sudo apt install xxx`.
Some commands must be used via `sudo`.

System resources can be classified in three main categories: compute, storage, and network.
Usually, when you come to a performance issue, it's always caused by exhaustion of those
resources. 

## Universal metric: System load

There're several ways to get system load, `w`, `uptime`, `top`, `cat /proc/loadavg`

`uptime` example:

    03:34:23 up 20:31,  1 user,  load average: 1.02, 0.65, 0.45

Top right corner three values named `load average` is system load.

They means: average system load during last minute/last 5 minutes/last 15 minutes periods.

If `load/(# of cpu core) > 1`  means there're tasks pending in cpu queue. Usually, you will feel **slow**.

It's different with cpu utilization. CPU utilization is a metrics shows how busy cpu is handling tasks. 

If system load is high, means tasks are pending in CPU queue, maybe a result of:
- High cpu usage.
- Poor disk performance(disk io).
- Exhaustion of ram.
- ...

## free

`free -k/-m/-g` show memory usage in KB/MB/GB

`free -h`  humanize output (automatically show in KB/MB/GB...)

eg:

                total        used        free      shared  buff/cache   available
    Mem:        3.9G         421M        3.2G      80K     291M         3.2G
    Swap:       4.0G         2.4G        1.6G
    
    
`total` is available physical ram size.

`used` is ram consumed = total - free - buffers - cache

`free` is unused memory.

`shared`  memory is potentially shared by other processes.
 eg: tmpfs, shared library(.so file will only be loaded once in memory and shared by all processes). 
 
 `buff/cache` kernel buffer and disk cache, they will be reallocated to processes when needed.
 
 On some old systems, there's no `available` column, we think available physical memory = free + buff/cache.
 
 `available` means estimation of how much memory is available if you start a new program without swapping. 
 Since not all space in buffer/cache is reclaimable, it's more accuracy than `free + buff/cache`

## top

There're some light weight tools to debug overall system status, eg: vmstat, mpstat, pidstat, sar ...

If doing manually debug, `top` is enough for most case (If you're on a super busy server, be careful to run top, it's heavy).

        top - 04:23:27 up 21:20,  1 user,  load average: 0.22, 0.44, 0.47
        Tasks: 156 total,   1 running, 111 sleeping,   0 stopped,   0 zombie
        %Cpu(s):  6.6 us,  4.2 sy,  0.0 ni, 88.6 id,  0.0 wa,  0.0 hi,  0.5 si,  0.0 st
        KiB Mem :  4039548 total,   192936 free,  3128636 used,   717976 buff/cache
        KiB Swap:  3014116 total,  2407360 free,   606756 used.   290856 avail Mem 

      PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM  TIME+ COMMAND                                           
         1672 dd-agent  20   0 1236988  51044   9864 S  11.3  1.3  12:31.44 /opt/datadog-agent/bin/agent/agent run -p /opt/d+ 
        11698 root      20   0  948668 143180  28208 S   1.3  3.5  11:06.58 /usr/local/bin/uwsgi --ini /opt/uwsgi/app1.ini 
         3883 root      20   0 1154656 190700  46340 S   1.0  4.7   0:11.11 /usr/local/bin/uwsgi --ini /opt/uwsgi/app2.ini

Explains:

- `zombie` means zombie processes, it's usually caused by parent process exited before waiting child processes `exit`, mainly caused by bug.    
- `KiB mem` is same as output of command `free`.
- `VIRT` means virtual memory: how much memory the process is able to **access**, ps: not means using. Including: code size, physical ram consumed, swap usage, files on disk `mmap` to process ...
- `RES` means physical ram consumed by this process.
- `SHR` means shared memory, see section in `free` command.
  

Several useful commands after launching top interface:

- press `1`, show all cpu activities on all cores.
- press `c`, show full command name.
- press `M`,  sort by memory usage.
- press `P`, sort by cpu usage.

## Storage

### disk space usage

`lsblk` show all mounted disks and size

    NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0    8G  0 disk 
    └─xvda1 202:1    0    8G  0 part /
    xvdb    202:16   0  200G  0 disk /mnt
    
`df -h` show all disks usage:

    Filesystem      Size  Used Avail Use% Mounted on
    udev            3.9G     0  3.9G   0% /dev
    tmpfs           799M   81M  718M  11% /run
    /dev/xvda1      7.8G  5.8G  1.6G  80% /
    tmpfs           3.9G     0  3.9G   0% /dev/shm
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/xvdb       197G   64G  125G  34% /mnt
    tmpfs           799M     0  799M   0% /run/user/1003
    tmpfs           799M     0  799M   0% /run/user/1004

`du -hd1`  show all directories size in current dir, `-d1` means depth 1:

    4.5M	./mobile
    116M	./.git
    16K	./script
    8.0K	./node_modules
    1.4G	./web
    11M	./server
    4.0K	./docs
    1.5G	.
    
Sometimes we still see free space left, but system reports disk full.
Possible reason is inode is exhausted by many small files.

We can check with `df -hi`, to show inode usage per disk:

    Filesystem     Inodes IUsed IFree IUse% Mounted on
    udev             997K   393  996K    1% /dev
    tmpfs            998K   568  998K    1% /run
    /dev/xvda1       512K  248K  265K   49% /
    tmpfs            998K     1  998K    1% /dev/shm
    tmpfs            998K     3  998K    1% /run/lock
    tmpfs            998K    16  998K    1% /sys/fs/cgroup
    /dev/xvdb         13M  1.3M   12M   10% /mnt
    tmpfs            998K     5  998K    1% /run/user/1003
    tmpfs            998K     5  998K    1% /run/user/1004
    
with `du -hd1 --inode` show inode usage in current dir:

    450	./abc
    1.2K	./eesx
    1.4K	./xze
    1.3K	./dsfdfs
    3.8K	./dsfsf
    237	./334
    166	./3sfdf
    4.0K	./3ssfsf
    1.3K	./afsdfs
    1.1K	./eee
    1.7K	./sdfsfs
    17K	.

With `ncdu` we can calculate dir size very fast

### disk performance

If you want to test disk performance, you can use tools like `dd` (simple sequential-write test),
`fio` (much powerful, support multi thread and random write).

`iostat -y 2`, report io usage every 2 second, by default iostat's first line is stats since system boot,
 it's not  useful, `-y` can discard first report.

    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
              23.62    0.00    0.00    1.51    0.00   74.87

    Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
    xvda             48.00      2300.00       268.00       2300        268

`iowait` means percentage of time cpu was idle waiting for io.

`tps` means number of I/O requests per second.

`iotop -o` a top like program show io usage per process, `-o` means only show processes have io activity.

    Total DISK READ :       0.00 B/s | Total DISK WRITE :     157.12 K/s                                             [0/0]
    Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s
      TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND                                                
      387 be/3 root        0.00 B/s  141.79 K/s  0.00 %  0.00 % systemd-journald
      841 be/4 syslog      0.00 B/s    7.66 K/s  0.00 %  0.00 % rsyslogd -n [rs:main Q:Reg]
     1053 be/4 www-data    0.00 B/s    3.83 K/s  0.00 %  0.00 % nginx: worker process
     1695 be/4 dd-agent    0.00 B/s    3.83 K/s  0.00 %  0.00 % agent run -p /opt/datadog-agent/run/agent.pid

## Network

Who is using port: `lsof -i:80`:

    COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
    nginx   1052     root   39u  IPv4  20514      0t0  TCP *:80 (LISTEN)
    nginx   1053 www-data   39u  IPv4  20514      0t0  TCP *:80 (LISTEN)
    nginx   1054 www-data   39u  IPv4  20514      0t0  TCP *:80 (LISTEN)
    
Show all ports listening by pid: `lsof -i -a -p 1052`:

    COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
    nginx   1052 root   36u  IPv4  20511      0t0  TCP *:2345 (LISTEN)
    nginx   1052 root   37u  IPv4  20512      0t0  TCP *:1245 (LISTEN)
    nginx   1052 root   38u  IPv4  20513      0t0  TCP *:3244 (LISTEN)
    nginx   1052 root   39u  IPv4  20514      0t0  TCP *:8734 (LISTEN)
    nginx   1052 root   40u  IPv4  20515      0t0  TCP *:1334 (LISTEN)
    
traffic sniff with tcpdump:

    sudo tcpdump -i any dst port 5010 -A  # sniff all traffic sent to port 5010, print in ascii, -i any means listen all network interfaces.
    sudo tcpdump -i any src port 5010 -A  # sniff all traffic sent from port 5010, print in ascii
    sudo tcpdump -i any dst host 10.0.0.1 -A # snifff all traffic sent to ip 10.0.0.1, print in ascii

Show all network interface:`ip a`

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 02:79:13:0a:8c:39 brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
           valid_lft 85354sec preferred_lft 85354sec
        inet6 fe80::79:13ff:fe0a:8c39/64 scope link 
           valid_lft forever preferred_lft forever
    3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 08:00:27:8b:5a:14 brd ff:ff:ff:ff:ff:ff
        inet 192.168.33.10/24 brd 192.168.33.255 scope global enp0s8
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe8b:5a14/64 scope link 
           valid_lft forever preferred_lft forever


Show route table: `ip r`

    default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100 
    10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 
    10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100 
    192.168.33.0/24 dev enp0s8 proto kernel scope link src 192.168.33.10 


Traffic summarize with `iftop -n`, `-n` means not lookup hostname(by default it will find hostname based on ip with dns `PTR` record.)

                           191Mb                   381Mb                  572Mb                   763Mb              [0/0]
    +----------------------+-----------------------+----------------------+-----------------------+-----------------------
    10.0.44.20                                    => 10.0.46.192                                   1.46Mb  1.46Mb  1.46Mb
                                                  <=                                               6.19Mb  6.19Mb  6.19Mb
    10.0.44.20                                    => 10.0.15.245                                    872Kb   872Kb   872Kb
                                                  <=                                                332Kb   332Kb   332Kb
    10.0.44.20                                    => 10.0.1.150                                     810Kb   810Kb   810Kb
    
On right side, three number means  traffic rate in past `2s, 10s, 40s`. `=> , <=` is traffic direction.

With `nethogs` you can see network traffic per process.

Misc:

- Show socket state with `ss` or `netstat`
- Show dns record: `dig baid.com`
- Test whether a tcp port is open: `telnet 10.0.0.3 22`
- everyone knows `ping`, it sends icmp packet to test whether host is available. with `hping3`, you can send tcp/udp/icmp/raw-ip packet.

## Debug process

find process pid via name: `ps aux | grep nginx`

find process's parent process: `pstree -aps 3460`

    systemd,1
      └─supervisord,29264 /usr/bin/supervisord -n -c /etc/supervisor/supervisord.conf
          └─uwsgi,3460 --ini /opt/uwsgi/app.ini
              └─uwsgi,3464 --ini /opt/uwsgi/app.ini
                  ├─{uwsgi},3465
                  └─{uwsgi},3469
    

show process summary with  top: `top -p 1054`

kill process:

- `sudo kill 1233`, send `SIGTERM` to target process, but `SIGTERM` can by caught in program(mean to do some cleanup during shutdown), sometimes it doesn't work.
- `sudo kill -9 1233`, send `SIGKILL` to force kill target process, `SIGKILL` can't be caught by program.
- sometimes even `kill -9` can't kill process, check process status by `ps aux`, maybe stuck in `D` status, means it's waiting for io finish.
D status can't by interrupted by signal. Only thing you can do is wait io finish or reboot. 

If you feel a process seems no response, but not dead, you can use `sudo strace -p 1223` to see what it's doing, an example from idle nginx master process:

    strace: Process 1052 attached
    rt_sigsuspend([], 8
    
It's calling `rt_sigsuspend`, from `main rt_sigsuspend`, you know it's waiting for signal.

More advanced and complex debug tools including: perf, systemtap, dtrace...
