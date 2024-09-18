# Configuration of Samba CTDB 

## Introduction

CTDB is a clustered database component in clustered Samba that provides a high-availability load-sharing CIFS server cluster.

The main functions of CTDB are:

* Provide a clustered version of the TDB database with automatic rebuild/recovery of the databases upon node failures.
* Monitor nodes in the cluster and services running on each node.
* Manage a pool of public IP addresses that are used to provide services to clients. Alternatively, CTDB can be used with LVS.
* Combined with a cluster filesystem CTDB provides a full high-availability (HA) environment for services such as clustered Samba, NFS and other services.



![ha_samba](https://github.com/user-attachments/assets/b3826b2d-aa11-4ff6-8ef8-bb4e004cb11b)




## 1. Installation of necessary packages

	apt install -y samba ctdb winbind libpam-winbind libnss-winbind 
 
- Stop and disable samnba services  
 
	  systemctl stop smbd.service nmbd.service winbind.service
	
	  systemctl disable smbd.service nmbd.service winbind.service

The later three services will be started by ctdb and not systemd.


## 2. Configuration of CTDB

**All these configuaration files should be same on all nodes !!**

* ###   ctdb.service unit


There are two bug reports open: [No /run/ctdb directory](https://bugs.launchpad.net/ubuntu/+source/ctdb/+bug/1821775) and [No /var/lib/ctdb directories](https://bugs.launchpad.net/ubuntu/+source/ctdb/+bug/1828799)

The following directories have to be created:

	mkdir -p /var/lib/ctdb/volatile
	mkdir /var/lib/ctdb/persistent
	mkdir /var/lib/ctdb/state

* ###  ctdb.conf

The `/etc/ctdb/ctdb.conf` file contains the recovery lock location in the cluster section. This has to be a file on the shared folder, it can be also a another shared common folder on all the nodes:

	[cluster]
		recovery lock = /mnt/shared_foder/ctdb/lock

![image](https://github.com/user-attachments/assets/b0af0d88-291a-4ff0-afdb-21c47119eb3c)



Create the necessary directory

	mkdir /mnt/shared_folder/ctdb

* ###  nodes

The file `/etc/ctdb/nodes` contains all **node IPs** (i.e. host IPs/ backend IPs). CTDB uses private IP addresses to communicate between nodes. Connections are made to IANA assigned TCP **port 4379** on each node. E.g.:

		
    172.16.1.1
  	172.16.1.2
  	172.16.1.3

* ###  public_addresses

The file `/etc/ctdb/public_addresses` contains the list of **service IPs** (floating IPs) **same amount as node IPs** and the network mask and interfaces they should be applied to:

	192.168.1.11/24  eth0
	192.168.1.12/24  eth0
	192.168.1.13/24  eth0

*The public addresses need to be put into the DNS services for the domain as multiple A records for the name of the fileserver cluster. This creates a round-robin DNS entry which load-balances the clients across the nodes.*

* ###  smb.conf

With CTDB all Samba configuration is stored in the clustered databases. The file `/etc/samba/smb.conf` only contains:

	[global]
		clustering = yes
		include = registry

* ###  script.options

The file `/etc/ctdb/script.options` needs to contain the line:

	CTDB_SAMBA_SKIP_SHARE_CHECK=yes
	

## 3. Configure Samba

Samba is now configured with `smb.conf` and first needs a working `[global]` section:

    [global]
      	workgroup = EXAMPLE
      	security = ads
      	realm = EXAMPLE.COM
      	netbios name = FILESERV
      	winbind use default domain = yes
      	winbind refresh tickets = yes
      	idmap config * : backend = tdb
      	idmap config * : range = 2000-9999
      	idmap config EXAMPLE : backend = rid
      	idmap config EXAMPLE : range = 10000-200000
      	template shell = /bin/bash
      	max log size = 50
      	log level = 3


![image](https://github.com/user-attachments/assets/f45ee8fe-adc5-4187-884f-0f0b509c3f9a)



### * Samba Shares


	[sharename]
        path = /export/sharename
        valid users = @ceph_test.com\cc 
        writable = yes
        read only = no
        guest ok = no
        browseable = yes
        force create mode = 0660
        create mask = 0777
        directory mask = 0777
        force directory mode = 0770
        access based share enum = yes
        hide unreadable = no


![image](https://github.com/user-attachments/assets/f24df77c-3094-4717-ac3c-f93eca463cc5)



### winbind

- Edit  `nano /etc/nsswitch.conf` and add the windbind to the following as shown below:
 
      passwd:         files winbind systemd
      group:          files winbind systemd
      shadow:         files winbind systemd
      gshadow:        files systemd
  

![image](https://github.com/user-attachments/assets/8840fb4a-4077-4a27-83a3-7792e66347de)


- Edit `nano /etc/pam.d/common-session` and add the following to it

      session optional        pam_mkhomedir.so skel=/etc/skel umask=077

![image](https://github.com/user-attachments/assets/e13b5be0-f621-4a45-8465-a1fafad36fd8)


- Now the cluster can be joind as member to the domain with a administrator user. This should be done only from a node.
  
      net ads join -U Administrator

- Restart Winbind and verify 

      systemctl restart winbind
      wbinfo -u 
## 4. Start CTDB 

	systemctl enable ctdb.service
	
	systemctl start ctdb.service

### Enable CTDB events scripts

    ctdb event script enable legacy 50.samba
    ctdb event script enable legacy 49.winbind

## 5. Check CTDB status

    ctdb status

## CTDB commands

```
 * ctdb status
 * ctdb uptime
 * ctdb listnodes
 * ctdb event status legacy monitor
 * ctdb event script list legacy
```
