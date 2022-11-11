--------------------------------------------------------------
Document on various HPC Networking Challenges, troubleshooting procedures and Solutions.

Made by Cyndie,

JUnior System Admin @ *icipe*, Kenya

 2022.
---------------------------------------------------------------
# My HPC Cluster:

## Head Node:

**Hostname:** admin.hpc.com

**IP Address:** 10.2.13.21

## Node 1:

**Hostname:** server1.hpc.com

**IP Address:** 10.4.2.3

## Node 2:

**Hostname:** server2.hpc.com

**IP Address:** 10.2.13.29


# ssh_exchange_identification: Connection closed by remote host

**Date:** 11th Oct, 2022

I tried to `ssh` to all nodes unsuccessfully.

```
[root@server1 ~]# ssh admin.hpc.com
ssh_exchange_identification: Connection closed by remote host

```
When I tried to `ping` my nodes, I got 100% packet loss:

```
[root@admin ~]# ping -c 3 server1.hpc.com
PING server1.hpc.com (10.4.2.3) 56(84) bytes of data.

--- server1.hpc.com ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms

```

But pinging an external server works:

```
[root@admin ~]# ping -c3 google.com
PING google.com (172.217.170.174) 56(84) bytes of data.
64 bytes from mba01s09-in-f14.1e100.net (172.217.170.174): icmp_seq=1 ttl=55 time=7.07 ms
64 bytes from mba01s09-in-f14.1e100.net (172.217.170.174): icmp_seq=2 ttl=55 time=6.81 ms
64 bytes from mba01s09-in-f14.1e100.net (172.217.170.174): icmp_seq=3 ttl=55 time=6.92 ms

--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 6.813/6.935/7.073/0.143 ms

```

My `/etc/hosts` file:

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
#Head Node IP address and local domain
10.2.13.21 admin.hpc.com hpcadmin
#Node1
10.4.2.3 server1.hpc.com server1
#Node2
10.2.13.29 server2.hpc.com cynhpc

```

## Troubleshooting:

### 1. Checked the `firewalld` Configuration Settings:

In head node: `ssh` was enabled, as seen in the `services` section:

```
[root@admin ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno1
  sources: 
  services: dhcpv6-client dns freeipa-ldap freeipa-ldaps high-availability mountd nfs rpc-bind ssh
  ports: 6817/udp 6817/tcp 6818/tcp 7321/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
	
```

In Node1, `ssh` was not enabled:

```
[root@server1 ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno1
  sources: 
  services: dhcpv6-client high-availability 
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[root@server1 ~]#	
```

In NOde2 `ssh` was enabled:

```
[root@server2 ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno1
  sources: 
  services: dhcpv6-client high-availability ssh 
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[root@server2 ~]#	
```

I enabled `port 22` through the firewall for Node1:

```
[root@server1 ~]# firewall-cmd --permanent --add-port=22/tcp
success
[root@server1 ~]# firewall-cmd --reload
success
[root@server1 ~]#
```

Re-Checked the firewall settings:

```
[root@server1 ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno1
  sources: 
  services: dhcpv6-client high-availability ssh 
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
[root@server1 ~]#	
```
Also checked the `iptables` config file for allowed ports:

```
[root@server1 ~]# iptables-save | grep dport\22
-A IN_public_allow -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW,UNTRACKED -j ACCEPT
-A IN_public_allow -p tcp -m tcp --dport 2224 -m conntrack --ctstate NEW,UNTRACKED -j ACCEPT
-A IN_public_allow -p tcp -m tcp --dport 22 -m conntrack --ctstate NEW,UNTRACKED -j ACCEPT
[root@server1 ~]#
```

### 2. I checked if the `sshd` daemon was running, in all nodes:

Head Node:

```
[root@admin ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2022-10-08 14:11:48 EAT; 2 days ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 1285 (sshd)
    Tasks: 1
   CGroup: /system.slice/sshd.service
           └─1285 /usr/sbin/sshd -D

Oct 08 14:11:48 admin.hpc.com systemd[1]: Starting OpenSSH server daemon...
Oct 08 14:11:48 admin.hpc.com sshd[1285]: Server listening on 0.0.0.0 port 22.
Oct 08 14:11:48 admin.hpc.com sshd[1285]: Server listening on :: port 22.
Oct 08 14:11:48 admin.hpc.com systemd[1]: Started OpenSSH server daemon.
[root@admin ~]#

```

Node1 :

```
[root@server1 ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2022-10-11 13:04:48 EAT; 6h ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 8233 (sshd)
   CGroup: /system.slice/sshd.service
           └─8233 /usr/sbin/sshd -D

Oct 11 13:04:47 server1.hpc.com systemd[1]: Starting OpenSSH server daemon...
Oct 11 13:04:47 server1.hpc.com sshd[8233]: Server listening on 0.0.0.0 port 22.
Oct 11 13:04:47 server1.hpc.com sshd[8233]: Server listening on :: port 22.
Oct 11 13:04:48 server1.hpc.com systemd[1]: Started OpenSSH server daemon.
[root@admin ~]#

```

### 3. I then used `mtr` tool to detect network faults:

```
[root@admin ~]# yum install mtr
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirror.aptus.co.tz
 * epel: epel.mirror.liquidtelecom.com
 * extras: mirror.aptus.co.tz
 * updates: mirror.aptus.co.tz
Package 2:mtr-0.85-7.el7.x86_64 already installed and latest version
Nothing to do

```
Already installed. Great.

1. Analysis on Node1:


```

[root@admin ~]# mtr server1.hpc.com --report
Start: Tue Oct 11 13:16:24 2022
HOST: admin.hpc.com               Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- gateway                    0.0%    10    0.4   0.4   0.3   0.5   0.0
  2.|-- ???                       100.0    10    0.0   0.0   0.0   0.0   0.0

```

Node2:

```
[root@admin ~]# mtr server2.hpc.com --report
Start: Tue Oct 11 13:20:24 2022
HOST: admin.hpc.com               Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- gateway                    0.0%    10    0.4   0.4   0.4   0.5   0.0
  2.|-- ???                       100.0    10    0.0   0.0   0.0   0.0   0.0

```

The parameters in the preceding command output are described as follows:

* **HOST:** IP address or domain name of the node
* **Loss%:** packet loss rate
* **Snt:** number of packets sent per second
* **Last:** last response time
* **Avg:** average response time
* **Best:** shortest response time
* **Wrst:** longest response time
* **StDev:** standard deviation, a larger value indicates a larger difference between the response time for each data packet on the node

According to the report:
* A 100% packet loss meant Node2 was not receiving any packets.
* It therefore had issues with its Network Configuration.

### 4. I also used `traceroute` for diagnostics.

```
[root@admin ~]# traceroute 10.4.2.3
traceroute to 10.4.2.3 (10.4.2.3), 30 hops max, 60 byte packets
 1  gateway (10.2.13.1)  0.471 ms  0.514 ms  0.638 ms
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * *
13  * * *
14  * * *
15  * * *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  * * *
28  * * *
29  * * *
30  * * *

```


### 5. I checked Network status using `systemctl`

I now checked the network status of my nodes:

Head Node:

```
[root@admin ~]# systemctl status network.service 
● network.service - LSB: Bring up/down networking
   Loaded: loaded (/etc/rc.d/init.d/network; bad; vendor preset: disabled)
   Active: active (exited) since Wed 2022-10-12 17:10:03 EAT; 11s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 3804 ExecStop=/etc/rc.d/init.d/network stop (code=exited, status=0/SUCCESS)
  Process: 3994 ExecStart=/etc/rc.d/init.d/network start (code=exited, status=0/SUCCESS)

Oct 12 17:10:02 admin.hpc.com systemd[1]: Starting LSB: Bring up/down networking...
Oct 12 17:10:02 admin.hpc.com network[3994]: Bringing up loopback interface:  [  OK  ]
Oct 12 17:10:02 admin.hpc.com network[3994]: Bringing up interface eno1:  Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/Ac...nnection/4)
Oct 12 17:10:02 admin.hpc.com network[3994]: [  OK  ]
Oct 12 17:10:03 admin.hpc.com systemd[1]: Started LSB: Bring up/down networking.
Hint: Some lines were ellipsized, use -l to show in full.
[root@admin ~]# 

```

Node 1:

```
[root@server1 ~]# systemctl status network.service 
● network.service - LSB: Bring up/down networking
   Loaded: loaded (/etc/rc.d/init.d/network; bad; vendor preset: disabled)
   Active: active (exited) since Wed 2022-10-12 17:10:03 EAT; 11s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 1256 ExecStop=/etc/rc.d/init.d/network stop (code=exited, status=0/SUCCESS)

Oct 12 17:10:02 server1.hpc.com systemd[1]: Starting LSB: Bring up/down networking...
Oct 12 17:10:02 server1.hpc.com network[1256]: Bringing up loopback interface:  [  OK  ]
Oct 12 17:10:02 server1.hpc.com network[1256]: Bringing up interface eno1:  [  OK  ]
Oct 12 17:10:03 server1.hpc.com systemd[1]: Started LSB: Bring up/down networking.
[root@admin ~]# 

```
Node2:

```
[root@server2 ~]# systemctl status network.service 
● network.service - LSB: Bring up/down networking
   Loaded: loaded (/etc/rc.d/init.d/network; bad; vendor preset: disabled)
   Active: active (exited) since Wed 2022-10-12 19:24:27 EAT; 13min ago
     Docs: man:systemd-sysv-generator(8)
  Process: 1219 ExecStop=/etc/rc.d/init.d/network stop (code=exited, status=0/SUCCESS)

Oct 12 17:10:02 server1.hpc.com systemd[1]: Starting LSB: Bring up/down networking...
Oct 12 17:10:02 server1.hpc.com network[1219]: Bringing up loopback interface:  [  OK  ]
Oct 12 17:10:02 server1.hpc.com network[1219]: Bringing up interface eno1:  [  OK  ]
Oct 12 17:10:03 server1.hpc.com systemd[1]: Started LSB: Bring up/down networking.
[root@server2 ~]# 

```

So This was why my network configuration was faulty!!!!

# Solution

I disabled **Network Manager**.

```
[root@admin ~]# systemctl stop NetworkManager
[root@admin ~]# systemctl disable NetworkManager
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
Removed symlink /etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service.

```

After disabling Network Manager I then restarted my network and checked my Network Status.

```
[root@admin ~]# systemctl restart network.service 
[root@admin ~]# systemctl status network.service 
● network.service - LSB: Bring up/down networking
   Loaded: loaded (/etc/rc.d/init.d/network; bad; vendor preset: disabled)
   Active: active (running) since Wed 2022-10-12 17:18:33 EAT; 2min 51s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 4643 ExecStop=/etc/rc.d/init.d/network stop (code=exited, status=0/SUCCESS)
  Process: 4801 ExecStart=/etc/rc.d/init.d/network start (code=exited, status=0/SUCCESS)
    Tasks: 1
   CGroup: /system.slice/network.service
           └─5015 /sbin/dhclient -1 -q -lf /var/lib/dhclient/dhclient-8db61bde-a1e6-4696-b31a-d0f5f39e8454-eno1.lease -pf /var/run/dhclient-eno1.pid -H admin eno1

Oct 12 17:18:26 admin.hpc.com network[4801]: Bringing up loopback interface:  [  OK  ]
Oct 12 17:18:26 admin.hpc.com network[4801]: Bringing up interface eno1:
Oct 12 17:18:30 admin.hpc.com dhclient[4959]: DHCPDISCOVER on eno1 to 255.255.255.255 port 67 interval 3 (xid=0x51427229)
Oct 12 17:18:30 admin.hpc.com dhclient[4959]: DHCPREQUEST on eno1 to 255.255.255.255 port 67 (xid=0x51427229)
Oct 12 17:18:30 admin.hpc.com dhclient[4959]: DHCPOFFER from 10.2.13.1
Oct 12 17:18:30 admin.hpc.com dhclient[4959]: DHCPACK from 10.2.13.1 (xid=0x51427229)
Oct 12 17:18:32 admin.hpc.com dhclient[4959]: bound to 10.2.13.21 -- renewal in 35685 seconds.
Oct 12 17:18:32 admin.hpc.com network[4801]: Determining IP information for eno1... done.
Oct 12 17:18:33 admin.hpc.com network[4801]: [  OK  ]
Oct 12 17:18:33 admin.hpc.com systemd[1]: Started LSB: Bring up/down networking.

```

My Network Service daemon was now up and running!!!

I did the same on my nodes:

Node1:

```
[root@server1 ~]# systemctl stop NetworkManager
[root@server1 ~]# systemctl disable NetworkManager
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
Removed symlink /etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service.
[root@server1 ~]# systemctl restart network.service 
[root@server1 ~]# systemctl status network.service 
● network.service - LSB: Bring up/down networking
   Loaded: loaded (/etc/rc.d/init.d/network; bad; vendor preset: disabled)
   Active: active (running) since Wed 2022-10-12 18:54:48 EAT; 3min 39s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 3414 ExecStop=/etc/rc.d/init.d/network stop (code=exited, status=0/SUCCESS)
  Process: 3575 ExecStart=/etc/rc.d/init.d/network start (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/network.service
           └─3772 /sbin/dhclient -1 -q -lf /var/lib/dhclient/dhclient-f8027990-6bda-45cb-8ea4-fa85b0a071a7-eno1.lease -pf /var/run/dhclient-eno1.pid -H server1 eno1

Oct 12 18:54:34 server1.hpc.com systemd[1]: Starting LSB: Bring up/down networking...
Oct 12 18:54:34 server1.hpc.com network[3575]: Bringing up loopback interface:  [  OK  ]
Oct 12 18:54:35 server1.hpc.com network[3575]: Bringing up interface eno1:
Oct 12 18:54:38 server1.hpc.com dhclient[3721]: DHCPREQUEST on eno1 to 255.255.255.255 port 67 (xid=0x15c8b011)
Oct 12 18:54:46 server1.hpc.com dhclient[3721]: DHCPREQUEST on eno1 to 255.255.255.255 port 67 (xid=0x15c8b011)
Oct 12 18:54:46 server1.hpc.com dhclient[3721]: DHCPACK from 10.4.0.1 (xid=0x15c8b011)
Oct 12 18:54:48 server1.hpc.com dhclient[3721]: bound to 10.4.2.3 -- renewal in 33254 seconds.
Oct 12 18:54:48 server1.hpc.com network[3575]: Determining IP information for eno1... done.
Oct 12 18:54:48 server1.hpc.com network[3575]: [  OK  ]
Oct 12 18:54:48 server1.hpc.com systemd[1]: Started LSB: Bring up/down networking.
[root@server1 ~]# 

```

### 6. Re-checking if my nodes' communication using `ping`

```
[root@admin ~]# ping -c 3 server1.hpc.com
PING server1.hpc.com (10.4.2.3) 56(84) bytes of data.
64 bytes from server1.hpc.com (10.4.2.3): icmp_seq=1 ttl=63 time=0.382 ms
64 bytes from server1.hpc.com (10.4.2.3): icmp_seq=2 ttl=63 time=0.371 ms
64 bytes from server1.hpc.com (10.4.2.3): icmp_seq=3 ttl=63 time=0.375 ms

--- server1.hpc.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.371/0.376/0.382/0.004 ms

```

It worked!!

### 7. Re-checking my `ssh` configuration.

```
[root@admin ~]# ssh server1.hpc.com
Last login: Wed Oct 12 17:06:04 2022
[root@server1 ~]# 

```

`ssh` was now working.
