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

Configure Postfix for SMTP and Dovecot for POP3/IMAP with SASL authentication.

### a) Configure Postfix (SMTP)

**1. Install Postfix:**
```bash
dnf install postfix -y
```

**2. Edit `/etc/postfix/main.cf`:**
```bash
nano /etc/postfix/main.cf
```
First, ensure the basic settings are correct (around line 75-120 and 419):
```ini
myhostname = sysadmin08.unitechlab.net
mydomain = unitechlab.net
myorigin = $mydomain
inet_interfaces = all
mynetworks = 127.0.0.0/8, 172.16.8.0/24
home_mailbox = Maildir/
```

**3. Add SASL and Security Settings:**
Scroll to the very bottom of `/etc/postfix/main.cf` and paste the following block:
```ini
disable_vrfy_command = yes
smtpd_helo_required = yes
message_size_limit = 10240000
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
smtpd_recipient_restrictions = permit_mynetworks, permit_auth_destination, permit_sasl_authenticated, reject
```

### b) Configure Dovecot (POP3/IMAP)

**1. Install Dovecot:**
```bash
dnf install dovecot -y
```

**2. Edit `/etc/dovecot/dovecot.conf`:**
```bash
nano /etc/dovecot/dovecot.conf
```
Find the `listen` line (around line 30) and ensure it is set to:
```ini
listen = *
```

**3. Edit `/etc/dovecot/conf.d/10-auth.conf`:**
```bash
nano /etc/dovecot/conf.d/10-auth.conf
```
*   **Line 10:** Uncomment and set:
    ```ini
    disable_plaintext_auth = no
    ```
*   **Line 100:** Change `auth_mechanisms` to:
    ```ini
    auth_mechanisms = plain login
    ```

**4. Edit `/etc/dovecot/conf.d/10-mail.conf`:**
```bash
nano /etc/dovecot/conf.d/10-mail.conf
```
*   **Line 30:** Uncomment and set:
    ```ini
    mail_location = maildir:~/Maildir
    ```

**5. Edit `/etc/dovecot/conf.d/10-master.conf`:**
```bash
nano /etc/dovecot/conf.d/10-master.conf
```
Scroll down to the `service auth` block (around line 95). Inside this block, find the `unix_listener` section (lines 107-109) and modify it to look exactly like this:
```ini
service auth {
  # ... other lines ...
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
  # ... other lines ...
}
```

**6. Edit `/etc/dovecot/conf.d/10-ssl.conf`:**
```bash
nano /etc/dovecot/conf.d/10-ssl.conf
```
*   **Line 8:** Ensure SSL is enabled:
    ```ini
    ssl = yes
    ```

### c) Start Services & Firewall

**1. Configure Firewall:**
```bash
firewall-cmd --add-service={smtp,pop3,imap} --permanent
firewall-cmd --reload
```

**2. Enable and Start Services:**
```bash
systemctl restart postfix
systemctl enable postfix
systemctl restart dovecot
systemctl enable dovecot
```

**3. Verification:**
Check if services are running and listening:
```bash
systemctl status postfix dovecot
ss -tunlp | grep -E '25|110|143|587'
```

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
nano /home/TeamProjects/readme.txt
(put your fullname and student ID here)

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

