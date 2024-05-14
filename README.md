# samba-setup
Samba Setup On Debian 12
# Step 1 – How To Install Samba with APT on Debian 12?

To install Samba on Debian 12 using APT, follow these steps:

1. Run system update:
   ```
   sudo apt update
   ```

2. Install Samba packages:
   ```
   sudo apt install samba smbclient cifs-utils
   ```

   This command installs all the required packages and dependencies for Samba.

3. Verify installation:
   ```
   samba --version
   ```

# Step 2 – How To Configure Samba Share on Debian 12?

To configure Samba shares on Debian 12, follow these steps:

1. Locate Samba configuration file:
   The Samba configuration file is located at `/etc/samba/smb.conf`.

2. Set Samba Global Settings:
   Open the Samba configuration file:
   ```
   sudo nano /etc/samba/smb.conf
   ```
   Under the `[global]` section, ensure the following line is configured:
   ```
   workgroup = WORKGROUP
   ```
   Save and close the file.

3. Create Shared Samba Directory:
   You can create public and private directories:
   ```
   sudo mkdir -p /path/to/directory
   ```
   Open the Samba configuration file again:
   ```
   sudo nano /etc/samba/smb.conf
   ```
   Add the share configuration at the end of the file:
   ```
   [samba]
      comment = Samba Folder
      path = /path/to/directory
      writable = yes
      guest ok = no
      valid users = root
      force create mode = 0775
      force directory mode = 0775
      inherit permissions = yes
   ```
   Save and close the file.

4. Create a Samba Share User Group:
   Ensure proper permissions for directories:
   ```
   sudo chmod 2775 /path/to/directory
   ```
   Note: The value 2 sets the SGID bit, allowing newly created files to inherit the parent group.

5. Set Samba user password:
   ```
   sudo smbpasswd -a root
   ```
   Follow the prompts to set the password.
   
   Enable the account:
   ```
   sudo smbpasswd -e root
   ```

6. Verify Samba Configuration:
   Test the Samba configuration:
   ```
   sudo testparm
   ```

