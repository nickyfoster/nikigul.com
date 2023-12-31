---
author:
  name: "Nikita Gulyayev"
date: 2022-01-18
title: 'Handy Linux Commands'
series:
- Tutorials
---

## Search for files and words
```
$ find /var/www -name '*.css'
$ grep font /var/www/style.css
$ grep -R font /var/www/
```

## List files
```
$ ls -Rlhat 
  args: -s list file sizes
```

## Detect distribution
```
$ lsb_release -a        # get debian-based discribution
$ cat /etc/os-release   # get systemd-based discribution
$ uname -a              # info about current system
```

## Get a list of service
```
$ service --status-all
$ journalctl -p error -S yesterday
  args: -b since last boot
```

## File compression
```
$ tar -cvf archive.tar ./dir    # archive
$ tar -czf archive.tar ./dir    # archive gz
$ tar -xvf archive.tar          # unarchive
$ tar -xzvf archive.tar         # unarchive gz
$ tar -tf arvhice.tar           # list 
```

## Investigate root directory for disk usage
```
$ du -sch /.[!.]* /*
```

## CPU information
```
$ lscpu
$ lshw | grep cpu
```

## Process monitoring
### List processes in a hierarchy
```
$ ps -e -o pid,args --forest
```

### List processes sorted by % cpu usage
```
$ ps -e -o pcpu,cpu,nice,state,cputime,args --sort pcpu | sed '/^ 0.0 /d'
```

### List processes sorted by mem (KB) usage
```
$ ps -e -orss=,args= | sort -b -k1,1n | pr -TW$COLUMNS
```

### List all threads for a particular process ("firefox-bin" process in example )
```
$ ps -C firefox-bin -L -o pid,tid,pcpu,state
```

### Gather info about process
```
$ lsof -p $$
```

## Write stoud to file etc
```
$ ls | tee file
$ ls > file
```
Example:

The following command will write current crontab entries to a file crontab-backup.txt 
and pass the crontabentries to sed command, which will do the substituion. 
After the substitution, it will be added as a new cron job.
```
$ crontab -l | tee crontab-backup.txt | sed 's/old/new/' | crontab –
```

## Troubleshooting
```
$ dmesg - get kernel ring buffer
$ dmesg --level=err,warn -T   # -T flaf for timestamp
$ dmesg | grep -i eth0      # display info about specific device
$ dmesg -u            # display only userspace messages
```
## IP and Network
```
$ ip addr                       # get network configuration
$ ip link show eth0             # show info about specific device
$ ifdown <interface name>       # stop interface
$ ifup <interface name>         # start interface
$ cat /etc/network/interfaces   # check network config file
$ mtr 8.8.8.8                   # check basic connectivity for Ubutnu, or user traceroute
$ ss -a                         # get all ports
$ ss -l                         # get listening sockets
$ ss -t                         # get all TCP connections
$ ss -lt                        # get all TCP sockets
$ ss -ua                        # get all UDP connections
$ ss -lu                        # get all UDP sockets
$ ss -p                         # dispplay PID of socket
$ ss -s                         # list summary statistics
$ iftop -f icmp                 # get all ICMP messages containing error etc
```

## TCPDUMP
```
$ tcpdump -nnSX port 443        # get all packets on port 
  args: -nn   # Don’t resolve hostnames or port names 
        -S    # Get the entire packet
        -X    # Get hex output
$ tcpdump port 3389
$ tcpdump -nvX src net 192.168.0.0/16 and dst net 10.0.0.0/8 or 172.16.0.0/16 # from one network to another
$ tcpdump -nnvvS src 10.5.2.3 and dst port 3389 # from specifit IP to specific port
$ tcpdump -vvAls0 | grep 'User-Agent:'          # find user-agents
```
## Memory
```
$ free -m   # get free mem in Mb
```

## Processes
```
$ ps aux -H                       # processes by hierarchy
$ ps aux --sort=-%cpu | head      # get top cpu consumers  
$ ps aux --sort=-%mem | head      # get top mem consumers
$ ps -A | grep -i stress          # get unresponsive processes
$ watch -n 1 'ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head'   # real time process monitoring
$ ps -C tmux                      # get processes by executable
$ ps aux                          # list all processes in BSD format
$ ps -fu 1000                     # list process for user 1000
$ ps -fp 1178                     # list processes by PID
$ ps -eo comm,etime,user | grep httpd       # get execution time of process
```

## System info
```
$ iostat -m 5  # get CPU and devices statistics in Mb every 5 seconds
$ vmstat
```