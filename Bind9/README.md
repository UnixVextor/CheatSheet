# Introduction
**Bind9**  is a [DNS](https://www.quickserv.co.th/knowledge-base/solutions/DNS-%e0%b8%84%e0%b8%b7%e0%b8%ad%e0%b8%ad%e0%b8%b0%e0%b9%84%e0%b8%a3/) server that has evolved to be a very flexible, full-featured DNS System.Whatever your application is, in this tutorial I use BIND9 use as my private DNS Server for my Homelab.

# Requirements
- Ubuntu Linux 20.04  or latest version.
- Public or Fake domain name such as `unixvextor.dev`(public) or `onetime.local`(fake) 

# Infrastructure

**This is just a example**

| Host        |        Role        | Private FQDN               | Private IP Address |
| ----------- |:------------------:| -------------------------- | ------------------ | 
| `ns`        |    `DNS Sever`     | `ns.unixvextor.dev`        | `10.10.0.254`      |
| `proxmox`   | `Generic hostname` | `proxmox.unixvextor.com`   | `10.10.0.12`       |  
| `portainer` | `Generic hostname` | `portainer.unixvextor.com` | `10.10.0.13`       |

# Installation
##  1. Install BIND9 on Ubuntu Linux 20.04

Update and Upgrade `apk` package on Ubuntu Linux
```bash
#update apk package
sudo apt update

#upgrade apk upgrade
sudo apt upgrade
```

Install BIND9 
```bash
sudo apt install bind9 bind9utils bind9-doc
```

Set BIND9 is `IPv4` Mode.
```bash
#vim is the text editor.
sudo vim /etc/default/named
```

Add `-4` to the end of the `OPTIONS` parameter.
```conf
OPTIONS=“-u bind -4”
```

Restart BIND9 service to implement the change.
```bash
sudo systemctl restart bind9
```

## 2.Configuring the DNS Server

Open file `named.conf.options` to set configuration of DNS Server.

```bash
sudo vim /etc/bind/named.conf.options
```

Set ACL(Access Control List) block call  `host`. This is which client allow recursive DNS queries.

```conf
#set acl or access control list for what ip can use this DNS Server

acl “host” {
	10.10.0.12 #proxmox
	10.10.0.13 #portainer
};
```

or set `host` block  is Network IP

```conf
#set acl or access control list for what network ip can use this DNS Server

acl “host” {
	10.10.0.0/24 #Network IP
};
```

Set configuration in `options` block

```conf
#set acl or access control list for what ip can use this DNS Server

acl “host” {
	10.10.0.12 #proxmox
	10.10.0.13 #portainer
};

options {
	directory “/var/cache/bind”; #Bind9 Cache data

	# enable recursive queries
	recursion yes; 
	
	# allows recursive queries from “host” client  
	allow-recursion { host; }; 

	# ns private IP address - listen on private network 
	listen-on { 10.10.0.254; };

	# disable zone transfers by default
	allow-transfer { none; };

	# forward if some domain name not in my DNS queries use this 
	forwarders {
		1.1.1.1;
		8.8.8.8;
	}
};
```

### Configuring the Local File

```bash
sudo vim /etc/bind/named.conf.local
```

Inside file `named.conf.local`
 - `zone` is a domain name to use in BIND9.
 
```conf
. . .
zone “unixvextor.dev” { # foward zone
	type primary;
	file “/etc/bind/zones/db.unixvextor.dev”; # zone file path
}

(Optional)
zone “10.10.in-address.arpa” { # Reverse Zone
	type primary;
	file “/etc/bind/zones/db.10.10”; # 10.10.0.0/24 subnet
}
```

### Creating the Forward Zone File

```bash
sudo mkdir /etc/bind/zones # create zone directory 
```

copy example forward zone file to `db.unixvextor.dev`

```bash
sudo cp /etc/bind/db.local /etc/bind/zones/db.unixvextor.dev
```

Edit forward zone file

```bash
sudo vim /etc/bind/zones/db.unixvextor.dev
```

Inside `db.unixvextor.dev`

```conf
$TTL   604800
@      IN       SOA localhost. root.localhost. (
							2       ; Serial
					   604800       ; Refresh
					    86400       ; Retry
					  2419200       ; Expire
					   604800       ; Negative Cache TTL
				  )
;
@      IN       NS  localhost.      ; delete this line
@      IN       A   127.0.0.1       ; delete this line
@      IN       AAA ::1.            ; delete this line
```

Replace to this

```conf
$TTL   604800
@      IN       SOA  unixvextor.dev. admin.unixvextor.dev. (
							3       ; Serial
					   604800       ; Refresh
					    86400       ; Retry
					  2419200       ; Expire
					   604800       ; Negative Cache TTL
				    )
;
; name server - NS records
      IN       NS  ns.unixvextor.dev.

; name server - A records
ns.unixvextor.dev.           IN.    A    10.10.0.254
; 10.10.0.0/24 - A records  
proxmox.unixvextor.dev.      IN     A    10.10.0.12      
portainer.unixvextor.dev.    IN     A    10.10.0.13
```

### (Optional) Reverse Zone File

Copy example reverse Zone file to `db.10.10`

```bash
sudo cp /etc/bind/db.127 /etc/bind/zones/db.10.10
```

Edit reverse zone file

```bash
sudo vim /etc/bind/zones/db.10.10
```

Inside `db.10.10`

```conf
$TTL   604800
@      IN       SOA localhost. root.localhost. (
							2       ; Serial
					   604800       ; Refresh
					    86400       ; Retry
					  2419200       ; Expire
					   604800       ; Negative Cache TTL
				  )
;
@      IN       NS  localhost.      ; delete this line
1.0.0  IN       A   localhost.      ; delete this line
```

Replace to this

```conf
$TTL   604800
@      IN       SOA  unixvextor.dev. admin.unixvextor.dev. (
							3       ; Serial
					   604800       ; Refresh
					    86400       ; Retry
					  2419200       ; Expire
					   604800       ; Negative Cache TTL
				    )
;
; name server - NS records
      IN       NS  ns.unixvextor.dev.

; name server - A records
254.0      IN     PTR ns.unixvextor.com. ; 10.10.0.254
; PTR records  
12.0       IN     PTR proxmox.unixvextor.com. ; 10.10.0.12 
13.0       IN     PTR portainer.unixvextor.com. ; 10.10.0.13
```

## 3.Checking the BIND Configuration Syntax

```bash
sudo named-checkconf
```

If your named configuration file has no error, there won’t be any error message and you will return to your shell prompt.

```bash
# check name zone
sudo named-checkzone unixvextor.dev /etc/bind/zones/db.unixvextor.dev
```

Output like this is OK and does not have any error. 

```
Output
zone unixvextor.dev/IN: loaded serial 3
OK
```

Also check reverse zone file

```bash
sudo named-checkzone 10.10.in-addr.arpa /etc/bind/zones/db.10.10
```

## 4.Restarting BIND

```bash
sudo systemctl restart bind9
```
