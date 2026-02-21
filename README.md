# Ubuntu-server-AD-DC
# Samba Active Directory Lab — Full Documentation

> **Environment:** Ubuntu Server 24 · Samba AD DC · Windows 10 client · Linux client  
> **Domain:** `lab11.lan` / `LAB11.LAN`  
> **Server hostname:** `ls11` · **IP:** `172.30.20.74/25`

---

## Table of Contents

- [Sprint 1 — Server Setup & Domain Provisioning](#sprint-1--server-setup--domain-provisioning)
  - [Profile Configuration](#1-profile-configuration)
  - [Network Configuration (Server)](#2-network-configuration-server)
  - [Interface Verification](#3-interface-verification)
  - [Hostname Configuration](#4-hostname-configuration)
  - [Samba AD DC Installation](#5-samba-ad-dc-installation)
  - [Domain Provisioning](#6-domain-provisioning)
  - [Domain Verification](#7-domain-verification)
  - [Linux Client — Network Configuration](#8-linux-client--network-configuration)
  - [Linux Client — Join Domain](#9-linux-client--join-domain)
  - [Verify Client Integration from Server](#10-verify-client-integration-from-server)
- [Sprint 2 — Users, Groups, OUs & GPOs](#sprint-2--users-groups-ous--gpos)
  - [Create Domain Users](#1-create-domain-users)
  - [Create Security Groups](#2-create-security-groups)
  - [Add Users to Groups](#3-add-users-to-groups)
  - [Windows Client — Join Domain](#4-windows-client--join-domain)
  - [Organizational Units (OUs)](#5-organizational-units-ous)
  - [Create Users inside OUs](#6-create-users-inside-ous)
  - [Password Policy (GPO)](#7-password-policy-gpo)
  - [Verify Password Policy](#8-verify-password-policy)
- [Sprint 3 — Shared Folders, Permissions & Disk Management](#sprint-3--shared-folders-permissions--disk-management)
  - [Create Groups and Users](#1-create-groups-and-users)
  - [Shared Folder Structure](#2-shared-folder-structure)
  - [Samba Share Configuration](#3-samba-share-configuration)
  - [NSSwitch & Winbind Integration](#4-nsswitch--winbind-integration)
  - [Set Ownership and Permissions](#5-set-ownership-and-permissions)
  - [Disk Management — Add a 10 GB Disk](#6-disk-management--add-a-10-gb-disk)
  - [Partition, Format and Mount](#7-partition-format-and-mount)
  - [Persistent Mount via fstab](#8-persistent-mount-via-fstab)
  - [Expose Backup Share in Samba](#9-expose-backup-share-in-samba)
  - [Automation — Backup Script with Cron](#10-automation--backup-script-with-cron)
- [Sprint 4 — Cross-Domain Trust](#sprint-4--cross-domain-trust)
  - [Second Server Setup](#1-second-server-setup)
  - [Provision Second Domain](#2-provision-second-domain)
  - [DNS Cross-Resolution](#3-dns-cross-resolution)
  - [Troubleshoot Ping Between Domains](#4-troubleshoot-ping-between-domains)

---

## Sprint 1 — Server Setup & Domain Provisioning

### 1. Profile Configuration

During the Ubuntu Server installation, the following profile was configured:

| Field | Value |
|---|---|
| Full name | `unai` |
| Server name | `ls11` |
| Username | `unai` |
| Password | *(set during install)* |

<!-- IMAGE: Ubuntu installer — profile configuration screen -->

---

### 2. Network Configuration (Server)

Two network interfaces were configured statically via Netplan.

```bash
sudo cat /etc/netplan/00-installer-config.yaml
```

```yaml
# This is the network config written by 'subiquity'
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      addresses:
        - 10.2.11.254/24
      nameservers:
        addresses:
          - 10.239.3.7
          - 10.239.3.8
      optional: true
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.74/25
      nameservers:
        addresses: [10.239.3.7, 10.239.3.8]
      routes:
        - to: default
          via: 172.30.20.1
      optional: true
```

<!-- IMAGE: Terminal output of the above netplan file -->

---

### 3. Interface Verification

```bash
ip a
```

Expected output confirms both interfaces are up:

- `enp0s3` → `172.30.20.74/25`
- `enp0s8` → `10.2.11.254/24`

<!-- IMAGE: ip a output showing both interfaces active -->

---

### 4. Hostname Configuration

```bash
cat /etc/hosts
```

```
127.0.0.1   localhost
127.0.0.1   ls11
```

```bash
cat /etc/hostname
```

```
ls11
```

<!-- IMAGE: /etc/hosts and /etc/hostname output -->

---

### 5. Samba AD DC Installation

Install all required packages for Samba Active Directory Domain Controller:

```bash
sudo apt install samba krb5-user smbclient winbind dnsutils -y
```

<!-- IMAGE: apt install command in terminal -->

Then stop the Samba service and back up the default configuration file before provisioning:

```bash
sudo systemctl stop samba-ad-dc 2>/dev/null || true
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

<!-- IMAGE: Terminal showing the backup commands -->

---

### 6. Domain Provisioning

Provision the Samba AD DC with the following parameters:

| Parameter | Value |
|---|---|
| Realm | `lab11.lan` |
| Domain (NetBIOS) | `LAB11` |
| Server role | `dc` |
| DNS backend | `SAMBA_INTERNAL` |
| Admin password | `admin_21` |
| Interfaces | `lo enp0s3` |

```bash
sudo samba-tool domain provision \
  --use-rfc2307 \
  --realm=lab11.lan \
  --domain=lab11 \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --adminpass='admin_21' \
  --option="interfaces=lo enp0s3" \
  --option="bind interfaces only=yes"
```

<!-- IMAGE: samba-tool domain provision command output -->

---

### 7. Domain Verification

Check that the domain was provisioned correctly:

```bash
sudo samba-tool domain info 172.30.20.74
```

Expected output:

```
Forest          : lab11.lan
Domain          : lab11.lan
Netbios domain  : LAB11
DC name         : ls11.lab11.lan
DC netbios name : LS11
Server site     : Default-First-Site-Name
Client site     : Default-First-Site-Name
```

<!-- IMAGE: samba-tool domain info output -->

**Connectivity test from the Linux client** — ping the domain, FQDN, and server IP:

```bash
ping lab11.lan
ping ls11.lab11.lan
ping 172.30.20.74
```

All three should return 0% packet loss.

<!-- IMAGE: Ping results from Linux client — all three successful -->

---

### 8. Linux Client — Network Configuration

The Linux client (`lc11`) was configured with a static IP on the same subnet, pointing to the Samba DC as its primary DNS server.

```bash
sudo cat /etc/netplan/01-network-manager-all.yaml
```

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.73/25
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses: [172.30.20.74, 10.239.3.7]
        search: [lab11.lan]
```

Verify active DNS servers:

```bash
resolvectl status | grep "DNS Server"
```

```
Current DNS Server: 10.239.3.7
       DNS Servers: 172.30.20.74 10.239.3.7
```

<!-- IMAGE: Client netplan config and resolvectl output -->

---

### 9. Linux Client — Join Domain

Install required packages on the client:

```bash
sudo apt install sssd-ad sssd-tools realmd adcli packagekit -y
```

<!-- IMAGE: apt install command on the client -->

Join the domain:

```bash
sudo realm join -v lab11.lan -U Administrator
```

The process performs the following steps automatically:
1. LDAP lookup to discover the domain controller
2. Package verification
3. Computer account creation in AD (`CN=LC11,CN=Computers,DC=lab11,DC=lan`)
4. Kerberos keytab generation at `/etc/krb5.keytab`
5. SSSD enablement and restart

<!-- IMAGE: Full realm join verbose output -->

Verify successful enrollment:

```bash
realm discover lab11.lan
```

```
lab11.lan
  type: kerberos
  realm-name: LAB11.LAN
  domain-name: lab11.lan
  configured: kerberos-member
  server-software: active-directory
  client-software: sssd
  login-formats: %U@lab11.lan
  login-policy: allow-realm-logins
```

<!-- IMAGE: realm discover output -->

---

### 10. Verify Client Integration from Server

From the domain controller, list all computer accounts to confirm both machines are registered:

```bash
sudo samba-tool computer list
```

```
LC11$
LS11$
```

<!-- IMAGE: samba-tool computer list output showing LC11$ and LS11$ -->

---

## Sprint 2 — Users, Groups, OUs & GPOs

### 1. Create Domain Users

```bash
sudo samba-tool user create Alice admin_21
sudo samba-tool user create Bob admin_21
sudo samba-tool user create Charlie admin_21
```

<!-- IMAGE: Terminal output — all three users created successfully -->

---

### 2. Create Security Groups

```bash
sudo samba-tool group add IT_Admins
sudo samba-tool group add Students
```

<!-- IMAGE: Group creation output -->

---

### 3. Add Users to Groups

```bash
# Alice → IT_Admins
sudo samba-tool group addmembers IT_Admins Alice

# Bob and Charlie → Students
sudo samba-tool group addmembers Students Bob
sudo samba-tool group addmembers Students Charlie
```

Verify membership:

```bash
sudo samba-tool group listmembers IT_Admins
# Output: Alice

sudo samba-tool group listmembers Students
# Output: Charlie, Bob
```

<!-- IMAGE: Group membership verification output -->

---

### 4. Windows Client — Join Domain

**Step 1 — Configure network** on the Windows machine:

| Setting | Value |
|---|---|
| IP Address | `172.30.20.72` |
| Subnet mask | `255.255.255.128` |
| Default gateway | `172.30.20.1` |
| Preferred DNS | `172.30.20.74` *(Samba DC)* |
| Alternate DNS | `10.239.3.7` |

<!-- IMAGE: Windows TCP/IPv4 properties dialog -->

**Step 2 — Join domain** via *System Properties → Change → Domain*:

| Field | Value |
|---|---|
| Username | `ADMINISTRATOR` |
| Password | *(admin password)* |
| Domain | `LAB11.LAN` |

<!-- IMAGE: Windows domain join credential dialog -->

**Step 3 — Set computer name and confirm domain user:**

| Field | Value |
|---|---|
| Computer name | `UNAIWIN` |
| Domain | `LAB11.LAN` |
| Domain user to add | `Administrator` |

<!-- IMAGE: Windows computer name dialog + domain user confirmation -->

**Step 4 — Login confirmation** — Windows login screen shows `LAB11.LAN\Administrator`.

<!-- IMAGE: Windows login screen showing LAB11.LAN\Administrator -->

---

### 5. Organizational Units (OUs)

Create an LDIF file to define three Organizational Units:

```bash
nano mis_ous.ldif
```

```ldif
dn: OU=IT_Department,DC=lab11,DC=lan
changetype: add
objectClass: top
objectClass: organizationalUnit

dn: OU=HR_Department,DC=lab11,DC=lan
changetype: add
objectClass: top
objectClass: organizationalUnit

dn: OU=Students_UO,DC=lab11,DC=lan
changetype: add
objectClass: top
objectClass: organizationalUnit
```

<!-- IMAGE: nano editor showing the LDIF file -->

Import the OUs into Active Directory:

```bash
sudo ldbadd -H /var/lib/samba/private/sam.ldb mis_ous.ldif
```

<!-- IMAGE: ldbadd command and confirmation output -->

---

### 6. Create Users inside OUs

> **Note:** The initial attempt failed because users already existed. They were deleted and re-created with the correct OU assignment.

```bash
# Delete existing users
sudo samba-tool user delete Alice
sudo samba-tool user delete Bob
sudo samba-tool user delete Charlie

# Re-create with OU assignment
sudo samba-tool user create Alice admin_21 --userou="OU=IT_Department"
sudo samba-tool user create Bob admin_21 --userou="OU=Students_UO"
sudo samba-tool user create Charlie admin_21 --userou="OU=Students_UO"
```

<!-- IMAGE: Terminal showing delete errors, deletion, and successful re-creation -->

Re-add users to their groups:

```bash
sudo samba-tool group addmembers IT_Admins Alice
sudo samba-tool group addmembers Students Bob
sudo samba-tool group addmembers Students Charlie
```

<!-- IMAGE: Group addmembers output -->

---

### 7. Password Policy (GPO)

Configure domain-wide password settings:

**Minimum password length — 8 characters:**

```bash
sudo samba-tool domain passwordsettings set --min-pwd-length=8
```

```
Minimum password length changed!
All changes applied successfully!
```

**Account lockout threshold — 3 attempts:**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-threshold=3
```

```
Account lockout threshold changed!
All changes applied successfully!
```

**Account lockout duration — 5 minutes:**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-duration=5
```

```
Account lockout duration changed!
All changes applied successfully!
```

<!-- IMAGE: All three passwordsettings commands and their output -->

---

### 8. Verify Password Policy

```bash
sudo samba-tool domain passwordsettings show
```

```
Password information for domain 'DC=lab11,DC=lan'

Password complexity:                    on
Store plaintext passwords:              off
Password history length:                24
Minimum password length:                8
Minimum password age (days):            1
Maximum password age (days):            42
Account lockout duration (mins):        5
Account lockout threshold (attempts):   3
Reset account lockout after (mins):     30
```

<!-- IMAGE: passwordsettings show output -->

**Client-side verification** — After 3 failed login attempts, the account is locked and the Windows login screen displays: *"Your account has been disabled. Please contact your system administrator."*

<!-- IMAGE: Windows login screen showing account disabled message -->

---

## Sprint 3 — Shared Folders, Permissions & Disk Management

### 1. Create Groups and Users

**Create groups:**

```bash
sudo samba-tool group add Finance
sudo samba-tool group add HR
sudo samba-tool group add IT_Support
```

**Create users:**

```bash
sudo samba-tool user create user01 admin_21
sudo samba-tool user create user02 admin_21
sudo samba-tool user create user03 admin_21
sudo samba-tool user create techsupport admin_21
```

**Assign users to groups:**

```bash
sudo samba-tool group addmembers Finance user01
sudo samba-tool group addmembers HR user02
sudo samba-tool group addmembers IT_Support techsupport
```

<!-- IMAGE: Group and user creation output -->

---

### 2. Shared Folder Structure

Create the directory structure on the server:

```bash
sudo mkdir -p /srv/samba/FinanceDocs
sudo mkdir -p /srv/samba/HRDocs
sudo mkdir -p /srv/samba/Public
```

<!-- IMAGE: mkdir commands -->

---

### 3. Samba Share Configuration

Edit `/etc/samba/smb.conf` and add share definitions at the end of the file:

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[global]
        bind interfaces only = Yes
        dns forwarder = 8.8.8.8
        interfaces = lo enp0s3
        netbios name = LS11
        realm = LAB11.LAN
        server role = active directory domain controller
        workgroup = LAB11
        idmap_ldb:use rfc2307 = yes

[sysvol]
        path = /var/lib/samba/sysvol
        read only = No

[netlogon]
        path = /var/lib/samba/sysvol/lab11.lan/scripts
        read only = No

[FinanceDocs]
        path = /srv/samba/FinanceDocs
        read only = no
        valid users = @Finance @IT_Admins
        force group = Finance

[HRDocs]
        path = /srv/samba/HRDocs
        read only = no
        valid users = @HR @IT_Admins
        force group = HR

[Public]
        path = /srv/samba/Public
        read only = no
        guest ok = yes
```

<!-- IMAGE: nano editor showing smb.conf with all shares -->

Apply the changes:

```bash
sudo systemctl restart samba-ad-dc
```

<!-- IMAGE: systemctl restart command -->

---

### 4. NSSwitch & Winbind Integration

Install Winbind PAM libraries to allow resolution of AD users/groups at the OS level:

```bash
sudo apt install libnss-winbind libpam-winbind -y
```

Edit `/etc/nsswitch.conf` and append `winbind` to the `passwd` and `group` lines:

```bash
sudo nano /etc/nsswitch.conf
```

```
passwd:     files systemd winbind
group:      files systemd winbind
shadow:     files
gshadow:    files
```

<!-- IMAGE: nano editor showing /etc/nsswitch.conf with winbind added -->

Restart Samba:

```bash
sudo systemctl restart samba-ad-dc
```

<!-- IMAGE: systemctl restart command -->

---

### 5. Set Ownership and Permissions

Set group ownership of each shared folder to the corresponding AD group:

```bash
sudo chown root:"LAB11\Finance" /srv/samba/FinanceDocs
sudo chown root:"LAB11\HR" /srv/samba/HRDocs
```

Set permissions so only group members can read/write, and others have no access:

```bash
sudo chmod 2770 /srv/samba/FinanceDocs   # group rwx, sticky bit, others: none
sudo chmod 2770 /srv/samba/HRDocs
sudo chmod 2777 /srv/samba/Public        # Public: everyone can read/write
```

<!-- IMAGE: chown and chmod commands -->

---

### 6. Disk Management — Add a 10 GB Disk

A new 10 GB virtual disk was added to the server via VirtualBox storage settings.

<!-- IMAGE: VirtualBox storage panel showing sda (40GB) and new sdb (10GB) -->

Confirm the disk is detected:

```bash
lsblk
```

```
NAME                      MAJ:MIN  SIZE  TYPE  MOUNTPOINTS
sda                         8:0    40G   disk
├─sda1                      8:1     1M   part
├─sda2                      8:2     2G   part  /boot
└─sda3                      8:3    38G   part
  └─ubuntu--vg-ubuntu--lv  253:0   19G   lvm   /
sdb                         8:16   10G   disk
sr0                        11:0  1024M   rom
```

<!-- IMAGE: lsblk output showing sdb as unpartitioned 10GB disk -->

---

### 7. Partition, Format and Mount

**Create a partition** with `fdisk`:

```bash
sudo fdisk /dev/sdb
```

Steps inside fdisk:
1. `n` — new partition
2. `p` — primary
3. `1` — partition number
4. Accept default first and last sector (use entire disk)
5. `w` — write and exit

```
Created a new partition 1 of type 'Linux' and of size 10 GiB.
```

<!-- IMAGE: fdisk interactive session output -->

**Format as ext4:**

```bash
sudo mkfs.ext4 /dev/sdb1
```

```
Filesystem UUID: 896984e1-6d2a-49c9-b8e5-4c1694c18d5d
```

<!-- IMAGE: mkfs.ext4 output -->

**Create mount point and mount:**

```bash
sudo mkdir /mnt/datadrive
sudo mount /dev/sdb1 /mnt/datadrive
```

<!-- IMAGE: mkdir and mount commands -->

---

### 8. Persistent Mount via fstab

Find the UUID of the new partition:

```bash
sudo blkid /dev/sdb1
```

```
/dev/sdb1: UUID="896984e1-6d2a-49c9-b8e5-4c1694c18d5d" BLOCK_SIZE="4096" TYPE="ext4"
```

<!-- IMAGE: blkid output -->

Add the entry to `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Append this line at the end:

```
UUID=896984e1-6d2a-49c9-b8e5-4c1694c18d5d /mnt/datadrive ext4 defaults 0 0
```

<!-- IMAGE: nano editor showing /etc/fstab with the new UUID entry -->

---

### 9. Expose Backup Share in Samba

Create the Backups folder on the new disk and set open permissions:

```bash
sudo mkdir /mnt/datadrive/Backups
sudo chmod 777 /mnt/datadrive/Backups
```

Add the share to `/etc/samba/smb.conf`:

```ini
[Backups]
    path = /mnt/datadrive/Backups
    read only = no
    guest ok = yes
```

<!-- IMAGE: Final smb.conf showing all shares including [Backups] -->

Restart Samba to apply:

```bash
sudo systemctl restart samba-ad-dc
```

<!-- IMAGE: systemctl restart command -->

---

### 10. Automation — Backup Script with Cron

**Create the backup script:**

```bash
nano /home/unai/backup_finanzas.sh
```

```bash
#!/bin/bash

ORIGEN="/srv/samba/FinanceDocs"
DESTINO="/mnt/datadrive/Backups"
FECHA=$(date +%Y-%m-%d)
ARCHIVO="backup_finance_$FECHA.tar.gz"

mkdir -p $DESTINO

tar -czf "$DESTINO/$ARCHIVO" "$ORIGEN"

echo "Backup completed on $(date)" >> /var/log/backup_samba.log
```

<!-- IMAGE: nano editor showing the backup script -->

**Make the script executable:**

```bash
chmod +x /home/unai/backup_finanzas.sh
```

<!-- IMAGE: chmod command -->

**Schedule with cron** — run every day at 22:00:

```bash
crontab -e
```

Add this line:

```cron
00 22 * * * /home/unai/backup_finanzas.sh
```

<!-- IMAGE: crontab file showing the scheduled job -->

---

## Sprint 4 — Cross-Domain Trust

### 1. Second Server Setup

A new Ubuntu Server was provisioned to host a second Samba AD domain (`school.lan`).

**Configure the network** on the new server (`lab11tr`, IP `172.30.20.72`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.72/25
      nameservers:
        addresses: [127.0.0.1, 10.239.3.7]
      routes:
        - to: default
          via: 172.30.20.1
```

<!-- IMAGE: netplan config on the second server -->

Configure DNS resolution to point to the first domain controller:

```bash
sudo unlink /etc/resolv.conf
sudo nano /etc/resolv.conf
```

```
nameserver 172.30.20.74
search lab11.lan
```

<!-- IMAGE: resolv.conf on second server -->

**Install required packages:**

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config \
  krb5-user dnsutils chrony net-tools
```

During installation, set the default Kerberos realm to `LAB11.LAN`.

<!-- IMAGE: Kerberos realm configuration dialog showing LAB11.LAN -->

---

### 2. Provision Second Domain

Provision the new domain `school.lan`:

```bash
sudo samba-tool domain provision \
  --realm=school.lan \
  --domain=SCHOOL \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --adminpass='admin_21'
```

<!-- IMAGE: samba-tool domain provision command on second server -->

Disable individual Samba services and enable the AD DC service:

```bash
sudo systemctl disable --now smbd nmbd winbind
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
```

<!-- IMAGE: systemctl disable and enable commands -->

Update `/etc/resolv.conf` to use the local DNS:

```bash
sudo nano /etc/resolv.conf
```

```
nameserver 127.0.0.1
search school.lan
```

<!-- IMAGE: resolv.conf on second server pointing to 127.0.0.1 -->

Verify the second domain info:

```bash
sudo samba-tool domain info 127.0.0.1
```

```
Forest          : school.lan
Domain          : school.lan
Netbios domain  : SCHOOL
DC name         : lab11tr.school.lan
DC netbios name : LAB11TR
Server site     : Default-First-Site-Name
Client site     : Default-First-Site-Name
```

<!-- IMAGE: samba-tool domain info output for school.lan -->

---

### 3. DNS Cross-Resolution

For the two domains to resolve each other, DNS zones and records must be added on both servers.

**On the second server (`school.lan`)** — add a DNS zone and NS record pointing to the first server:

```bash
sudo samba-tool dns zonecreate 127.0.0.1 lab11.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 lab11.lan @ NS ls11.lab11.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 lab11.lan ls11 A 172.30.20.74 -U Administrator
```

<!-- IMAGE: DNS zone creation and record addition on second server -->

**On the first server (`lab11.lan`)** — add a zone and NS record pointing to the second server:

```bash
sudo samba-tool dns zonecreate 127.0.0.1 school.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 school.lan @ NS lab11tr.school.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 school.lan lab11tr A 172.30.20.72 -U Administrator
```

<!-- IMAGE: DNS zone creation and record addition on first server -->

**Verify connectivity** — ping the second server's FQDN from the first server:

```bash
ping lab11tr.school.lan
```

```
PING lab11tr.school.lan (172.30.20.72) 56(84) bytes of data.
64 bytes from 172.30.20.72: icmp_seq=1 ttl=64 time=1.15 ms
--- lab11tr.school.lan ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

<!-- IMAGE: Ping from ls11 to lab11tr.school.lan — 0% packet loss -->

---

### 4. Troubleshoot Ping Between Domains

**Problem:** Initially, pings between the two domains failed in both directions.

```
# From ls11:
ping school.lan → No address associated with hostname

# From lab11tr:
ping lab11.lan → Temporary failure in name resolution
```

<!-- IMAGE: Side-by-side terminals showing ping failures -->

**Solution:** Change the `dns forwarder` in `/etc/samba/smb.conf` on the first server to point to the second server's IP, so it can forward unknown domain queries:

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[global]
        ...
        dns forwarder = 172.30.20.72   # ← changed from 8.8.8.8
        ...
```

<!-- IMAGE: smb.conf with dns forwarder changed to 172.30.20.72 -->

Restart Samba:

```bash
sudo systemctl restart samba-ad-dc
```

<!-- IMAGE: systemctl restart on first server -->

After this change, pings between the two domains resolve successfully, which is the prerequisite for establishing a formal forest trust relationship.

---

## Summary

| Sprint | Key Achievements |
|--------|-----------------|
| **Sprint 1** | Ubuntu Server installed, Samba AD DC provisioned for `lab11.lan`, Linux client joined the domain |
| **Sprint 2** | Domain users and groups created, Windows client joined, OUs defined, password policy enforced |
| **Sprint 3** | Shared folders with group-based ACLs, 10 GB disk added and mounted persistently, automated backup via cron |
| **Sprint 4** | Second domain `school.lan` provisioned, cross-domain DNS configured, connectivity between domains established |
