---
![Keenable](https://user-images.githubusercontent.com/67622286/115836356-02ac4b80-a435-11eb-854f-fa32663ccd6a.png)
---
---
# ℙ𝕘𝕊ℚ𝕃 ℝ𝕖𝕡𝕝𝕚𝕔𝕒𝕥𝕖𝕕 ℂ𝕝𝕦𝕤𝕥𝕖𝕣
---
## ʜᴏᴡ ᴛᴏ ɪɴꜱᴛᴀʟʟ ᴀ 2-ɴᴏᴅᴇ ᴘʀᴏᴍᴏᴛᴀʙʟᴇ (ꜱɪɴɢʟᴇ-ᴍᴀꜱᴛᴇʀ) ᴄʟᴜꜱᴛᴇʀ ᴏɴ ᴄᴇɴᴛᴏꜱ 7 ᴜꜱɪɴɢ ᴘᴏꜱᴛɢʀᴇꜱQʟ-11 ꜱʏɴᴄʜʀᴏɴᴏᴜꜱ ꜱᴛʀᴇᴀᴍɪɴɢ ʀᴇᴘʟɪᴄᴀᴛɪᴏɴ.
---


   <summary><h2 align="Left">Table of Contents</h2></summary>
  <ul>
      <li><a href="#Prerequisite">Prerequisite</a></li>
      <li><a href="#Installation">Installation</a></li>
      <li><a href="#">Configure Postgre SQL Setup</a></li>
      <li><a href="#">Configure & Start Corosync on both nodes</a></li>
      <li><a href="#">Start Pacemaker on both nodes</a></li>
      <li><a href="#">Start pcsd service on both nodes</a></li>
      <li><a href="#">Check PCS Status & Make configuration file(config.pcs) for PCS command</a></li>
      <li><a href="#">Enable Pacemaker and Corosync Service</a></li>
      <li><a href="#Operations">Operations</a></li>
      <li><a href="#">About Nodes Attributes</a></li>
  <li><a href="#">PGSQL Status</a></li>
  <li><a href="#">PGSQL Data Status</a></li>
      <li><a href="#">PGSQL-xlog-replay-loc</a></li>      
  </ul>  


<summary><h2 align="Left">ℙ𝕣𝕖𝕣𝕖𝕢𝕦𝕚𝕤𝕚𝕥𝕖</h2></summary>

### Software version
```
We developed pgsql RA using pacemaker 1.1.23 with heartbeat stack and resource-agents 4.1.1.

```

### Network Topology

- It's recommended that LANs(for service, pacemaker, replication) are separated and redundant.But we use these topology in this document to explain simply.
```

𝚗𝚘𝚍𝚎𝟷
- eth0 : 192.168.0.1 : LAN for service
- eth1 : 192.168.1.1 : LAN for Pacemaker
- eth2 : 192.168.2.1 : LAN for replication

𝚗𝚘𝚍𝚎𝟸
- eth0 : 192.168.0.2 : LAN for service
- eth1 : 192.168.1.2 : LAN for Pacemaker
- eth2 : 192.168.2.2 : LAN for replication

```
### virtual IP (vip)
```
- virtual IP1 : 192.168.0.3 : vip for eth0 (DB client connects this IP to access PostgreSQL(Master))
- virtual IP2 : 192.168.2.3 : vip for eth2 (replica connects this IP to replicate)

```

### PostgreSQL
```
Use PostgreSQL 9.1 or later.

```

### Parameters of pgsql RA

>The following parameters are added for replication:
<p style='text-align: justify;'>
  
- **rep_mode** - Choice from async or sync to use replication."async" is used for async mode only, "sync" is used for switching between sync mode and async mode.The following parameter node_list master_ip, and restore_command is necessary at async or sync modes(*).
- **node_list(*)** - The list of PRI and HS node names. Specifies a space-separated list of all node name (result of the uname -n command).
- **master_ip(*)** - HS connects to this IP. It equals virtual IP2 in this documents.
- **restore_command(*)** - restore_command specified in recovery.conf file when starting with HS.
- **repuser** - The user of replication which HS connects to PRI. Default is "postgres". Please use .pgpass file to set password.
- **primary_conninfo_opt** - RA generates recovery.conf file to start PostgreSQL as HS. host,port,user and application name of primary_conninfo are automatically set by RA. If you would like to set some additional parameters, you can specify them here.
- **tmpdir** - the rep_mode_conf and xlog_note.* and PGSQL.lock files are created in this directory. Default is /var/lib/pgsql/tmp directory. If the directory doesn't exist, RA makes it automatically.
- **xlog_check_count** - The count of cheking last_xlog_replay_location or last_xlog_receive_location is specified to compare data. Default is 3(times). It is counted at monitor interval.
- **stop_escalate_in_slave** - Number of shutdown retries (using -m fast) before resorting to -m immediate in replica state. In Master sate, you can use "stop_escalate" which is provided since early times.
- **restart_on_promote** - RA restarts PostgreSQL on promote instead of promote to prevent from increasing Timeline ID of PostgreSQL since HS can't connect PRI if Timeline ID is different. Default is false and you should copy data from PRI to align Timeline ID after promoting.
</p>


<summary><h2 align="Left">𝕀𝕟𝕤𝕥𝕒𝕝𝕝𝕒𝕥𝕚𝕠𝕟</h2></summary>
 
<h2>𝙸𝚗 𝚝𝚑𝚒𝚜 𝚍𝚘𝚌𝚞𝚖𝚎𝚗𝚝𝚜, 𝚠𝚎 𝚞𝚜𝚎 𝙲𝚎𝚗𝚝𝚘𝚜 𝟽</h2>

### OS (Centos 7) (both nodes)
- disable selinux
   - edit /etc/selinux/config
- disable firewall(iptables)
   - systemctl stop firewalld.service
   - systemctl disable firewalld.service
- set node name "node1" and "node2". 
   - edit /etc/hosts
- stop and disable NetworkManager and set up IPs as above.
   - systemctl stop NetworkManager.service 
   - systemctl disable NetworkManager.service 
- edit /etc/sysconfig/network-scripts/ifcfg-ethX

### Configure Yum Repo
```
#rpm -Uvh https://yum.postgresql.org/11/redhat/rhel-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

```

### Packages (both nodes)
```
#yum -y install postgresql11-server pacemaker corosync pcs

```

### Initialize Database
```
#/usr/pgsql-11/bin/postgresql-11-setup initdb
```

### Replacement of pgsql RA (both nodes)
```
#wget https://raw.github.com/ClusterLabs/resource-agents/master/heartbeat/pgsql
#cp pgsql /usr/lib/ocf/resource.d/heartbeat/
#chmod 755 /usr/lib/ocf/resource.d/heartbeat/pgsql
#vim /usr/lib/ocf/resource.d/heartbeat/pgsql
```
##### Replace below lines in pgsql(line no. 16)
```
#Initialization:

: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/lib/heartbeat}
  
```
    
##### with
```
#Initialization:
OCF_ROOT=/usr/lib/ocf
: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/lib/heartbeat}

```

##### Check PostgreSQL path in pgsql(line no. 47) 
```
Default value: OCF_RESKEY_pgdata_default=/var/lib/pgsql/data
for Postgresql 11: OCF_RESKEY_pgdata_default=/var/lib/pgsql/11/data
```


<summary><h2 align="Left">ℂ𝕠𝕟𝕗𝕚𝕘𝕦𝕣𝕖 ℙ𝕠𝕤𝕥𝕘𝕣𝕖 𝕊ℚ𝕃 𝕊𝕖𝕥𝕦𝕡</h2></summary>

### PostgreSQL (node1 only)
```
#su - postgres
$mkdir -m 700 /var/lib/pgsql/11/pg_archive
```

### Edit postgresql.conf.
>The main set part as follows. Please refer to the manual of PostgreSQL for other parameter.
Check the starting  with the PostgreSQL unit, and the replication is possible.

```
listen_addresses = '*'
wal_level = hot_standby
synchronous_commit = on
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/11/pg_archive/%f'
max_wal_senders=5
wal_keep_segments = 32
hot_standby = on
restart_after_crash = off
wal_receiver_status_interval = 2
max_standby_streaming_delay = -1
max_standby_archive_delay = -1
hot_standby_feedback = on

```
```
Edit pg_hba.conf.
Be careful. This explanation is not considered about the security.
host    all             all     127.0.0.1/32        trust
host    all             all     192.168.0.0/16      trust
host    replication     all     192.168.0.0/16      trust

```
#### Start Postgresql on node1

```
 #su - postgres
 $ pg_ctl -D /var/lib/pgsql/11/data start
```

#### Co𝚙𝚢 𝚍𝚊𝚝𝚊 𝚏𝚛𝚘𝚖 𝚗𝚘𝚍𝚎𝟷 𝚝𝚘 𝚗𝚘𝚍𝚎𝟸

```
 #su - postgres
 $ rm -rf /var/lib/pgsql/11/data/*
 $ pg_basebackup -h 192.168.2.1 -U postgres -D /var/lib/pgsql/11/data -X stream -P
 $ mkdir -m 700 /var/lib/pgsql/11/pg_archive
```

```
Create /var/lib/pgsql/11/data/recovery.conf to confirm replication.
 standby_mode = 'on'
 primary_conninfo = 'host=192.168.2.1 port=5432 user=postgres application_name=node2'
 restore_command = 'cp /var/lib/pgsql/11/pg_archive/%f %p'
 recovery_target_timeline = 'latest'
```

### Start PostgreSQL on node2

```
 $ pg_ctl -D /var/lib/pgsql/11/data/ start
```

 
  
### Confirm PostgreSQL replication success (node1 only)

```
#su - postgres
```

- $ psql -c "select client_addr,sync_state from pg_stat_replication;"

|                              |        |         
| :--------------------------------:|:------------------------------:|      
 | client_addr | sync_state|
 |192.168.2.2| sync |
 
### Stop Postgresql on both nodes

```
 $ pg_ctl -D /var/lib/pgsql/11/data stop
 $ exit
 ```


<summary><h2 align="Left">ℂ𝕠𝕟𝕗𝕚𝕘𝕦𝕣𝕖 & 𝕊𝕥𝕒𝕣𝕥 ℂ𝕠𝕣𝕠𝕤𝕪𝕟𝕔 𝕠𝕟 𝕓𝕠𝕥𝕙 𝕟𝕠𝕕𝕖𝕤</h2></summary>

```
Create /etc/corosync/corosync.conf
quorum {
    provider: corosync_votequorum
    expected_votes: 2
}
aisexec {
    user: root
    group: root
}
service {
    name: pacemaker
    ver: 0
}
totem {
    version: 2
    secauth: off
    interface {
        ringnumber: 0
        bindnetaddr: 192.168.1.0
        mcastaddr: 239.255.1.1
    }
}
logging {
    to_syslog: yes
}
```

### Start corosync

```
#systemctl start corosync.service
```

>You can see this log in /var/log/messages when you succeed in starting corosync.
 Starting Corosync Cluster Engine (corosync): [  OK  ]

  
<summary><h2 align="Left">𝕊𝕥𝕒𝕣𝕥 ℙ𝕒𝕔𝕖𝕞𝕒𝕜𝕖𝕣 𝕠𝕟 𝕓𝕠𝕥𝕙 𝕟𝕠𝕕𝕖𝕤</h2></summary>
- Clear current settings if it exists.

```
#rm -f /var/lib/pacemaker/cib/cib*
```
  
### Start pacemaker

```
#systemctl start pacemaker.service
```


<summary><h2 align="Left">𝕊𝕥𝕒𝕣𝕥 𝕡𝕔𝕤𝕕 𝕤𝕖𝕣𝕧𝕚𝕔𝕖 𝕠𝕟 𝕓𝕠𝕥𝕙 𝕟𝕠𝕕𝕖𝕤</h2></summary>

```
#systemctl enable pcsd.service
#systemctl start pcsd.service

```


<summary><h2 align="Left">ℂ𝕙𝕖𝕔𝕜 ℙℂ𝕊 𝕊𝕥𝕒𝕥𝕦𝕤 & 𝕄𝕒𝕜𝕖 𝕔𝕠𝕟𝕗𝕚𝕘𝕦𝕣𝕒𝕥𝕚𝕠𝕟 𝕗𝕚𝕝𝕖(𝕔𝕠𝕟𝕗𝕚𝕘.𝕡𝕔𝕤) 𝕗𝕠𝕣 ℙℂ𝕊 𝕔𝕠𝕞𝕞𝕒𝕟𝕕</h2></summary>

```
#crm_mon -Afr -1
```

```
Ｓｔａｃｋ： ｃｏｒｏｓｙｎｃ
Current DC: node2 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Mon May 29 19:30:29 2021
Last change: Mon May 29 19:30:29 2021 by root via crm_attribute on node2

2 Nodes configured, unknown expected votes
0 Resources configured.

Online: [ node1 node2 ]

Ｆｕｌｌ　ｌｉｓｔ　ｏｆ　ｒｅｓｏｕｒｃｅｓ：

𝑵𝒐𝒅𝒆 𝑨𝒕𝒕𝒓𝒊𝒃𝒖𝒕𝒆𝒔:
- Node node1:
- Node node2:

𝑴𝒊𝒈𝒓𝒂𝒕𝒊𝒐𝒏 𝒔𝒖𝒎𝒎𝒂𝒓𝒚:
- Node node1:
- Node node2:
```

### Make Configuration file (config.pcs)

>In this sample configuration, "vip-master" means virtual IP1 and "vip-rep" means virtual IP2,and we use restart_on_promote="true" to explain operations simply.
If you use false,you should start pacemaker on node1 only and laod configuration.After that you should copy data from node1 to node2 and start pacemaker on node2 to align  Timeline ID.

```

pcs cluster cib pgsql_cfg

pcs -f pgsql_cfg property set no-quorum-policy="ignore"
pcs -f pgsql_cfg property set stonith-enabled="false"
pcs -f pgsql_cfg resource defaults resource-stickiness="INFINITY"
pcs -f pgsql_cfg resource defaults migration-threshold="1"

pcs -f pgsql_cfg resource create vip-master IPaddr2 \
   ip="192.168.0.3" \
   nic="eth0" \
   cidr_netmask="24" \
   op start   timeout="60s" interval="0s"  on-fail="restart" \
   op monitor timeout="60s" interval="10s" on-fail="restart" \
   op stop    timeout="60s" interval="0s"  on-fail="block"

pcs -f pgsql_cfg resource create vip-rep IPaddr2 \
   ip="192.168.2.3" \
   nic="eth2" \
   cidr_netmask="24" \
   meta migration-threshold="0" \
   op start   timeout="60s" interval="0s"  on-fail="stop" \
   op monitor timeout="60s" interval="10s" on-fail="restart" \
   op stop    timeout="60s" interval="0s"  on-fail="ignore"

pcs -f pgsql_cfg resource create pgsql pgsql \
   pgctl="/usr/bin/pg_ctl" \
   psql="/usr/bin/psql" \
   pgdata="/var/lib/pgsql/11/data/" \
   rep_mode="sync" \
   node_list="node1 node2" \
   restore_command="cp /var/lib/pgsql/11/pg_archive/%f %p" \
   primary_conninfo_opt="keepalives_idle=60 keepalives_interval=5 keepalives_count=5" \
   master_ip="192.168.2.3" \
   restart_on_promote='true' \
   op start   timeout="60s" interval="0s"  on-fail="restart" \
   op monitor timeout="60s" interval="4s" on-fail="restart" \
   op monitor timeout="60s" interval="3s"  on-fail="restart" role="Master" \
   op promote timeout="60s" interval="0s"  on-fail="restart" \
   op demote  timeout="60s" interval="0s"  on-fail="stop" \
   op stop    timeout="60s" interval="0s"  on-fail="block" \
   op notify  timeout="60s" interval="0s"

pcs -f pgsql_cfg resource master msPostgresql pgsql \
   master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true

pcs -f pgsql_cfg resource group add master-group vip-master vip-rep

pcs -f pgsql_cfg constraint colocation add master-group with Master msPostgresql INFINITY
pcs -f pgsql_cfg constraint order promote msPostgresql then start master-group symmetrical=false score=INFINITY
pcs -f pgsql_cfg constraint order demote  msPostgresql then stop  master-group symmetrical=false score=0

pcs cluster cib-push pgsql_cfg
```

### Load configuration

```
#sh config.pcs
```

### Check status again

```
#crm_mon -Afr -1
Stack: corosync
Current DC: node1 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Sat May 29 21:02:34 2021
Last change: Sat May 29 21:02:33 2021 by root via crm_attribute on node1

2 nodes configured
4 resource instances configured

Online: [ node1 node2 ]

Full list of resources:

 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node1
     vip-rep  (ocf::heartbeat:IPaddr2): Started node1

𝑵𝒐𝒅𝒆 𝑨𝒕𝒕𝒓𝒊𝒃𝒖𝒕𝒆𝒔:
- Node node1:
    + master-pgsql                      : 1000      
    + pgsql-data-status                 : LATEST    
    + pgsql-master-baseline             : 0000000006000098
    + pgsql-status                      : PRI       
- Node node2:
    + master-pgsql                      : 100       
    + pgsql-data-status                 : STREAMING|SYNC
    + pgsql-status                      : HS:sync   

𝑴𝒊𝒈𝒓𝒂𝒕𝒊𝒐𝒏 𝒔𝒖𝒎𝒎𝒂𝒓𝒚:
 Node node2:
 Node node1:
```


<summary><h2 align="Left">𝔼𝕟𝕒𝕓𝕝𝕖 ℙ𝕒𝕔𝕖𝕞𝕒𝕜𝕖𝕣 𝕒𝕟𝕕 ℂ𝕠𝕣𝕠𝕤𝕪𝕟𝕔 𝕊𝕖𝕣𝕧𝕚𝕔𝕖</h2></summary>

```
 #systemctl enable corosync.service
 #systemctl enable pacemaker.service

```


<summary><h2 align="Left">𝕆𝕡𝕖𝕣𝕒𝕥𝕚𝕠𝕟𝕤</h2></summary>
 <h2>𝙰𝚏𝚝𝚎𝚛 𝚏𝚊𝚒𝚕-𝚘𝚟𝚎𝚛</h2>

### Kill PostgreSQL process at node1 to occur fail-over
```
#killall -9 postgres 
```
### Check status
```
#crm_mon -Afr -1
Stack: corosync
Current DC: node1 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Sat May 29 21:05:09 2021
Last change: Sat May 29 21:02:33 2021 by root via crm_attribute on node1

2 nodes configured
4 resource instances configured

Online: [ node1 node2 ]

Full list of resources:

 Master/Slave Set: msPostgresql [pgsql]
     Slaves: [ node2 ]
     Stopped: [ node1 ]
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Stopped
     vip-rep  (ocf::heartbeat:IPaddr2): Stopped

Node Attributes:
- Node node1:
    + master-pgsql                      : -INFINITY 
    + pgsql-data-status                 : DISCONNECT    
    + pgsql-status                      : STOP      
- Node node2:
    + master-pgsql                      : 1000       
    + pgsql-data-status                 : LATEST
    + pgsql-master-baseline             : 0000000006017680
    + pgsql-status                      : PRI  

Migration Summary:
- Node node2:
- Node node1:
   pgsql: migration-threshold=1 fail-count=1 last-failure='Sat May 29 21:05:07 2021'

Failed Resource Actions:
- pgsql_monitor_3000 on node1 'unknown error' (1): call=23, status=complete, exitreason='',
    last-rc-change='Sat May 29 21:05:07 2021', queued=0ms, exec=0ms
```
### Recovery node1 as replica
>Copy all data from node2 because the data of node1 may be inconsistent.
```

 ##### (at node1)
 #su - postgres
 $ rm -rf /var/lib/pgsql/11/data/
 $ pg_basebackup -h 192.168.2.3 -U postgres -D /var/lib/pgsql/11/data -X stream -P
 $ rm /var/lib/pgsql/11/tmp/PGSQL.lock
 $ exit
 
 #pcs resource cleanup msPostgresql
 
 ```

### Check status again
```
#crm_mon -Afr -1

Stack: corosync
Current DC: node2 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Mon May 31 09:53:18 2021
Last change: Mon May 31 08:29:16 2021 by root via crm_attribute on node2

2 nodes configured
4 resource instances configured

Online: [ node1 node2 ]

Full list of resources:

 Master/Slave Set: msPostgresql [pgsql]
     Masters: [ node2 ]
     Slaves: [ node1 ]
 Resource Group: master-group
     vip-master (ocf::heartbeat:IPaddr2): Started node2
     vip-rep  (ocf::heartbeat:IPaddr2): Started node2

Node Attributes:
- Node node1:
    + master-pgsql                      : 100       
    + pgsql-data-status                 : STREAMING|SYNC
    + pgsql-status                      : HS:sync   
- Node node2:
    + master-pgsql                      : 1000      
    + pgsql-data-status                 : LATEST    
    + pgsql-master-baseline             : 000000000A000098
    + pgsql-status                      : PRI       

Migration Summary:
- Node node1:
- Node node2:

```
<summary><h2 align="Left">𝔸𝕓𝕠𝕦𝕥 ℕ𝕠𝕕𝕖𝕤 & 𝔸𝕥𝕥𝕣𝕚𝕓𝕦𝕥𝕖𝕤</h2></summary>
 <h2>The RA defines the following states as a node attribute value of Pacemaker</h2>
 
- Attribute can be seen in "crm_mon -A". 

<summary><h2 align="Left">ℙ𝔾𝕊ℚ𝕃 𝕊𝕥𝕒𝕥𝕦𝕤</h2></summary>

- A present state of PostgreSQL is displayed by the attribute value to which PRI or either HS node is displayed.

```
- 𝐒𝐓𝐎𝐏 = PostgreSQL has stopped.
- 𝐇𝐒:𝐚𝐥𝐨𝐧𝐞 - It works as HS but HS doesn't connect PRI.
- 𝐇𝐒:𝐜𝐨𝐧𝐧𝐞𝐜𝐭𝐞𝐝 - It works as HS,and connected with PRI but HS isn't normal state of the replication (Data has not caught up with PRI).
- 𝐇𝐒:𝐚𝐬𝐲𝐧𝐜 - It works as HS, and the state is async mode. When RA is used in the rep_mode=async, it is possible to be promoted to Master.
- 𝐇𝐒:𝐬𝐲𝐧𝐜 - It works as HS, and the state is sync mode. It is possible to be promoted to Master for the rep_mode=sync.
- 𝐏𝐑𝐈 - It operates by PRI.

```

<details><summary><h2 align="Left">ℙ𝔾𝕊ℚ𝕃 𝔻𝕒𝕥𝕒 𝕊𝕥𝕒𝕥𝕦𝕤</h2></summary>

- The transitional state of data is displayed. This state remains after stopping pacemaker.
- When starting pacemaker next time, this state is used to judge whether my data is old or not.

```

- 𝐃𝐈𝐒𝐂𝐎𝐍𝐍𝐄𝐂𝐓 - Master changes other node state into DISCONNECT if Master can't detect connection of replication because of LAN failure or breakdown of replica and so on.
- {𝐬𝐭𝐚𝐭𝐞}|{𝐬𝐲𝐧𝐜_𝐬𝐭𝐚𝐭𝐞} - Master changes other node state into {state}|{sync_state} if Master detects connection of replication {state} and {sync_state} means state of replication which is retrieved using "select state and sync_state from pg_stat_replication" on Master.For example,  INIT, CATCHUP, and STREAMING are displayed in {state} and ASYNC, SYNC are displayed in {sync_state}
- 𝐋𝐀𝐓𝐄𝐒𝐓 - It's displayed when it's Master.
```
- These states are the transitional state of final data, and it may be not consistent with the state of actual data.
- For instance, During PRI, the state is "LATEST". But the node is stopped or down, this state "LATEST" is maintained if Master doesn't exist in other nodes. It never changes to "DISCONNECT" for oneself.
- When other node newly is promoted, this new Master changes the state of old Master to "DISCONNECT". When any node can not become Master, this "LATEST" will be keeped.
</details>
<details><summary><h2 align="Left">ℙ𝔾𝕊ℚ𝕃-𝕩𝕝𝕠𝕘-𝕣𝕖𝕡𝕝𝕒𝕪-𝕝𝕠𝕔</h2></summary>

- There is no Master node, it is displayed. RA  decide to promote one node to Master comparing with the value of the last_replay_xlog_location or last_receive_xlog_replay_location among other node.

</details>

<details><summary><h2 align="left">𝓓𝓸𝓬𝓾𝓶𝓮𝓷𝓽𝓮𝓭 𝓑𝔂</h2></summary>
  
|                              |                      |
| :-------------------:|:------------------------------: |        
 |*Name* | **Monika Verma** |  
|*Org.* | ****Keen & Able computers Pvt Ltd****  |  
| *Title*  | **Project Coordinator** |
| *Date*  | **31st May 2021**     |

</details>

### *Thank You*
