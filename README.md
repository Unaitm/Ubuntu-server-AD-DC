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
  - [Kerberos Authentication Test](#4-kerberos-authentication-test)
  - [Windows Client — Join Domain](#5-windows-client--join-domain)
  - [Organizational Units (OUs)](#6-organizational-units-ous)
  - [Create Users inside OUs](#7-create-users-inside-ous)
  - [Password Policy (GPO)](#8-password-policy-gpo)
  - [Verify Password Policy](#9-verify-password-policy)
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
  - [Kerberos Cross-Domain Authentication Test](#5-kerberos-cross-domain-authentication-test)

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

<img width="593" height="268" alt="image" src="https://github.com/user-attachments/assets/23da8d89-555a-4dc8-b4fa-a2928c18f03e" />

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

<img width="476" height="407" alt="image" src="https://github.com/user-attachments/assets/152bb1bf-64e5-434c-bb01-0cce67ad862e" />

---

### 3. Interface Verification

```bash
ip a
```

Expected output confirms both interfaces are up:

- `enp0s3` → `172.30.20.74/25`
- `enp0s8` → `10.2.11.254/24`

<img width="601" height="266" alt="image" src="https://github.com/user-attachments/assets/e0ae09c8-5c13-4d1c-9467-063193dea5e4" />

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

<img width="480" height="213" alt="image" src="https://github.com/user-attachments/assets/1b3e04d1-a7e8-4cf0-8dcb-8c7b3dca034e" />

---

### 5. Samba AD DC Installation

Install all required packages for Samba Active Directory Domain Controller:

```bash
sudo apt install samba krb5-user smbclient winbind dnsutils -y
```

<img width="598" height="25" alt="image" src="https://github.com/user-attachments/assets/eb8ecc98-9289-425b-aefa-4c28bc95fb2c" />

Then stop the Samba service and back up the default configuration file before provisioning:

```bash
sudo systemctl stop samba-ad-dc 2>/dev/null || true
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

<img width="525" height="59" alt="image" src="https://github.com/user-attachments/assets/efd6dd82-986e-4c94-91f0-1b9c72d03b7e" />

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

<img width="605" height="39" alt="image" src="https://github.com/user-attachments/assets/783318bc-673f-491a-a65d-146f59ebacce" />

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

<img width="432" height="148" alt="image" src="https://github.com/user-attachments/assets/7ee86888-a0c1-44d9-b17a-3d180cdbe76b" />

**Connectivity test from the Linux client** — ping the domain, FQDN, and server IP:

```bash
ping lab11.lan
ping ls11.lab11.lan
ping 172.30.20.74
```

All three should return 0% packet loss.

<img width="599" height="482" alt="image" src="https://github.com/user-attachments/assets/09769593-5aae-40db-bd0f-9fb810a3827d" />

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

<img width="596" height="404" alt="image" src="https://github.com/user-attachments/assets/c15194fc-ba4a-4a07-9fab-f7ac99ea93ae" />

---

### 9. Linux Client — Join Domain

Install required packages on the client:

```bash
sudo apt install sssd-ad sssd-tools realmd adcli packagekit -y
```

<img width="595" height="23" alt="image" src="https://github.com/user-attachments/assets/ce9243c4-6eaf-4655-bf91-69c5e9b1def9" />

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

<img width="1174" height="687" alt="image" src="https://github.com/user-attachments/assets/3e6beec5-41a0-453b-9e2b-4b54115589b4" />
<img width="1173" height="208" alt="image" src="https://github.com/user-attachments/assets/ec77c9b5-fec6-48e9-ac7d-71392fd3db62" />

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

<img width="675" height="599" alt="image" src="https://github.com/user-attachments/assets/1ed28943-9c81-4fa7-9e79-f2ae880cb1c6" />

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

<img width="664" height="164" alt="image" src="https://github.com/user-attachments/assets/683878ce-32d3-4034-8724-01b6eaf38c24" />

---

## Sprint 2 — Users, Groups, OUs & GPOs

### 1. Create Domain Users

```bash
sudo samba-tool user create Alice admin_21
sudo samba-tool user create Bob admin_21
sudo samba-tool user create Charlie admin_21
```

<img width="904" height="243" alt="image" src="https://github.com/user-attachments/assets/c1ce5df4-d9d1-49dd-840f-45cf7c4cea19" />

---

### 2. Create Security Groups

```bash
sudo samba-tool group add IT_Admins
sudo samba-tool group add Students
```

<img width="775" height="131" alt="image" src="https://github.com/user-attachments/assets/071731b7-1e9a-45d9-894b-e1dcffcf07ff" />

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

<img width="978" height="375" alt="image" src="https://github.com/user-attachments/assets/4f1a9932-91e6-4d22-bde5-ba0b3e7bd2c2" />

---

### 4. Kerberos Authentication Test

After creating domain users, verify that Kerberos authentication is working correctly by obtaining and inspecting a ticket for one of the new users.

**Request a Kerberos Ticket Granting Ticket (TGT):**

```bash
kinit Alice@LAB11.LAN
```

You will be prompted for the user's password. A warning will indicate when the password is set to expire:

```
Password for Alice@LAB11.LAN:
Warning: Your password will expire in 41 days on ...
```

**List the active Kerberos tickets:**

```bash
klist
```

```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: Alice@LAB11.LAN

Valid starting       Expires              Service principal
22/02/26 21:29:16   23/02/26 07:29:16   krbtgt/LAB11.LAN@LAB11.LAN
        renew until 23/02/26 21:29:10
```

The output confirms:
- The ticket cache is stored at `/tmp/krb5cc_1000`
- The default principal is `Alice@LAB11.LAN`
- The TGT is valid for 10 hours and renewable for 24 hours
- The service principal `krbtgt/LAB11.LAN@LAB11.LAN` confirms the domain controller issued the ticket correctly

**Destroy the ticket when done:**

```bash
kdestroy
```

This removes all cached Kerberos credentials, which is good practice after testing.

<img width="722" height="175" alt="image" src="https://github.com/user-attachments/assets/YOUR-KERBEROS-SCREENSHOT-ID" />

> ✅ A successfully issued TGT confirms that Kerberos is fully operational on the domain, the user account exists and is active, and the KDC (Key Distribution Center) on the Samba DC is responding correctly.

---

### 5. Windows Client — Join Domain

**Step 1 — Configure network** on the Windows machine:

| Setting | Value |
|---|---|
| IP Address | `172.30.20.72` |
| Subnet mask | `255.255.255.128` |
| Default gateway | `172.30.20.1` |
| Preferred DNS | `172.30.20.74` *(Samba DC)* |
| Alternate DNS | `10.239.3.7` |

<img width="568" height="607" alt="image" src="https://github.com/user-attachments/assets/888b8c99-f39f-448b-afcc-490d2cf8a101" />

**Step 2 — Join domain** via *System Properties → Change → Domain*:

| Field | Value |
|---|---|
| Username | `ADMINISTRATOR` |
| Password | *(admin password)* |
| Domain | `LAB11.LAN` |

<img width="1032" height="373" alt="image" src="https://github.com/user-attachments/assets/a3e4a5a6-312e-4468-bc5e-7a0ae4c6cbdb" />

**Step 3 — Set computer name and confirm domain user:**

| Field | Value |
|---|---|
| Computer name | `UNAIWIN` |
| Domain | `LAB11.LAN` |
| Domain user to add | `Administrator` |

<img width="603" height="797" alt="image" src="https://github.com/user-attachments/assets/6143b975-bf27-4ea1-9234-15c53aa2a39f" />

**Step 4 — Login confirmation** — Windows login screen shows `LAB11.LAN\Administrator`.

<img width="964" height="716" alt="image" src="https://github.com/user-attachments/assets/ec708729-77a1-4737-997e-da1b07c9a33f" />

---

### 6. Organizational Units (OUs)

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

<img width="804" height="389" alt="image" src="https://github.com/user-attachments/assets/4e105a5d-d60f-4add-a4de-1f61f9fd3a80" />

Import the OUs into Active Directory:

```bash
sudo ldbadd -H /var/lib/samba/private/sam.ldb mis_ous.ldif
```

<img width="922" height="39" alt="image" src="https://github.com/user-attachments/assets/1e44cea3-d501-4a48-8ee2-d4dc11bb2d4a" />

---

### 7. Create Users inside OUs

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

<img width="972" height="452" alt="image" src="https://github.com/user-attachments/assets/1d2a2c5b-69d3-451c-8712-fbd78b387b4d" />

Re-add users to their groups:

```bash
sudo samba-tool group addmembers IT_Admins Alice
sudo samba-tool group addmembers Students Bob
sudo samba-tool group addmembers Students Charlie
```

<img width="822" height="175" alt="image" src="https://github.com/user-attachments/assets/1c92ee8b-53e0-4962-90b2-78ab05ce6d2e" />

---

### 8. Password Policy (GPO)

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

<img width="962" height="115" alt="image" src="https://github.com/user-attachments/assets/015106ab-aef1-4c15-ad6d-5f298781fbe9" />
<img width="962" height="104" alt="image" src="https://github.com/user-attachments/assets/22d4e2dc-4573-44c6-816c-5eb99d397e39" />
<img width="964" height="106" alt="image" src="https://github.com/user-attachments/assets/e79faafa-4a49-4d05-af53-d6d4136b6643" />

---

### 9. Verify Password Policy

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

<img width="745" height="359" alt="image" src="https://github.com/user-attachments/assets/7addaaa5-3ce9-48ea-bd47-8d621297913a" />

**Client-side verification** — After 3 failed login attempts, the account is locked and the Windows login screen displays: *"Your account has been disabled. Please contact your system administrator."*

<img width="906" height="737" alt="image" src="https://github.com/user-attachments/assets/0d3d928f-7483-419b-9ba8-4ad00e035ff2" />

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

<img width="640" height="263" alt="image" src="https://github.com/user-attachments/assets/43819010-b98f-4c70-b192-24b89122d569" />

---

### 2. Shared Folder Structure

Create the directory structure on the server:

```bash
sudo mkdir -p /srv/samba/FinanceDocs
sudo mkdir -p /srv/samba/HRDocs
sudo mkdir -p /srv/samba/Public
```

<img width="651" height="86" alt="image" src="https://github.com/user-attachments/assets/c694a822-4be9-4b6a-b319-8705433a13e6" />

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

<img width="454" height="543" alt="image" src="https://github.com/user-attachments/assets/0b314236-4075-42e5-8258-49f81f1527f3" />

Apply the changes:

```bash
sudo systemctl restart samba-ad-dc
```

<img width="632" height="31" alt="image" src="https://github.com/user-attachments/assets/803baab1-f74d-4e53-8202-d81a7de36cc5" />

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

<img width="949" height="302" alt="image" src="https://github.com/user-attachments/assets/0429d63b-1ac5-4034-b5ac-db2b8efe03db" />

Restart Samba:

```bash
sudo systemctl restart samba-ad-dc
```

<img width="621" height="33" alt="image" src="https://github.com/user-attachments/assets/dbbd2d36-690c-47bc-b257-048a0f279f5e" />

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

<img width="874" height="53" alt="image" src="https://github.com/user-attachments/assets/0555281f-823e-45ab-877e-75202ce2e711" />
<img width="672" height="87" alt="image" src="https://github.com/user-attachments/assets/ae9e8bfd-b00e-4e49-b9b0-57c95560e3b7" />

---

### 6. Disk Management — Add a 10 GB Disk

A new 10 GB virtual disk was added to the server via VirtualBox storage settings.

<img width="736" height="174" alt="image" src="https://github.com/user-attachments/assets/331073e8-6665-4a49-9cb6-3f1e6be531f7" />

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

<img width="893" height="415" alt="image" src="https://github.com/user-attachments/assets/bfadb8c3-e740-4fcc-83f6-d909209a5da2" />

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

<img width="963" height="694" alt="image" src="https://github.com/user-attachments/assets/ffeb84ca-a740-4fcd-a9b1-6c44b5c1e999" />

**Format as ext4:**

```bash
sudo mkfs.ext4 /dev/sdb1
```

```
Filesystem UUID: 896984e1-6d2a-49c9-b8e5-4c1694c18d5d
```

<img width="903" height="322" alt="image" src="https://github.com/user-attachments/assets/9f97b7f2-9a8e-4692-a74e-8c73eb5bab63" />

**Create mount point and mount:**

```bash
sudo mkdir /mnt/datadrive
sudo mount /dev/sdb1 /mnt/datadrive
```

<img width="629" height="57" alt="image" src="https://github.com/user-attachments/assets/5642ff7c-04a3-4508-bdf4-9dd5d5183ad1" />

---

### 8. Persistent Mount via fstab

Find the UUID of the new partition:

```bash
sudo blkid /dev/sdb1
```

```
/dev/sdb1: UUID="896984e1-6d2a-49c9-b8e5-4c1694c18d5d" BLOCK_SIZE="4096" TYPE="ext4"
```

<img width="967" height="105" alt="image" src="https://github.com/user-attachments/assets/08b34d4d-0474-459c-bdcf-d4d9e6d6d0e2" />

Add the entry to `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Append this line at the end:

```
UUID=896984e1-6d2a-49c9-b8e5-4c1694c18d5d /mnt/datadrive ext4 defaults 0 0
```

<img width="868" height="232" alt="image" src="https://github.com/user-attachments/assets/2e7a1f41-b790-4126-bc78-8e592578e408" />

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

<img width="522" height="783" alt="image" src="https://github.com/user-attachments/assets/f38d8a9d-bca1-40b8-b1e2-ebc04b49e612" />

Restart Samba to apply:

```bash
sudo systemctl restart samba-ad-dc
```

<img width="863" height="54" alt="image" src="https://github.com/user-attachments/assets/8b51acbb-a0b5-428a-935b-8797b1534112" />

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

<img width="893" height="377" alt="image" src="https://github.com/user-attachments/assets/28dc3228-bd0c-4249-a17d-566d7accc5e6" />

**Make the script executable:**

```bash
chmod +x /home/unai/backup_finanzas.sh
```

<img width="1094" height="51" alt="image" src="https://github.com/user-attachments/assets/9c25035e-1da6-43ff-85f9-d8eebe26f24b" />

**Schedule with cron** — run every day at 22:00:

```bash
crontab -e
```

Add this line:

```cron
00 22 * * * /home/unai/backup_finanzas.sh
```

<img width="1109" height="176" alt="image" src="https://github.com/user-attachments/assets/c4b993b4-83b8-46b5-b8f2-6c38dddfb699" />

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

<img width="1041" height="440" alt="image" src="https://github.com/user-attachments/assets/5f75b100-cfeb-4166-971f-e4650fa4fdb8" />

Configure DNS resolution to point to the first domain controller:

```bash
sudo unlink /etc/resolv.conf
sudo nano /etc/resolv.conf
```

```
nameserver 172.30.20.74
search lab11.lan
```

<img width="678" height="203" alt="image" src="https://github.com/user-attachments/assets/051a936e-0455-41fc-90f9-575f22e79945" />

**Install required packages:**

```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  smbclient winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config \
  krb5-user dnsutils chrony net-tools
```

During installation, set the default Kerberos realm to `LAB11.LAN`.

<img width="1099" height="458" alt="image" src="https://github.com/user-attachments/assets/8c10a7e0-5d01-4009-8ecd-5d7860063f0c" />

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

<img width="1107" height="65" alt="image" src="https://github.com/user-attachments/assets/c478e172-b11a-4921-a7dd-7c252b1cefba" />

Disable individual Samba services and enable the AD DC service:

```bash
sudo systemctl disable --now smbd nmbd winbind
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
```

<img width="1108" height="373" alt="image" src="https://github.com/user-attachments/assets/5b0f9a10-83b9-4891-828e-2cf8870a19af" />

Update `/etc/resolv.conf` to use the local DNS:

```bash
sudo nano /etc/resolv.conf
```

```
nameserver 127.0.0.1
search school.lan
```

<img width="765" height="107" alt="image" src="https://github.com/user-attachments/assets/2c6b8e59-50a0-4297-9761-eff4131614df" />

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

<img width="817" height="304" alt="image" src="https://github.com/user-attachments/assets/5bdc777c-4021-4996-a026-a568a0152c37" />

---

### 3. DNS Cross-Resolution

For the two domains to resolve each other, DNS zones and records must be added on both servers.

**On the second server (`school.lan`)** — add a DNS zone and NS record pointing to the first server:

```bash
sudo samba-tool dns zonecreate 127.0.0.1 lab11.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 lab11.lan @ NS ls11.lab11.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 lab11.lan ls11 A 172.30.20.74 -U Administrator
```

<img width="1109" height="147" alt="image" src="https://github.com/user-attachments/assets/a4bf53e3-3976-48b3-b30c-054182ab284c" />

**On the first server (`lab11.lan`)** — add a zone and NS record pointing to the second server:

```bash
sudo samba-tool dns zonecreate 127.0.0.1 school.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 school.lan @ NS lab11tr.school.lan -U Administrator
sudo samba-tool dns add 127.0.0.1 school.lan lab11tr A 172.30.20.72 -U Administrator
```

<img width="1109" height="150" alt="image" src="https://github.com/user-attachments/assets/7b50cc1c-c648-4a08-aa9a-ffec04166703" />

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

---

### 4. Troubleshoot Ping Between Domains

**Problem:** Initially, pings between the two domains failed in both directions.

```
# From ls11:
ping school.lan → No address associated with hostname

# From lab11tr:
ping lab11.lan → Temporary failure in name resolution
```

<img width="1252" height="128" alt="image" src="https://github.com/user-attachments/assets/c02c7756-12f5-445f-bb2e-91ba4cb6fb92" />

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

<img width="915" height="370" alt="image" src="https://github.com/user-attachments/assets/0850e8a3-a75f-4747-9f0c-dab2eceafbe4" />

Restart Samba:

```bash
sudo systemctl restart samba-ad-dc
```

<img width="720" height="37" alt="image" src="https://github.com/user-attachments/assets/b8d746ca-5d0d-498d-85bc-b64d94b9171d" />

After this change, pings between the two domains resolve successfully, which is the prerequisite for establishing a formal forest trust relationship.

---

### 5. Kerberos Cross-Domain Authentication Test

With DNS resolution working between both domains, the final verification is to confirm that a user can obtain a Kerberos ticket from the second domain (`SCHOOL.LAN`), proving the trust is fully operational at the authentication level.

**Request a TGT for a user in the second domain:**

```bash
kinit Administrator@SCHOOL.LAN
```

You will be prompted for the password. A warning about password expiry is expected and normal:

```
Password for Administrator@SCHOOL.LAN:
Warning: Your password will expire in 41 days on ...
```

**List the active Kerberos tickets:**

```bash
klist
```

```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: Administrator@SCHOOL.LAN

Valid starting       Expires              Service principal
22/02/26 21:29:16   23/02/26 07:29:16   krbtgt/SCHOOL.LAN@SCHOOL.LAN
        renew until 23/02/26 21:29:10
```

The output confirms:
- The ticket cache is stored at `/tmp/krb5cc_1000`
- The default principal resolves to `Administrator@SCHOOL.LAN` — the second domain
- The TGT was issued by `krbtgt/SCHOOL.LAN@SCHOOL.LAN`, confirming the second DC's KDC is reachable and responding

**Destroy the ticket after testing:**

```bash
kdestroy
```

> ✅ A successfully issued TGT against `SCHOOL.LAN` from the `LAB11.LAN` server confirms that cross-domain name resolution, network connectivity, and Kerberos authentication are all working correctly — the foundation required for a formal Active Directory forest trust.

---

## Summary

| Sprint | Key Achievements |
|--------|-----------------|
| **Sprint 1** | Ubuntu Server installed, Samba AD DC provisioned for `lab11.lan`, Linux client joined the domain |
| **Sprint 2** | Domain users and groups created, Kerberos authentication verified, Windows client joined, OUs defined, password policy enforced |
| **Sprint 3** | Shared folders with group-based ACLs, 10 GB disk added and mounted persistently, automated backup via cron |
| **Sprint 4** | Second domain `school.lan` provisioned, cross-domain DNS configured, connectivity established, Kerberos cross-domain authentication verified |
