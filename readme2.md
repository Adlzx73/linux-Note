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

Configure BIND using ACLs and specific Zone file structures.

### a) Install and Configure BIND

**1. Install Packages:**
```bash
dnf -y install bind bind-utils
rpm -q bind
rpm -q bind-utils
```

**2. Configure `/etc/named.conf`:**
```bash
nano /etc/named.conf
```
Clear the existing file (or carefully edit) to match this structure exactly:

```named
# Define access control list for local network
acl internal-network {
    172.16.8.0/24;
};

options {
    listen-on port 53 { any; };
    listen-on-v6 { any; };
    
    allow-query { localhost; internal-network; };
    
    allow-transfer { localhost; };
    
    # Path to zone files
    directory "/var/named";
};

# Add zones for your network and domain
# SN = 08 (Station Number)

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

# Standard includes
include "/etc/named.rfc1912.zones";
```

**3. Configure IPv4 Mode Only:**
```bash
nano /etc/sysconfig/named
```
Add this to the end of the file:
```ini
OPTIONS="-4"
```

### b) Create Zone Files

**1. Create Forward Zone:**
```bash
nano /var/named/unitechlab.net.fwd
```
Content:
```dns
$TTL 86400
@ IN SOA sysadmin08.unitechlab.net. root.sysadmin08.unitechlab.net. (
    2025102801 ;Serial
    3600       ;Refresh
    1800       ;Retry
    604800     ;Expire
    86400      ;Minimum TTL
)

# Define Name Server
@   IN  NS  sysadmin08.unitechlab.net.

# Define Name Server IP
@   IN  A   172.16.8.8

# Define Mail Exchanger (MX)
@   IN  MX 10 sysadmin08.unitechlab.net.

# Define Hosts
sysadmin08  IN  A   172.16.8.8
www         IN  A   172.16.8.8
```

**2. Create Reverse Zone:**
```bash
nano /var/named/unitechlab.net.rev
```
Content:
```dns
$TTL 86400
@ IN SOA sysadmin08.unitechlab.net. root.sysadmin08.unitechlab.net. (
    2025102801 ;Serial
    3600       ;Refresh
    1800       ;Retry
    604800     ;Expire
    86400      ;Minimum TTL
)

@   IN  NS  sysadmin08.unitechlab.net.

# PTR Record
8   IN  PTR sysadmin08.unitechlab.net.
```

### c) Start Services & Firewall

```bash
systemctl enable --now named

# Configure Firewall
firewall-cmd --add-service=dns
firewall-cmd --runtime-to-permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all
```

### d) Verify Resolution (Client Side)

**1. Change DNS setting to refer to own DNS (on the server itself for testing):**
*(Replace `ens160` with your interface name, e.g., `enp0s3`)*
```bash
nmcli connection modify enp0s3 ipv4.dns 172.16.8.8
nmcli connection down enp0s3; nmcli connection up enp0s3
```

**2. Verify Name Resolution:**
```bash
nslookup unitechlab.net
nslookup www.unitechlab.net
nslookup 172.16.8.8
```

---

## 2. DHCP Server

Configure DHCP using the `dynamic-bootp` parameter.

### a) Install and Configure

**1. Install DHCP:**
```bash
dnf -y install dhcp-server
```

**2. Prepare Configuration File:**
```bash
# Backup original
cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.ori

# Copy sample file
cp /usr/share/doc/dhcp-server/dhcpd.conf.example /etc/dhcp/dhcpd.conf
```

**3. Edit `/etc/dhcp/dhcpd.conf`:**
```bash
nano /etc/dhcp/dhcpd.conf
```
Modify the file to look like this:

```conf
# Specify domain name
option domain-name "unitechlab.net";
option domain-name-servers 172.16.8.8, 8.8.8.8;

# Default lease times
default-lease-time 600;
max-lease-time 7200;

# Declare authoritative
authoritative;

# Network configuration
subnet 172.16.8.0 netmask 255.255.255.0 {
    # Range of lease IP addresses
    range dynamic-bootp 172.16.8.200 172.16.8.250;
    
    # Broadcast address
    option broadcast-address 172.16.8.255;
    
    # Gateway
    option routers 172.16.8.1;
}
```

### b) Start Services & Firewall

```bash
systemctl enable --now dhcpd

# Configure Firewall
firewall-cmd --add-service=dhcp
firewall-cmd --runtime-to-permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all
```

### c) Client Configuration (Linux Client)

Run these commands on a **Linux Client PC** to force it to use your DHCP server.

**1. Install Client (if needed):**
```bash
dnf -y install dhcp-client
```

**2. Modify Interface to use DHCP:**
*(Replace `ens33` with client's actual interface name)*
```bash
nmcli connection modify ens33 ipv4.method auto
```

**3. Restart Network Interface:**
```bash
nmcli connection down ens33; nmcli connection up ens33
```

**4. Verify IP:**
```bash
ip addr
# Should show an IP between 172.16.8.200 and 172.16.8.250
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

### d) Show Mailbox Directory

The `Maildir/` directory is created automatically when an email is first delivered.

**1. Create a user to check:**
```bash
useradd mailuser
passwd mailuser
```

**2. Trigger directory creation (Send a test email):**
```bash
# Send a test email to localhost to force creation
echo "Hello" | mail -s "Test Email" mailuser@unitechlab.net
```
*(Note: If the `mail` command is not found, install it with `dnf install mailx -y`)*

**3. Verify the directory exists:**
```bash
ls -ld /home/mailuser/Maildir/
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

**Linux Client:**
```bash
# Connect as anonymous (hit Enter when asked for password)
smbclient //172.16.8.8/Team_Share -U %
```
