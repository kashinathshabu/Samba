# Configuration of Samba CTDB 

## Introduction

CTDB is a clustered database component in clustered Samba that provides a high-availability load-sharing CIFS server cluster.

The main functions of CTDB are:

* Provide a clustered version of the TDB database with automatic rebuild/recovery of the databases upon node failures.
* Monitor nodes in the cluster and services running on each node.
* Manage a pool of public IP addresses that are used to provide services to clients. Alternatively, CTDB can be used with LVS.
* Combined with a cluster filesystem CTDB provides a full high-availability (HA) environment for services such as clustered Samba, NFS and other services.



![ha_samba](https://github.com/user-attachments/assets/b3826b2d-aa11-4ff6-8ef8-bb4e004cb11b)





## Installation of necessary packages

	apt install -y samba ctdb 
	
	systemctl stop smbd.service nmbd.service winbind.service
	
	systemctl disable smbd.service nmbd.service winbind.service

The later three services will be started by ctdbd and not systemd.


## Configuration of CTDB

**All these configuaration files should be same on all nodes !!**

### ctdb.service unit


There are two bug reports open: [No /run/ctdb directory](https://bugs.launchpad.net/ubuntu/+source/ctdb/+bug/1821775) and [No /var/lib/ctdb directories](https://bugs.launchpad.net/ubuntu/+source/ctdb/+bug/1828799)

The following directories have to be created:

	mkdir -p /var/lib/ctdb/volatile
	mkdir /var/lib/ctdb/persistent
	mkdir /var/lib/ctdb/state

### ctdb.conf

The `/etc/ctdb/ctdb.conf` file contains the recovery lock location in the cluster section. This has to be a file on the shared folder:

	[cluster]
		recovery lock = /mnt/shared_foder/ctdb/lock

Create the necessary directory

	mkdir /mnt/shared_folder/ctdb

### nodes

The file `/etc/ctdb/nodes` contains all **node IPs** (i.e. host IPs/ backend IPs). CTDB uses private IP addresses to communicate between nodes. Connections are made to IANA assigned TCP **port 4379** on each node. E.g.:

		
    172.16.1.1
  	172.16.1.2
  	172.16.1.3

### public_addresses

The file `/etc/ctdb/public_addresses` contains the list of **service IPs** (floating IPs) **same amount as node IPs** and the network mask and interfaces they should be applied to:

	192.168.1.11/24  eth0
	192.168.1.12/24  eth0
	192.168.1.13/24  eth0

*The public addresses need to be put into the DNS services for the domain as multiple A records for the name of the fileserver cluster. This creates a round-robin DNS entry which load-balances the clients across the nodes.*

### smb.conf

With CTDB all Samba configuration is stored in the clustered databases. The file `/etc/samba/smb.conf` only contains:

	[global]
		clustering = yes
		include = registry

### script.options

The file `/etc/ctdb/script.options` needs to contain the line:

	CTDB_SAMBA_SKIP_SHARE_CHECK=yes
	

## Configure Samba

Samba is now configured with `smb.conf` and first needs a working `[global]` section:


	[global]
  	clustering = yes
  	include = registry
		workgroup = EXAMPLE
		security = user
		netbios name = FILESERV
		max log size = 50
		log level = 3


### Samba Shares


	[sharename]
		path = /export/sharename
		writable = yes
    read only = no
    guest ok = yes
    browseable = yes


## Start CTDB 

	systemctl enable ctdb.service
	
	systemctl start ctdb.service

### Configure CTDB to take care of Samba-services

    ctdb event script enable legacy 50.samba

## CTDB commands

```
 * ctdb status
 * ctdb uptime
 * ctdb listnodes
 * ctdb event status legacy monitor
 * ctdb event script list legacy```
