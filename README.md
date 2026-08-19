# Samba Setup Guide for Debian, Ubuntu & Fedora Linux

A guide to installing, configuring, and hardening Samba with modern protocol standards (SMB3, SMB signing, encryption), multi-OS support (Windows 10/11, macOS, Linux), WS-Discovery network browsing, and full SELinux support for Fedora / RHEL.

---

## Table of Contents

- [Supported Distributions](#supported-distributions)
- [Prerequisites](#prerequisites)
- [1. Installation](#1-installation)
- [2. Storage & Directory Setup](#2-storage--directory-setup)
- [3. SELinux Configuration (Fedora / RHEL)](#3-selinux-configuration-fedora--rhel)
- [4. Samba Configuration (`smb.conf`)](#4-samba-configuration-smbconf)
  - [Global Hardening & Modern Defaults](#global-hardening--modern-defaults)
  - [macOS Optimization (`vfs_fruit`)](#macos-optimization-vfs_fruit)
  - [Share Definitions](#share-definitions)
- [5. User Management](#5-user-management)
- [6. Network Discovery (WS-Discovery)](#6-network-discovery-ws-discovery)
- [7. Firewall Configuration](#7-firewall-configuration)
- [8. Service Management & Verification](#8-service-management--verification)
- [9. Client Connection & Mounting](#9-client-connection--mounting)
  - [Linux Client (Automount via `fstab`)](#linux-client-automount-via-fstab)
  - [Windows 10 / 11 Client](#windows-10--11-client)
  - [macOS Client](#macos-client)
- [10. Troubleshooting & Diagnostics](#10-troubleshooting--diagnostics)

---

## Supported Distributions

| Distribution | Versions Tested | Package Manager | Systemd Service Names |
| :--- | :--- | :--- | :--- |
| **Ubuntu** | 22.04 LTS / 24.04 LTS / 26.04 | `apt` | `smbd`, `nmbd`, `wsdd2` |
| **Debian** | 12 (Bookworm) / 13 (Trixie) | `apt` | `smbd`, `nmbd`, `wsdd2` |
| **Fedora** | 39 / 40 / 41+ (and RHEL 9/10 / Alma / Rocky) | `dnf` | `smb`, `nmb`, `wsdd` |

---

## Prerequisites

- Root or `sudo` administrative privileges.
- Static IP or DHCP reservation configured on the server.
- Basic network connectivity.

---

## 1. Installation

Install Samba, client tools for verification, and the WS-Discovery daemon (for modern Windows network browsing):

### Debian / Ubuntu

```bash
sudo apt update
sudo apt install samba smbclient cifs-utils wsdd2 -y
```

> [!NOTE]
> On Debian/Ubuntu, `wsdd2` provides the system daemon (`wsdd2.service`). The standalone `wsdd` package in Ubuntu is a client library and does not include a systemd service.

### Fedora Linux (and RHEL derivatives)

```bash
sudo dnf install samba samba-client samba-common-tools cifs-utils wsdd policycoreutils-python-utils -y
```

> [!NOTE]
> `policycoreutils-python-utils` provides `semanage` for managing SELinux file contexts on Fedora.

---

## 2. Storage & Directory Setup

Create dedicated directory structures for your shares with proper group ownership and SGID bit inheritance.

### Create Share Directory

```bash
sudo mkdir -p /srv/samba/shared
sudo mkdir -p /srv/samba/private
```

### Configure Linux User Group and Permissions

Create a dedicated system group for Samba users and set SGID permissions (ensures newly created files inherit the parent group):

```bash
# Create group
sudo groupadd sambashare

# Assign ownership to root:sambashare
sudo chown -R root:sambashare /srv/samba

# Apply SGID and permissions (rwxrws--- for private, rwxrwxr-x for shared)
sudo chmod -R 2775 /srv/samba/shared
sudo chmod -R 2770 /srv/samba/private
```

---

## 3. SELinux Configuration (Fedora / RHEL)

> [!IMPORTANT]
> On Fedora and RHEL systems, SELinux is in **Enforcing** mode by default. If you do not assign the `samba_share_t` context, access to shares will be blocked with **Permission Denied** errors even if standard Linux file permissions are correct.

### Apply SELinux File Contexts

```bash
# Register the persistent SELinux context for the samba directory
sudo semanage fcontext -a -t samba_share_t "/srv/samba(/.*)?"

# Relabel existing files and directories
sudo restorecon -Rv /srv/samba
```

### SELinux Booleans (If Applicable)

If you plan to share user home directories or external drives mounted under `/mnt` or `/media`:

```bash
# Allow sharing of home directories (if needed)
sudo setsebool -P samba_enable_home_dirs on

# Allow read/write export of arbitrary directories (e.g., external disks)
sudo setsebool -P samba_export_all_rw on
```

---

## 4. Samba Configuration (`smb.conf`)

### Step 1: Backup Default Configuration

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

### Step 2: Edit `/etc/samba/smb.conf`

Open the configuration file:

```bash
sudo nano /etc/samba/smb.conf
```

Replace or adapt with the modern, hardened configuration below:

```ini
[global]
    # Network Identity
    workgroup = WORKGROUP
    server string = %h Samba Server (Version %v)
    netbios name = SAMBA-SERVER
    security = user
    map to guest = Bad User
    dns proxy = no

    # ---------------------------------------------------------
    # Modern Protocol & Security Standards
    # ---------------------------------------------------------
    # Minimum protocol SMB2_10 (SMB 2.1) or SMB3_00 (SMB 3.0)
    # SMB1 is disabled by default in modern Samba.
    server min protocol = SMB2_10
    client min protocol = SMB2_10

    # SMB Signing: Mandatory on Windows 11 (24H2+) & protects against MITM
    server signing = mandatory
    client signing = mandatory

    # SMB Encryption (Options: auto, desired, required)
    # 'desired' encrypts if client supports it; 'required' strictly forces encryption
    smb encrypt = desired

    # Restrict anonymous browsing
    restrict anonymous = 2

    # ---------------------------------------------------------
    # macOS & Multi-OS Compatibility (Apple Fruit VFS)
    # ---------------------------------------------------------
    # Dramatically speeds up macOS Finder browsing and handles Apple metadata
    vfs objects = fruit streams_xattr
    fruit:metadata = stream
    fruit:model = Macmini
    fruit:posix_rename = yes
    fruit:veto_appledouble = no
    fruit:wipe_intentionally_left_blank_rfork = yes
    fruit:delete_empty_adfiles = yes

    # ---------------------------------------------------------
    # Performance & File Handling
    # ---------------------------------------------------------
    ea support = yes
    store dos attributes = yes
    inherit permissions = yes
    hide unreadable = yes

    # Logging
    log file = /var/log/samba/log.%m
    max log size = 10000
    logging = systemd

# =============================================================
# Share Definitions
# =============================================================

# 1. Authenticated Group Share (Read/Write for sambashare members)
[Shared]
    comment = General Shared Storage
    path = /srv/samba/shared
    browseable = yes
    read only = no
    guest ok = no
    valid users = @sambashare
    force group = sambashare
    create mask = 0664
    directory mask = 2775
    force create mode = 0664
    force directory mode = 2775

# 2. Secure Private Share (Strict restricted permissions)
[Private]
    comment = Restricted Private Storage
    path = /srv/samba/private
    browseable = yes
    read only = no
    guest ok = no
    valid users = @sambashare
    force group = sambashare
    create mask = 0660
    directory mask = 2770
    force create mode = 0660
    force directory mode = 2770

# 3. (Optional) Public Read-Only / Guest Share
# Uncomment to enable anonymous read-only access
;[Public]
;    comment = Public Read-Only Share
;    path = /srv/samba/public
;    browseable = yes
;    read only = yes
;    guest ok = yes
```

### Step 3: Validate Configuration Syntax

Always verify the configuration syntax with `testparm`:

```bash
sudo testparm -s
```

Ensure no syntax errors or loaded parameter warnings are reported.

---

## 5. User Management

Samba uses its own password database (`passdb`) synchronized with Linux system accounts.

### Step 1: Create a System User (Without Shell Access)

If the user does not already exist on Linux:

```bash
# Create user without interactive shell access and assign to sambashare group
sudo useradd -M -s /usr/sbin/nologin -G sambashare sambauser
```

> [!TIP]
> To add an **existing** Linux user to the Samba group:
> ```bash
> sudo usermod -aG sambashare existingusername
> ```

### Step 2: Set Samba Password & Enable Account

```bash
# Set Samba password
sudo smbpasswd -a sambauser

# Ensure the Samba account is enabled
sudo smbpasswd -e sambauser
```

---

## 6. Network Discovery (WS-Discovery)

Modern Windows 10 & 11 and macOS systems have disabled legacy NetBIOS/SMBv1 network scanning by default. To make your Samba server automatically appear under **Network** in Windows Explorer and macOS Finder, enable the WS-Discovery daemon:

### Debian / Ubuntu (`wsdd2`)

```bash
# Enable and start wsdd2
sudo systemctl enable --now wsdd2

# Check status
sudo systemctl status wsdd2
```

### Fedora Linux (`wsdd`)

```bash
# Enable and start wsdd
sudo systemctl enable --now wsdd

# Check status
sudo systemctl status wsdd
```

---

## 7. Firewall Configuration

Open the required SMB and network discovery ports:

### Debian / Ubuntu (`ufw`)

```bash
# Open Samba ports (TCP 139, 445; UDP 137, 138)
sudo ufw allow Samba

# Open WS-Discovery ports (UDP 3702, TCP 5357) for Windows network auto-discovery
sudo ufw allow 3702/udp
sudo ufw allow 5357/tcp

sudo ufw reload
```

### Fedora Linux (`firewalld`)

```bash
# Open Samba service permanently
sudo firewall-cmd --permanent --add-service=samba

# Open WS-Discovery ports
sudo firewall-cmd --permanent --add-service=ws-discovery
# Or explicitly:
sudo firewall-cmd --permanent --add-port=3702/udp
sudo firewall-cmd --permanent --add-port=5357/tcp

# Apply changes
sudo firewall-cmd --reload
```

---

## 8. Service Management & Verification

### Start & Enable Services

#### On Debian / Ubuntu:
```bash
sudo systemctl restart smbd nmbd wsdd2
sudo systemctl enable smbd nmbd wsdd2
```

#### On Fedora Linux / RHEL:
```bash
sudo systemctl restart smb nmb wsdd
sudo systemctl enable smb nmb wsdd
```

### Test Local Authentication

Verify the share list and authentication from the command line:

```bash
smbclient -L localhost -U sambauser
```

Check active connections and locks in real-time:

```bash
sudo smbstatus
```

---

## 9. Client Connection & Mounting

### Linux Client (Automount via `fstab`)

To mount a Samba share securely on boot without storing plaintext passwords in `/etc/fstab`:

#### 1. Install CIFS Utilities
- **Ubuntu/Debian**: `sudo apt install cifs-utils`
- **Fedora**: `sudo dnf install cifs-utils`

#### 2. Create a Secure Credentials File
```bash
sudo mkdir -p /etc/samba
sudo nano /etc/samba/credentials-share
```

Add your credentials:
```ini
username=sambauser
password=YourSecurePassword
domain=WORKGROUP
```

Lock down file permissions so only root can read it:
```bash
sudo chmod 600 /etc/samba/credentials-share
```

#### 3. Create Mount Point
```bash
sudo mkdir -p /mnt/samba-share
```

#### 4. Add Entry to `/etc/fstab`
```ini
//SERVER_IP/Shared /mnt/samba-share cifs credentials=/etc/samba/credentials-share,iocharset=utf8,vers=3.1.1,uid=1000,gid=1000,file_mode=0775,dir_mode=0775,_netdev,nofail,x-systemd.automount 0 0
```

> [!NOTE]
> - `vers=3.1.1`: Enforces modern SMB 3.1.1.
> - `_netdev`: Waits for network before mounting.
> - `nofail`: Prevents boot hangs if server is offline.
> - `x-systemd.automount`: Mounts on demand when accessed.

#### 5. Test Mount
```bash
sudo systemctl daemon-reload
sudo mount -a
ls -la /mnt/samba-share
```

---

### Windows 10 / 11 Client

1. Open **File Explorer**.
2. Go to **Network** (the server will appear via WS-Discovery).
3. Alternatively, type the UNC path in the address bar:
   ```text
   \\SERVER_IP\Shared
   ```
4. When prompted, enter your Samba username and password. Check **Remember my credentials**.
5. To map to a drive letter: Right-click **This PC** > **Map network drive** > Choose drive letter (e.g. `Z:`) and enter `\\SERVER_IP\Shared`.

---

### macOS Client

1. Open **Finder**.
2. Press `Cmd + K` (or go to **Go** > **Connect to Server...**).
3. Enter the server address:
   ```text
   smb://SERVER_IP/Shared
   ```
4. Click **Connect**, choose **Registered User**, and enter your credentials.

---

## 10. Troubleshooting & Diagnostics

### 1. SELinux Denials (Fedora / RHEL)
If you get **Permission Denied** despite correct `chmod`/`chown`:
```bash
# Check SELinux audit log for denials
sudo ausearch -m avc -ts recent

# Re-apply the Samba context
sudo restorecon -Rv /srv/samba
```

### 2. View Service Logs
- **Debian / Ubuntu**:
  ```bash
  sudo journalctl -u smbd -e
  sudo tail -f /var/log/samba/log.smbd
  ```
- **Fedora / RHEL**:
  ```bash
  sudo journalctl -u smb -e
  sudo tail -f /var/log/samba/log.smb
  ```

### 3. Windows 11 Signing Errors
Windows 11 (24H2+) requires SMB signing by default. If Windows reports `The signature is invalid` or connection fails:
Ensure `server signing = mandatory` is set in the `[global]` section of `/etc/samba/smb.conf`.

### 4. Verify Active Protocol & Cipher Negotiation
To inspect the negotiated SMB protocol version and encryption from a Linux client:
```bash
smbclient //SERVER_IP/Shared -U sambauser -c 'status'
```

---

## Security Best Practices Checklist

- [x] Disabled legacy SMB1 protocol (`server min protocol = SMB2_10` or higher).
- [x] Enabled mandatory SMB signing to defend against NTLM relay and MITM attacks.
- [x] Enabled WS-Discovery (`wsdd`) to eliminate reliance on obsolete NetBIOS / SMB1 broadcasts.
- [x] Restricted permissions with SGID and dedicated `sambashare` group.
- [x] Enforced SELinux labels (`samba_share_t`) on Fedora/RHEL systems.
- [x] Protected client mount credentials using `chmod 600` credentials files.
- [x] Configured `vfs_fruit` for clean macOS Finder and Apple double-file management.

---

## License

This guide is open source and available under the MIT License.
