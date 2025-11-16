# Securing SSH Access on Proxmox VE 9+

Complete guide to strengthen Proxmox server security by applying the principle of least privilege.

## 📋 Objective

This guide allows you to:

- Create a dedicated administrator user
- Configure SSH key authentication for this user and for root
- Disable password authentication and restrict direct root access
- Integrate the new user into the Proxmox web interface with full privileges

## ⚙️ Prerequisites

- Initial access as `root` to your Proxmox server (via SSH or the web interface shell)
- Your SSH public key (contained in a file like `~/.ssh/id_rsa.pub` or `~/.ssh/id_ed25519.pub` on your local machine)

## 🚀 Installation

### Step 1: Create the Administrative User

We create a standard user (e.g., `pveadmin`) who will obtain root privileges only when necessary via the `sudo` command.

**Create the user:**

```bash
# This command creates the user and their home directory /home/pveadmin
adduser pveadmin
```

Follow the instructions to set a strong password.

**Add the user to the sudo group:**

```bash
# This gives them the right to execute commands as root
usermod -aG sudo pveadmin
```

### Step 2: Configure SSH Key Authentication

This step enables secure, passwordless connection. We will add your public key to both `pveadmin` and `root` accounts.

**For the pveadmin user:**

```bash
# Switch to the pveadmin user
su - pveadmin

# Create the .ssh directory with proper permissions
mkdir ~/.ssh
chmod 700 ~/.ssh

# Add your public key to the authorized_keys file
# REPLACE the content in quotes with YOUR complete public key
echo "ssh-rsa AAAA... your_email@machine" >> ~/.ssh/authorized_keys

# Apply security permissions to the file
chmod 600 ~/.ssh/authorized_keys

# Return to the root session
exit
```

**For the root user:**

This is a good practice to maintain a backup key-based access.

```bash
# Make sure you're root. Create the .ssh directory if it doesn't exist.
mkdir -p /root/.ssh
chmod 700 /root/.ssh

# Add your public key to root's authorized_keys file
echo "ssh-rsa AAAA... your_email@machine" >> /root/.ssh/authorized_keys

# Apply security permissions
chmod 600 /root/.ssh/authorized_keys
```

### Step 3: Critical Connection Verification

⚠️ **DO NOT PROCEED TO THE NEXT STEP WITHOUT PASSING THIS TEST.**

Open a new terminal on your local machine and test both connections. They should connect without asking for a password.

```bash
# Connection test for pveadmin
ssh pveadmin@YOUR_PROXMOX_IP

# Connection test for root
ssh root@YOUR_PROXMOX_IP
```

### Step 4: Secure the SSH Server (sshd_config)

Now that key-based access is functional, we can disable less secure connection methods.

**Open the SSH configuration file:**

```bash
nano /etc/ssh/sshd_config
```

**Modify or add the following lines:**

Make sure they are not commented out (no `#` at the beginning).

```ini
# Authentication:

# Prohibits root login by password, but allows it via SSH key.
# This is a good compromise for Proxmox cluster management and emergency access.
# For maximum security (completely block root SSH), use 'no' instead.
PermitRootLogin prohibit-password

# Ensure that public key authentication is explicitly enabled.
PubkeyAuthentication yes

# Completely disable password authentication for all users.
# This is the most important step to prevent brute force attacks.
PasswordAuthentication no
```

**Restart the SSH service to apply the changes:**

```bash
systemctl restart sshd
```

### Step 5: Integration with Proxmox Web Interface

The `pveadmin` user exists on the system but not yet in Proxmox management. We will add it and grant administrator rights.

1. Log in to the Proxmox web interface as `root`

2. **Add the user:**
   - Go to `Datacenter` → `Permissions` → `Users`
   - Click on `Add`
   - User name: `pveadmin`
   - Realm: `Linux PAM standard authentication`
   - Click on `Add`

3. **Assign permissions:**
   - Go to `Datacenter` → `Permissions`
   - Click on `Add` → `User Permission`
   - Path: `/` (the root, for the entire datacenter)
   - User: `pveadmin@pam`
   - Role: `Administrator`
   - Click on `Add`

4. **Enable Two-Factor Authentication (MFA) - Recommended:**
   - Go to `Datacenter` → `Permissions` → `Two Factor`
   - Select the user `pveadmin@pam`
   - Click on `Add` → `TOTP`
   - Scan the QR code with your authenticator app (Google Authenticator, Authy, Microsoft Authenticator, etc.)
   - Enter the verification code from your app
   - Click on `Add`
   
   Alternatively, you can configure MFA directly from the user settings:
   - Go to `Datacenter` → `Permissions` → `Users`
   - Select `pveadmin@pam`
   - Click on `TFA` button
   - Choose your preferred method (TOTP, WebAuthn, or Recovery Keys)
   - Follow the setup wizard

## ✅ Final Result

You can now log out of the web interface and reconnect with the `pveadmin` user.

- ✅ Direct SSH access with root password is blocked (key-based access still available for emergency)
- ✅ Daily SSH access is done via `pveadmin`, a user with standard privileges
- ✅ Administrative tasks on the command line require `sudo`, ensuring better traceability
- ✅ Authentication is secured by a cryptographically robust SSH key
- ✅ The new `pveadmin` user has full access to the web interface, equivalent to root
- ✅ Two-Factor Authentication (MFA) adds an extra layer of security to web interface access

## 🔒 Best Practices

- Always keep a copy of your SSH private key in a safe place
- Never share your SSH private key
- Use ED25519 SSH keys for optimal security: `ssh-keygen -t ed25519 -C "your_email@example.com"`
- Always test your SSH connections before closing your initial root session
- Enable MFA/2FA for all administrative accounts
- Store your MFA recovery codes in a secure location (password manager or encrypted storage)

## 📝 License

This guide is provided for informational purposes. Use it at your own risk.

---

**Note:** Always ensure you have backup access to your server (physical console or KVM) before modifying SSH configuration.
