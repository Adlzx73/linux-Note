# Linux Lab Exercise (Rocky Linux)

This repository contains the step-by-step configuration for setting up essential network services on a Rocky Linux server.

## 📋 Scenario & Environment

**Scenario:** Junior System Administrator Trainee at UniTech Training Centre.  
**OS:** Rocky Linux 9  
**Network:** 192.168.0.0/24 (Simulated as 172.16.8.0/24 for this specific setup)  
**Station Number (SN):** 08

### Server Details
*   **IP Address:** `172.16.8.8`
*   **Gateway:** `172.16.8.1`
*   **Hostname:** `sysadmin08.unitechlab.net`
*   **Domain:** `unitechlab.net`

---

## 🛠️ Prerequisites: Initial Setup

Connect via SSH and set the hostname.

```bash
# Set the hostname matching the SN
hostnamectl set-hostname sysadmin08.unitechlab.net

# Verify
hostname
```

---

## 1. DNS Server (BIND)

Configure BIND to handle name resolution for `unitechlab.net`.

### a) Installation & Configuration

**Install BIND:**
```bash
dnf install bind bind-utils -y
```

**Edit `/etc/named.conf`:**
```bash
nano /etc/named.conf
```
Modify the `options` block:
```conf
options {
    listen-on port 53 { 127.0.0.1; 172.16.8.8; };
    allow-query     { localhost; 172.16.8.0/24; };
    ...
};
```
Add zone definitions at the bottom:
```conf
zone "unitechlab.net" IN {
    type master;
    file "unitechlab.net.fwd";
    allow-update { none; };
};

zone "8.16.172.in-addr.arpa" IN {
    type master;
    file "unitechlab.net.rev";
    allow-update { none; };
};
```

**Create Forward Zone (`/var/named/unitechlab.net.fwd`):**
```dns
$TTL 86400
@   IN  SOA     sysadmin08.unitechlab.net. root.unitechlab.net. (
        2025102801  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)
@   IN  NS      sysadmin08.unitechlab.net.
unitechlab.net.         IN  A   172.16.8.8
sysadmin08.unitechlab.net.  IN  A   172.16.8.8
```

**Create Reverse Zone (`/var/named/unitechlab.net.rev`):**
```dns
$TTL 86400
@   IN  SOA     sysadmin08.unitechlab.net. root.unitechlab.net. (
        2025102801  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)
@   IN  NS      sysadmin08.unitechlab.net.
8   IN  PTR     sysadmin08.unitechlab.net.
```

### b) Start Services

```bash
# Set permissions
chown root:named /etc/named.conf
chgrp named /var/named/unitechlab.net.fwd /var/named/unitechlab.net.rev

# Firewall
firewall-cmd --add-service=dns --permanent
firewall-cmd --reload

# Start Service
systemctl start named
systemctl enable named
```

### c) Verification

Update `/etc/resolv.conf` to use `nameserver 172.16.8.8`, then run:
```bash
nslookup unitechlab.net
nslookup 172.16.8.8
```

---

## 2. DHCP Server

Configure dynamic IP assignment for the range `172.16.8.200` - `172.16.8.250`.

### a) Configuration

**Install DHCP:**
```bash
dnf install dhcp-server -y
```

**Edit `/etc/dhcp/dhcpd.conf`:**
```conf
subnet 172.16.8.0 netmask 255.255.255.0 {
  range 172.16.8.200 172.16.8.250;
  option domain-name-servers 172.16.8.8;
  option domain-name "unitechlab.net";
  option routers 172.16.8.1;
  option broadcast-address 172.16.8.255;
  default-lease-time 600;
  max-lease-time 7200;
}
```

### b) Start Services

```bash
firewall-cmd --add-service=dhcp --permanent
firewall-cmd --reload

systemctl start dhcpd
systemctl enable dhcpd
```

---

## 3. FTP Server (Vsftpd)

Secure FTP configuration restricting users to home directories.

### a) Configuration

**Install Vsftpd:**
```bash
dnf install vsftpd -y
```

**Edit `/etc/vsftpd/vsftpd.conf`:**
```conf
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES
```

**Create User:**
```bash
useradd ftpuser
echo "P@ssw0rd" | passwd --stdin ftpuser
```

### b) Start Services

```bash
# Firewall
firewall-cmd --add-service=ftp --permanent
firewall-cmd --reload

# SELinux (Important)
setsebool -P ftpd_full_access on

# Start Service
systemctl start vsftpd
systemctl enable vsftpd
```

---

## 4. Mail Server (Postfix + Dovecot)

### a) Postfix (SMTP)

**Install:**
```bash
dnf install postfix -y
```

**Edit `/etc/postfix/main.cf`:**
```conf
myhostname = sysadmin08.unitechlab.net
mydomain = unitechlab.net
myorigin = $mydomain
inet_interfaces = all
mynetworks = 127.0.0.0/8, 172.16.8.0/24
home_mailbox = Maildir/
```

### b) Dovecot (POP3/IMAP)

**Install:**
```bash
dnf install dovecot -y
```

**Edit `/etc/dovecot/conf.d/10-mail.conf`:**
```conf
mail_location = maildir:~/Maildir
```

### c) Start Services & Firewall

```bash
firewall-cmd --add-service={smtp,pop3,imap} --permanent
firewall-cmd --reload

systemctl start postfix
systemctl enable postfix
systemctl start dovecot
systemctl enable dovecot
```

---

## 5. User & Group Management

```bash
# Create Group
groupadd TeamLab

# Create Users and Add to Group
useradd -G TeamLab user1
useradd -G TeamLab user2

# Verify
id user1
```

---

## 6. File & Directory Management

Create a shared project folder with restricted permissions.

```bash
# Create Directory
mkdir /home/TeamProjects

# Create File
echo "Full Name: [YOUR NAME], Student ID: [YOUR ID]" > /home/TeamProjects/readme.txt

# Set Permissions (Group ownership and Access Control)
chown -R root:TeamLab /home/TeamProjects
chmod -R 770 /home/TeamProjects
```

---

## 7. Samba Shared Folder

Share `/home/TeamProjects` with Windows/Linux clients.

### a) Configuration

**Install Samba:**
```bash
dnf install samba samba-client -y
```

**Edit `/etc/samba/smb.conf`:**
Append to end of file:
```ini
[Team_Share]
    path = /home/TeamProjects
    guest ok = yes
    writable = yes
    browseable = yes
```

### b) Start Services

```bash
# SELinux Context (Critical for Access)
chcon -t samba_share_t /home/TeamProjects

# Firewall
firewall-cmd --add-service=samba --permanent
firewall-cmd --reload

# Start Services
systemctl start smb nmb
systemctl enable smb nmb
```

### c) Access Testing

**Windows:** `\\172.16.8.8\Team_Share`  
**Linux:** `smbclient //172.16.8.8/Team_Share -U %`

```
