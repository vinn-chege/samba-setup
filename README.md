# Samba Setup Guide for Debian 12;13/Ubuntu 22.04;24.04

A comprehensive guide to installing and configuring Samba with security best practices on Debian 12 and Ubuntu 22.04/24.04.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Security Configuration](#security-configuration)
- [User Management](#user-management)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Root or sudo access
- Debian/Ubuntu/Linux system
- Network connectivity

## Installation

### Step 1: Update System Packages

First, update your system package list:

```bash
sudo apt update
```

### Step 2: Install Samba Packages

Install Samba and required utilities:

```bash
sudo apt install samba smbclient cifs-utils -y
```

This installs:
- `samba`: The main Samba server package
- `smbclient`: SMB client utilities for testing
- `cifs-utils`: Tools for mounting CIFS/SMB shares

### Step 3: Verify Installation

Check the installed Samba version:

```bash
samba --version
```

## Configuration

### Step 1: Backup Configuration File

Before making changes, create a backup of the default configuration:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

### Step 2: Configure Global Settings

Open the Samba configuration file:

```bash
sudo nano /etc/samba/smb.conf
```

Configure the `[global]` section with basic settings:

```ini
[global]
   workgroup = WORKGROUP
   server string = Samba Server %v
   netbios name = SAMBA-SERVER
   security = user
   map to guest = Bad User
   dns proxy = no
```

**Note:** Adjust `WORKGROUP` to match your network workgroup name if different.

### Step 3: Create Shared Directory

Create the directory you want to share:

```bash
sudo mkdir -p /srv/samba/share
```

Set appropriate permissions:

```bash
sudo chmod 2775 /srv/samba/share
```

The `2775` permission sets:
- `2`: SGID bit (new files inherit parent group)
- `775`: Owner and group can read/write/execute, others can read/execute

### Step 4: Configure Share

Add your share configuration at the end of `/etc/samba/smb.conf`:

```ini
[share]
   comment = Samba Share Folder
   path = /srv/samba/share
   writable = yes
   guest ok = no
   valid users = @sambashare
   force create mode = 0664
   force directory mode = 2775
   inherit permissions = yes
   browseable = yes
```

**Configuration Explanation:**
- `comment`: Description of the share
- `path`: Absolute path to the shared directory
- `writable`: Allows write access
- `guest ok = no`: Requires authentication
- `valid users`: Users or groups allowed to access (use `@groupname` for groups)
- `force create mode`: Default permissions for new files
- `force directory mode`: Default permissions for new directories
- `inherit permissions`: Inherit permissions from parent directory

## Security Configuration

### Step 1: Enforce SMBv3 Protocol (Disable Older Versions)

Modern Samba versions support SMBv3, which provides enhanced security. Disable older, insecure SMB versions (SMBv1 and SMBv2) by adding these lines to the `[global]` section:

```ini
[global]
   # ... existing global settings ...
   
   # Protocol security settings
   server min protocol = SMB3
   server max protocol = SMB3
   client min protocol = SMB3
   client max protocol = SMB3
   
   # Disable SMBv1 (insecure)
   server min protocol = SMB3_11
   client min protocol = SMB3_11
```

**Security Benefits:**
- **SMBv3** provides encryption, improved authentication, and better performance
- **SMBv1** is deprecated and vulnerable (e.g., EternalBlue exploit)
- **SMBv2** has known security issues

### Step 2: Enable Encryption

Force encryption for all connections:

```ini
[global]
   # ... existing settings ...
   
   # Force encryption
   smb encrypt = required
```

### Step 3: Configure Firewall

Allow Samba through your firewall:

**UFW (Ubuntu):**
```bash
sudo ufw allow Samba
# Or specify ports manually:
sudo ufw allow 139/tcp
sudo ufw allow 445/tcp
sudo ufw allow 137/udp
sudo ufw allow 138/udp
```

**firewalld (if installed):**
```bash
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
```

**iptables:**
```bash
sudo iptables -A INPUT -p tcp --dport 139 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 445 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 137 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 138 -j ACCEPT
```

### Step 4: Restrict Network Access (Optional)

Limit Samba access to specific networks:

```ini
[global]
   # ... existing settings ...
   
   # Restrict to local network (adjust IP range as needed)
   hosts allow = 192.168.1.0/24 127.0.0.1
   hosts deny = 0.0.0.0/0
```

### Step 5: Additional Security Settings

Add these security hardening options to the `[global]` section:

```ini
[global]
   # ... existing settings ...
   
   # Security hardening
   restrict anonymous = 2
   hide unreadable = yes
   hide dot files = yes
   delete veto files = yes
   veto files = /.*/
   unix extensions = no
```

## User Management

### Step 1: Create Samba User Group

Create a dedicated group for Samba users:

```bash
sudo groupadd sambashare
```

### Step 2: Create System User (if needed)

If the user doesn't exist, create a system user:

```bash
sudo useradd -M -s /usr/sbin/nologin -G sambashare username
```

**Options:**
- `-M`: Don't create home directory
- `-s /usr/sbin/nologin`: Prevent shell login
- `-G sambashare`: Add to sambashare group

### Step 3: Set Samba Password

Set the Samba password for the user (this is separate from the system password):

```bash
sudo smbpasswd -a username
```

You'll be prompted to enter and confirm the password.

### Step 4: Enable Samba User

Enable the Samba user account:

```bash
sudo smbpasswd -e username
```

### Step 5: Set Directory Ownership

Set proper ownership for the shared directory:

```bash
sudo chown -R root:sambashare /srv/samba/share
```

## Verification

### Step 1: Test Configuration

Test the Samba configuration for syntax errors:

```bash
sudo testparm
```

Review the output to ensure your configuration is correct.

### Step 2: Restart Samba Services

Restart Samba services to apply changes:

```bash
sudo systemctl restart smbd
sudo systemctl restart nmbd
```

Enable Samba to start on boot:

```bash
sudo systemctl enable smbd
sudo systemctl enable nmbd
```

### Step 3: Check Service Status

Verify services are running:

```bash
sudo systemctl status smbd
sudo systemctl status nmbd
```

### Step 4: Test Connection Locally

Test the connection from the server:

```bash
smbclient -L localhost -U username
```

### Step 5: Mount Share (Test)

Test mounting the share:

```bash
sudo mkdir /mnt/samba-test
sudo mount -t cifs //localhost/share /mnt/samba-test -o username=username,vers=3.0
```

## Troubleshooting

### View Samba Logs

Check Samba logs for errors:

```bash
sudo tail -f /var/log/samba/log.smbd
sudo tail -f /var/log/samba/log.nmbd
```

### Common Issues

1. **Permission Denied**: Check directory permissions and ownership
2. **Connection Refused**: Verify firewall rules and service status
3. **Authentication Failed**: Ensure user is enabled with `smbpasswd -e`
4. **Protocol Negotiation Failed**: Verify SMB version settings match client capabilities

### Verify SMB Protocol Version

Check which SMB protocol versions are enabled:

```bash
sudo testparm -v | grep -i protocol
```

## Additional Resources

- [Samba Official Documentation](https://www.samba.org/samba/docs/)
- [Samba Security Best Practices](https://wiki.samba.org/index.php/Samba_Security_Considerations)
- [SMB Protocol Versions](https://wiki.samba.org/index.php/Samba3/SMB2)

## License

This guide is provided as-is for educational purposes.
