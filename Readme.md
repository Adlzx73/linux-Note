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

Configure specific FTP settings, chroot lists, and firewall rules as per the provided configuration images.

### a) Installation & Configuration

**1. Install Vsftpd:**
```bash
dnf -y install vsftpd
```

**2. Configure `/etc/vsftpd/vsftpd.conf`:**
```bash
nano /etc/vsftpd/vsftpd.conf
```
Locate and change (or add) the following parameters to match these exact values:

```ini
# Line 12: Disable anonymous
anonymous_enable=NO

# Line 82, 83: Allow ASCII mode
ascii_upload_enable=YES
ascii_download_enable=YES

# Line 100, 101: Enable chroot and list
chroot_local_user=YES
chroot_list_enable=YES

# Line 103: Specify chroot list file
chroot_list_file=/etc/vsftpd/chroot_list

# Line 109: Enable recursive list
ls_recurse_enable=YES

# Line 114: Listen settings (Set NO if listening on IPv6 too)
listen=NO

# Line 123: IPv6 Listen settings
listen_ipv6=YES

# Add the following lines to the end of the file:
local_root=public_html
use_localtime=YES
seccomp_sandbox=NO
```

**3. Configure Chroot List:**
Create the exception list file. Users in this file will be allowed to move out of their home directory.
```bash
nano /etc/vsftpd/chroot_list
```
Add the username inside this file:
```text
ftpuser
```

### b) User Account Setup

**1. Create User:**
*Note: The images used 'ikmbest', but we use 'ftpuser' to match the Lab PDF requirements.*
```bash
useradd ftpuser
passwd ftpuser
# Enter password: P@ssw0rd
# Re-type: P@ssw0rd
```

**2. Create Local Root Directory:**
Since we set `local_root=public_html` in the config, we must create this folder inside the user's home, or login will fail.
```bash
mkdir /home/ftpuser/public_html
chown ftpuser:ftpuser /home/ftpuser/public_html
chmod 755 /home/ftpuser/public_html
```

### c) Start Services & Security

**1. Enable Service:**
```bash
systemctl enable --now vsftpd
```

**2. Configure SELinux:**
```bash
setsebool -P ftpd_full_access on
```

**3. Configure Firewall:**
```bash
firewall-cmd --add-service=ftp
firewall-cmd --runtime-to-permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all
```

### d) Access Testing from Client

**Method 1: Windows File Explorer (Visual)**
1.  Open **File Explorer** on your Windows client.
2.  In the address bar at the top, type:
    ```text
    ftp://172.16.8.8
    ```
3.  Press **Enter**.
4.  A login prompt will appear:
    *   **User:** `ftpuser`
    *   **Password:** `P@ssw0rd`
5.  You should successfully log in and see the contents of the `public_html` folder.

**Method 2: Command Line (Windows CMD or Linux)**
1.  Open Command Prompt (Windows) or Terminal (Linux).
2.  Connect to the server:
    ```bash
    ftp 172.16.8.8
    ```
3.  Enter credentials when prompted:
    *   **Name:** `ftpuser`
    *   **Password:** `P@ssw0rd`
4.  If successful, you will see `230 Login successful`.
5.  Run commands to test:
    ```bash
    pwd   # Check current directory (should be /)
    ls    # List files
    bye   # Exit
    ```

---

## 4. Mail Server (Postfix + Dovecot)

Configure Postfix for SMTP and Dovecot for POP3/IMAP.

### a) Configure Postfix (SMTP)

**1. Install Postfix:**
```bash
dnf install postfix -y
```

**2. Edit `/etc/postfix/main.cf`:**
```bash
nano /etc/postfix/main.cf
```
Locate and modify the following lines (line numbers are approximate):

*   **Line 95:** Uncomment and set hostname:
    ```ini
    myhostname = sysadmin08.unitechlab.net
    ```
*   **Line 102:** Uncomment and set domain:
    ```ini
    mydomain = unitechlab.net
    ```
*   **Line 118:** Uncomment origin:
    ```ini
    myorigin = $mydomain
    ```
*   **Line 132:** Uncomment interfaces:
    ```ini
    inet_interfaces = all
    ```
*   **Line 135:** Comment out localhost binding:
    ```ini
    #inet_interfaces = localhost
    ```
*   **Line 138:** Change protocol to IPv4:
    ```ini
    inet_protocols = ipv4
    ```
*   **Line 183:** Comment out the short destination line:
    ```ini
    #mydestination = $myhostname, localhost.$mydomain, localhost
    ```
*   **Line 184:** Uncomment the long destination line:
    ```ini
    mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
    ```
*   **Line 283:** Uncomment and set your network:
    ```ini
    mynetworks = 172.16.8.0/24, 127.0.0.0/8
    ```
*   **Line 438:** Uncomment to use Maildir:
    ```ini
    home_mailbox = Maildir/
    ```
*   **Line 593:** Add the banner:
    ```ini
    smtpd_banner = $myhostname ESMTP
    ```

**3. Add SASL/Dovecot Auth Settings:**
Scroll to the very bottom of the file and paste this block to enable authentication:
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
*   **Line 30:** Uncomment/Set:
    ```ini
    listen = *
    ```

**3. Edit `/etc/dovecot/conf.d/10-auth.conf`:**
```bash
nano /etc/dovecot/conf.d/10-auth.conf
```
*   **Line 10:** Uncomment:
    ```ini
    disable_plaintext_auth = no
    ```
*   **Line 100:** Change to:
    ```ini
    auth_mechanisms = plain login
    ```

**4. Edit `/etc/dovecot/conf.d/10-mail.conf`:**
```bash
nano /etc/dovecot/conf.d/10-mail.conf
```
*   **Line 30:** Uncomment:
    ```ini
    mail_location = maildir:~/Maildir
    ```

**5. Edit `/etc/dovecot/conf.d/10-master.conf`:**
```bash
nano /etc/dovecot/conf.d/10-master.conf
```
Find `service auth` (approx line 95), then edit the `unix_listener` block (lines 107-109) to match this:
```ini
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
```

**6. Edit `/etc/dovecot/conf.d/10-ssl.conf`:**
```bash
nano /etc/dovecot/conf.d/10-ssl.conf
```
*   **Line 8:** Set:
    ```ini
    ssl = yes
    ```

### c) Start Services & Verification

**1. Configure Firewall:**
```bash
firewall-cmd --add-service={smtp,pop3,imap} --permanent
firewall-cmd --reload
```

**2. Start Services:**
```bash
systemctl restart postfix dovecot
systemctl enable postfix dovecot
```

**3. Check Status:**
```bash
# Verify services are running
systemctl status postfix dovecot
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

### a) Install and Configure

**1. Install Samba:**
```bash
dnf install samba samba-client -y
```

**2. Edit `/etc/samba/smb.conf`:**
```bash
nano /etc/samba/smb.conf
```
Append the following block to the end of the file:
```ini
[Team_Share]
    path = /home/TeamProjects
    browseable = yes
    writable = yes
    guest ok = yes
    read only = no
    create mask = 0777
    directory mask = 0777
```

### b) Setup Samba User Password
Although Linux users (`user1`) exist (from Task 5), they cannot access Samba until you add them to the Samba password database.

```bash
# Set a Samba password for user1 (You will be prompted to type it twice)
smbpasswd -a user1

# (Optional) Set one for root if you want admin access
smbpasswd -a root
```

### c) Fix Permissions & Security
To allow both Guests and Users to write files:

```bash
# 1. Set Permissions (Read/Write for Everyone)
chmod -R 777 /home/TeamProjects

# 2. Set SELinux Context
chcon -R -t samba_share_t /home/TeamProjects
```

### d) Start Services & Firewall

```bash
firewall-cmd --add-service=samba --permanent
firewall-cmd --reload

systemctl restart smb nmb
systemctl enable smb nmb
```

### e) Access Testing (Windows Client)

1.  Open File Explorer and type `\\172.16.8.8\Team_Share`.
2.  **If prompted for a Username/Password:**

    **Option A: Login as Guest**
    *   **Username:** `Guest`
    *   **Password:** (Leave Blank)

    **Option B: Login as User**
    *   **Username:** `user1`
    *   **Password:** (The password you set in step 'b' using `smbpasswd`)
