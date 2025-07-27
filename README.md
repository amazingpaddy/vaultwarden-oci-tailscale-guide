# Vaultwarden Self-Hosting Guide: Oracle Cloud Always Free & Tailscale

This guide provides a comprehensive, step-by-step process for self-hosting Vaultwarden on an Oracle Cloud Infrastructure (OCI) Ubuntu VM using the Always Free Tier and securing access exclusively with Tailscale. This setup allows you to host your password manager without incurring any monetary costs for the VM or network access.

---

## Prerequisites

Before you begin, ensure you have the following:

1.  **Oracle Ubuntu VM Instance (VM.Standard.A1.Flex):** You should have an active Oracle Ubuntu VM instance with SSH access. This VM is part of the OCI Always Free Tier.
2.  **Non-root user with sudo privileges:** It is best practice to perform these operations as a non-root user with sudo access for security.
3.  **Tailscale Account:** A free Tailscale account is required to create your private, encrypted network.
4.  **Basic understanding of Linux command line.**

---

## Index of Steps

1.  Update and Upgrade System
2.  Assign a Reserved Public IP Address to Your VM (Oracle Cloud)
3.  Configure Firewall (UFW) and Oracle Cloud Security List
4.  Install Docker and Docker Compose
5.  Install Vaultwarden with Docker Compose (Local Binding)
6.  Automated Encrypted Backup to Oracle Cloud Object Storage (OCI) using rclone
7.  Automated Scheduled Backups with rclone and Cron
8.  Configure Logrotate for Backup Logs
9.  Set up Tailscale for Private Access
10. Harden VM Security (Close Public Ports)
11. Post-Installation & Maintenance

---

## Step-by-Step Guide

### 1. Update and Upgrade System

First, ensure your Ubuntu VM is up to date.

```bash
sudo apt update
sudo apt upgrade -y
```

After these commands, a reboot is highly recommended for the new kernel to take effect and to ensure all updates are fully applied.

```bash
sudo reboot
```

Wait a minute or two for the VM to come back online, then reconnect via SSH.

**Note on VM Reboot Warning:** When rebooting from the Oracle Cloud console, you might see a warning about applications taking more than 15 minutes to shut down and potential data corruption. For a freshly updated VM without active services, this risk is extremely low, and you can safely proceed.

### 2. Assign a Reserved Public IP Address to Your VM (Oracle Cloud)

For long-term reliability, assign a reserved public IP address to your VM. One reserved public IP address is included in the Always Free tier. This IP will not be used for direct access to Vaultwarden, but is necessary for initial SSH and OCI management.

**Important Note on Ephemeral IPs:** By default, public IP addresses for Oracle Cloud "Always Free" VM instances are ephemeral (temporary). If you stop and then start your VM, the public IP address will change. A reserved public IP address will never change, even if you stop and start your instance, ensuring stability for your VM's external identity.

1.  **Log in to your OCI Console.**
2.  Navigate to **Networking > Public IPs**.
3.  Click on **"Reserve Public IP"**.
    * **Name:** Give it a meaningful name, e.g., `your_reserved_ip_name`.
    * **Compartment:** Select the same compartment where your VM instance resides.
    * **IP Address Pool:** Leave as "Oracle".
    * Click **"Reserve"**.
4.  Once the IP is reserved, assign it to your VM instance:
    * Go to **Compute > Instances**.
    * Click on your VM instance (e.g., `your-vm-name`).
    * In the "Attached VNICs" section, click on the VNIC that has the public IP you want to replace.
    * Under "IPv4 Addresses," click on the "Edit" link next to your current ephemeral public IP.
    * For "Public IP," select **"EXISTING_PUBLIC_IP"** from the dropdown menu.
    * Choose the reserved IP address you just created from the list.
    * Click **"Update"**.

### 3. Configure Firewall (UFW) and Oracle Cloud Security List

Ubuntu's Uncomplicated Firewall (UFW) and Oracle Cloud Infrastructure (OCI) Security List are crucial for securing your VM. Initially, we'll keep SSH open. Other ports will be closed after Tailscale is configured.

#### 3.1. Install UFW (if not already installed)

**Note:** If `ufw` command is not found, it means UFW is not installed by default on your specific Ubuntu image.

```bash
sudo apt install ufw -y
```

#### 3.2. Configure UFW

```bash
sudo ufw status                # Check UFW status (it might be inactive initially)
sudo ufw allow OpenSSH         # Ensure SSH is allowed so you don't lock yourself out
sudo ufw enable                # Enable UFW
sudo ufw status                # Verify the rules are active
```

#### 3.3. Configure Oracle Cloud Infrastructure (OCI) Security List

This is crucial for allowing external traffic to reach your VM.

1.  **Log in to your Oracle Cloud console.**
2.  Navigate to **Networking -> Virtual Cloud Networks**.
3.  Click on the VCN associated with your VM.
4.  Click on "Security Lists" (usually the default one).
5.  Add an Ingress Rule for:
    * **Port 22 (SSH):**
        * Source CIDR: `0.0.0.0/0` (or a more restricted IP range if you have a static public IP for your local machine)
        * IP Protocol: `TCP`
        * Destination Port Range: `22`
        * Description: `SSH`

### 4. Install Docker and Docker Compose

Docker will be used to run Vaultwarden in containers. Docker Compose is installed as a plugin.

```bash
# Add Docker's official GPG key & repository
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker packages
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker # Apply the group change immediately without logging out and back in
```

Verify Docker installation:

```bash
docker run hello-world
```

You should see a message confirming Docker is working.

Verify Docker Compose installation:

```bash
docker compose version
```

You should see output similar to `Docker Compose version v2.x.x`.

### 5. Install Vaultwarden with Docker Compose (Local Binding)

Vaultwarden will be configured to listen only on the local machine's loopback interface (`127.0.0.1`), meaning it won't be publicly accessible directly. Tailscale will then proxy to this local address.

#### 5.1. Create a directory for Vaultwarden data

```bash
mkdir ~/vaultwarden-data
cd ~/vaultwarden-data
```

#### 5.2. Create `docker-compose.yml` for Vaultwarden

```bash
nano docker-compose.yml
```

Paste the following content into the file. **Before saving, make the crucial edits below.**

```yaml
version: '3.8'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    volumes:
      - ./data:/data # This will store your Vaultwarden data persistently
    environment:
      # IMPORTANT: Vaultwarden will be accessed via Tailscale hostname (e.g., [https://your-vm-hostname.your-tailnet-name.ts.net](https://your-vm-hostname.your-tailnet-name.ts.net))
      # This DOMAIN setting tells Vaultwarden its internal URL, which Tailscale will proxy to.
      DOMAIN: 'http://127.0.0.1:8080'
      # Optional: Admin page token (highly recommended for management)
      # Generate a strong, random string: openssl rand -base64 48
      # ADMIN_TOKEN: 'YOUR_GENERATED_TOKEN_HERE' # This will be replaced by a hash later via admin panel
      SIGNUPS_ALLOWED: 'true' # Set to 'false' after creating your first user
      # For other environment variables, refer to the Vaultwarden wiki:
      # [https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview](https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview)
    # Vaultwarden listens on port 80 by default inside the container.
    # We bind it to 127.0.0.1:8080 on the VM so only local processes (like Tailscale) can access it.
    ports:
      - '127.0.0.1:8080:80' # Bind to localhost:8080 on the VM
      - '3012:3012' # Websocket port for real-time sync (can also be bound to 127.0.0.1 if strict)
networks:
  default: # No external network needed if only accessed locally or via Tailscale
    external: false
```

**Crucial Edits before saving:**

1.  **Admin Token (ADMIN_TOKEN):**
    * Generate a strong, random string for the token:
        ```bash
        openssl rand -base64 48
        ```
    * Copy the output.
    * Uncomment the `ADMIN_TOKEN` line in your `docker-compose.yml` (remove the `#`).
    * Replace `'YOUR_GENERATED_TOKEN_HERE'` with the random string you generated. Keep this token safe!
2.  **Signups Allowed (SIGNUPS_ALLOWED):** Leave `SIGNUPS_ALLOWED: 'true'` for now. You'll need this to create your first user account.

Save the file: Press `Ctrl+O`, then `Enter`, then `Ctrl+X`.

#### 5.3. Start Vaultwarden

```bash
docker compose up -d
```

### 6. Automated Encrypted Backup to Oracle Cloud Object Storage (OCI) using rclone

We will use `rclone` for backups, encrypting data locally before sending it to OCI. This is a free solution using OCI's Always Free Object Storage.

#### 6.1. Create an OCI User and Auth Token for rclone

1.  **Create an IAM User for rclone:**
    * In OCI Console: Navigate to **Identity & Security > Identity > Users**.
    * Click **"Create User"**.
    * **Name:** `rclone_backup_user` (or a similar descriptive name).
    * Provide First Name, Last Name, and Email (e.g., `backup@example.com`).
    * Click **"Create"**.
2.  **Generate API Key for the User:**
    * Click on the newly created user (e.g., `rclone_backup_user`).
    * Under "Resources" (left-hand menu), click **"API Keys"**.
    * Click **"Add API Key"**.
    * Select **"Generate API Key Pair"**.
    * Click **"Download Private Key"**. Save this file (e.g., `oci_api_key.pem`) securely on your local computer.
    * Click **"Add"**.
    * **Crucial Step:** OCI will then show you the "Configuration File Preview". Copy the entire content of this preview (it starts with `[DEFAULT]` and includes `user`, `fingerprint`, `key_file`, `tenancy`, and `region`). Paste this into a temporary text file on your local computer. You'll need this for `rclone config`.
3.  **Generate an Auth Token for the User:**
    * Under "Resources" (left-hand menu), click **"Auth Tokens"**.
    * Click **"Generate Token"**.
    * Give it a **Description** (e.g., `rclone_backup_token`).
    * Click **"Generate Token"**.
    * **CRUCIAL:** OCI will display the Auth Token string **ONCE**. Copy this entire token string immediately and securely. You will not be able to retrieve it again.
4.  **Create an IAM Group and Add the User:**
    * In OCI Console: Navigate to **Identity & Security > Identity > Groups**.
    * Click **"Create Group"**.
    * **Name:** `rclone_backup_group`
    * Click **"Create"**.
    * Click on the newly created `rclone_backup_group`.
    * Click **"Add User to Group"**.
    * Select `rclone_backup_user` and click **"Add"**.
5.  **Create an IAM Policy to Allow Access:**
    * In OCI Console: Navigate to **Identity & Security > Identity > Policies**.
    * Click **"Create Policy"**.
    * **Name:** `rclone_backup_policy`
    * **Description:** Allows `rclone_backup_group` to manage object storage backups.
    * **Policy Version:** Set to `VERSION 2`.
    * **Policy Statements:** Click **"Show manual editor"** and paste the following, replacing `<OCID_OF_YOUR_COMPARTMENT>` with the actual OCID of your compartment (e.g., `ocid1.compartment.oc1..aaaaaxxxxxxxx`):
        ```
        Allow group rclone_backup_group to manage objects in compartment id <OCID_OF_YOUR_COMPARTMENT> where target.bucket.name = 'vaultwarden-backups-yourname'
        Allow group rclone_backup_group to manage buckets in compartment id <OCID_OF_YOUR_COMPARTMENT> where target.bucket.name = 'vaultwarden-backups-yourname'
        ```
    * Click **"Create"**.

#### 6.2. Create an OCI Object Storage Bucket

1.  **Log in to your OCI Console.**
2.  Navigate to **Storage > Buckets**.
3.  Ensure you are in the correct **Compartment**.
4.  Click **"Create Bucket"**.
    * **Bucket Name:** Give it a meaningful name, e.g., `vaultwarden-backups-yourname` (must be unique across all of OCI).
    * **Storage Tier:** Select `Standard`.
    * Leave other options as default.
    * Click **"Create"**.

#### 6.3. Configure rclone for OCI Object Storage

1.  Transfer your `oci_api_key.pem` (the one for `rclone_backup_user`) to your VM:
    ```bash
    scp -i ~/.ssh/your_vm_ssh_key.pem /path/to/downloaded/oci_api_key.pem ubuntu@YOUR_VM_PUBLIC_IP:~/.oci/oci_api_key.pem
    ```
    On your VM, secure the key:
    ```bash
    mkdir -p ~/.oci
    chmod 600 ~/.oci/oci_api_key.pem
    ```
2.  Start the interactive `rclone` configuration:
    ```bash
    rclone config
    ```
3.  Follow the prompts:
    * `n)` for New remote
    * **Name:** `oci_vaultwarden_backup`
    * **Storage:** Type `oracleobjectstorage` (or its number, usually `41`)
    * **Provider:** Type `user_principal_auth` (or its number, usually `2`)
    * **Config file path:** Press `Enter` (default `~/.oci/config`)
    * **Config profile:** Press `Enter` (default `Default`)
    * **Tenancy OCID:** Paste your Tenancy OCID (from "Configuration File Preview").
    * **User OCID:** Paste your `rclone_backup_user`'s OCID.
    * **Region:** Type `ca-toronto-1` (your OCI region).
    * **Private Key file path:** Type `/home/ubuntu/.oci/oci_api_key.pem` (the absolute path where you `scp`'d the key).
    * **Private Key file pass phrase:** Leave empty. Press `Enter`.
    * **Fingerprint:** Paste the Fingerprint of the `rclone_backup_user`'s public key.
    * **Auth Token:** Leave empty. Press `Enter`.
    * **Compartment OCID:** Paste the OCID of your `your_compartment_name` compartment (where your bucket is).
    * **Endpoint:** Leave empty. Press `Enter`.
    * **Storage Tier:** Leave empty. Press `Enter`.
    * **Edit advanced config? (y/n):** Type `n` and press `Enter`.
    * `y)/e)/d)/r)/c)/s)/q):` Type `q` to quit the config.

#### 6.4. Configure an rclone crypt remote

This will create a new virtual remote that encrypts data before sending it to your `oci_vaultwarden_backup` remote.

1.  Start the interactive `rclone` configuration again:
    ```bash
    rclone config
    ```
2.  Follow the prompts for a new remote:
    * `n)` for New remote
    * **Name:** `vaultwarden_encrypted`
    * **Storage:** Type `crypt` (or its number, usually `15`)
    * **Remote to encrypt/decrypt:** Type `oci_vaultwarden_backup:vaultwarden-backups-yourname/vaultwarden_backups` and press `Enter`.
        * `oci_vaultwarden_backup`: This is the name of the OCI remote you just configured.
        * `vaultwarden-backups-yourname`: This is your actual OCI bucket name.
        * `/vaultwarden_backups`: This specifies a subdirectory within your OCI bucket where encrypted files will be stored.
    * **Password:** Type `1` for "Enter your own password" and press `Enter`.
        * **Enter password for encryption:** Enter a very strong, unique password here. **Store this password extremely securely!**
        * **Confirm password:** Enter the same password again.
    * **Password confirmation:** Type `1` for "Enter your own password" and press `Enter`.
        * **Enter password for encryption:** Enter the same password again.
        * **Confirm password:** Enter the same password again.
    * **Filename encryption:** Type `3` for "off" and press `Enter`.
    * **Directory name encryption:** Type `3` for "off" and press `Enter`.
    * **Password or pass phrase for salt:** Type `n` for "No" and press `Enter`.
    * **Edit advanced config? (y/n):** Type `n` and press `Enter`.
    * `y)/e)/d)/r)/c)/s)/q):` Type `q` to quit the `rclone config` interface.

### 7. Automated Scheduled Backups with rclone and Cron

#### 7.1. Create the Backup Script

```bash
mkdir -p ~/vaultwarden-data/scripts # Create scripts directory inside vaultwarden-data for backup
nano ~/vaultwarden-data/scripts/vaultwarden_backup.sh
```

Paste the following content into the script. Adjust `RETENTION_DAYS` if needed.

```bash
#!/bin/bash
# --- Configuration ---
# Source directory on your VM containing Vaultwarden data
SOURCE_DIR="/home/ubuntu/vaultwarden-data"
# Name of the rclone crypt remote that points to OCI Object Storage
RCLONE_REMOTE="vaultwarden_encrypted"
# Subdirectory within the rclone crypt remote to store backups (e.g., vaultwarden_encrypted:backups/)
RCLONE_DEST_SUBDIR="daily_snapshots" # This will be the base folder for your timestamped backups
# Log file for backup activity
LOG_FILE="/var/log/vaultwarden_backup.log"
# Retention policy: Keep backups for this many days (e.g., 30 for approx 1 month)
RETENTION_DAYS=30

# Ensure log file exists and is writable
sudo touch "$LOG_FILE"
sudo chmod 666 "$LOG_FILE" # Make it writable by all, or 660 and ensure ubuntu is in a group that can write to /var/log

# --- Backup Function ---
perform_backup() {
  TIMESTAMP=$(date +%Y-%m-%d_%H-%M-%S)
  DEST_PATH="$RCLONE_REMOTE:$RCLONE_DEST_SUBDIR/$TIMESTAMP"
  echo "$(date): Starting Vaultwarden backup to $DEST_PATH..." | tee -a "$LOG_FILE"
  # Use rclone copy to create a new snapshot in a timestamped directory
  rclone copy "$SOURCE_DIR/" "$DEST_PATH" \
    --exclude="tmp/**" \
    --exclude="rclone.conf" \
    --exclude=".restic_env" \
    --exclude="logs/**" \
    --log-file="$LOG_FILE" \
    --log-level="INFO" \
    --progress # Progress is usually seen in manual runs, not in cron logs

  if [ $? -eq 0 ]; then
    echo "$(date): Backup to $DEST_PATH completed successfully." | tee -a "$LOG_FILE"
  else
    echo "$(date): ERROR: Backup to $DEST_PATH failed!" | tee -a "$LOG_FILE"
    # Optional: Add error notification here (e.g., email notification script)
    exit 1 # Exit with error status if backup fails
  fi
}

# --- Retention Function ---
manage_retention() {
  echo "$(date): Starting backup retention cleanup (keeping last ${RETENTION_DAYS} days)..." | tee -a "$LOG_FILE"

  # List all directories under RCLONE_REMOTE:RCLONE_DEST_SUBDIR
  # Filter for directories that are named as timestamps (YYYY-MM-DD_HH-MM-SS)
  rclone lsf "$RCLONE_REMOTE:$RCLONE_DEST_SUBDIR" --dirs-only | while read -r DIR; do
    DIR_NAME=$(echo "$DIR" | sed 's/\///') # Remove trailing slash

    # Extract date part (YYYY-MM-DD) from directory name
    # Basic validation to ensure it looks like a date
    if [[ "$DIR_NAME" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}_[0-9]{2}-[0-9]{2}-[0-9]{2}$ ]]; then
      DIR_DATE_PART=$(echo "$DIR_NAME" | cut -d'_' -f1)
    else
      echo "$(date): Skipping non-timestamped directory: $DIR_NAME" | tee -a "$LOG_FILE"
      continue
    fi

    # Calculate age in days (requires GNU date, standard on Ubuntu)
    if ! OLD_SECONDS=$(date -d "$DIR_DATE_PART" +%s 2>/dev/null); then
      echo "$(date): Error parsing date for directory: $DIR_NAME" | tee -a "$LOG_FILE"
      continue
    fi

    CURRENT_SECONDS=$(date +%s)
    AGE_SECONDS=$((CURRENT_SECONDS - OLD_SECONDS))
    AGE_DAYS=$((AGE_SECONDS / 86400)) # 86400 seconds in a day

    if [ "$AGE_DAYS" -gt "$RETENTION_DAYS" ]; then
      echo "$(date): Deleting old backup snapshot: $RCLONE_REMOTE:$RCLONE_DEST_SUBDIR/$DIR_NAME (Age: ${AGE_DAYS} days)" | tee -a "$LOG_FILE"
      rclone purge "$RCLONE_REMOTE:$RCLONE_DEST_SUBDIR/$DIR_NAME" \
        --log-file="$LOG_FILE" \
        --log-level="INFO"

      if [ $? -ne 0 ]; then
        echo "$(date): ERROR: Failed to delete old backup snapshot $DIR_NAME" | tee -a "$LOG_FILE"
      fi
    fi
  done
  echo "$(date): Backup retention cleanup completed." | tee -a "$LOG_FILE"
}

# --- Main Execution ---
perform_backup
manage_retention
```

Save the script: Press `Ctrl+O`, then `Enter`, then `Ctrl+X`.

#### 7.2. Make the Script Executable

```bash
chmod +x ~/vaultwarden-data/scripts/vaultwarden_backup.sh
```

#### 7.3. Test the Script Manually

```bash
~/vaultwarden-data/scripts/vaultwarden_backup.sh
```

Check the output on your terminal and then check `tail /var/log/vaultwarden_backup.log` and your OCI bucket.

**Note on Backup Content:** `rclone` by default does not copy empty directories unless you specifically use the `--create-empty-src-dirs` flag. If subdirectories like `attachments/` are empty on your VM, they will not appear in the backup until they contain files.

#### 7.4. Set Up the Cron Job

This will schedule the script to run daily at 3:00 AM and 3:00 PM.

1.  Open your crontab for editing:
    ```bash
    crontab -e
    ```
2.  Add the following two lines at the end of the file, updating the path to your backup script:
    ```cron
    0 3 * * * /home/ubuntu/vaultwarden-data/scripts/vaultwarden_backup.sh >> /var/log/cron.log 2>&1
    0 15 * * * /home/ubuntu/vaultwarden-data/scripts/vaultwarden_backup.sh >> /var/log/cron.log 2>&1
    ```
3.  Save and exit the crontab editor.

### 8. Configure Logrotate for Backup Logs (30-day retention)

This will automatically rotate and prune your log files.

1.  Create a new `logrotate` configuration file:
    ```bash
    sudo nano /etc/logrotate.d/vaultwarden-backup
    ```
2.  Paste the following content into the file:
    ```
    /var/log/vaultwarden_backup.log /var/log/cron.log {
        daily
        rotate 30
        compress
        delaycompress
        missingok
        notifempty
        create 0640 ubuntu adm
        sharedscripts
        postrotate
            /usr/bin/find /var/log/ -name "vaultwarden_backup.log.[0-9]*.gz" -mtime +30 -delete || true
            /usr/bin/find /var/log/ -name "cron.log.[0-9]*.gz" -mtime +30 -delete || true
        endscript
    }
    ```
    **Explanation of Logrotate Directives:**
    * `/var/log/vaultwarden_backup.log /var/log/cron.log`: Specifies the log files to be managed.
    * `daily`: Rotates the log file every day.
    * `rotate 30`: Keeps 30 rotated log files (effectively 30 days of compressed history).
    * `compress`: Compresses the rotated log files.
    * `delaycompress`: Delays compression until the next rotation cycle.
    * `missingok`: If the log file is missing, do not exit with an error.
    * `notifempty`: Do not rotate the log file if it is empty.
    * `create 0640 ubuntu adm`: Creates a new log file after rotation with specified permissions, owner (`ubuntu`), and group (`adm`).
    * `sharedscripts`: If multiple log files are specified, run the `postrotate` script only once after all logs have been rotated.
    * `postrotate ... endscript`: Commands to execute after the log file has been rotated, ensuring older compressed logs are explicitly deleted.
3.  Save the file: Press `Ctrl+O`, then `Enter`, then `Ctrl+X`.

**Note on Logrotate Execution:** The `logrotate` utility is typically run once a day by a system-wide cron job (usually via `/etc/cron.daily/`). You do not need to add it to your personal crontab. It's highly unlikely to clash with your rclone backup as they operate on different file sets.

### 9. Set up Tailscale for Private Access

Tailscale creates a secure, encrypted mesh network (a "tailnet") between your devices and servers, allowing them to communicate as if they were on the same local network. This enhances security by reducing public exposure.

**Important Note on Tailscale Cost:** Tailscale offers a free tier that allows up to 3 users and 100 devices, which is sufficient for personal or small family use.

**Important Note on SSH Access with Tailscale:** After setting up Tailscale, you can still SSH into your VM using its public IP address if Port 22 (SSH) remains open in your Oracle Cloud Security List and UFW. However, a key benefit of Tailscale is the ability to close public ports, including SSH, and rely solely on Tailscale for secure access.

#### 9.1. Sign Up for Tailscale

1.  Open your web browser on your local computer.
2.  Go to the Tailscale website: `https://tailscale.com/`
3.  Click on the "Sign Up" or "Get Started for Free" button.
4.  You'll be prompted to sign up using an existing identity provider like Google, Microsoft, Apple, or GitHub. Choose the one you prefer and follow the authentication steps.

#### 9.2. Install Tailscale on Your Oracle Ubuntu VM

1.  Go to your SSH terminal connected to your Oracle Ubuntu VM.
2.  Add Tailscale's GPG key and repository:
    ```bash
    # Add Tailscale's GPG key
    curl -fsSL [https://pkgs.tailscale.com/stable/ubuntu/noble.gpg](https://pkgs.tailscale.com/stable/ubuntu/noble.gpg) | sudo gpg --dearmor -o /usr/share/keyrings/tailscale-archive-keyring.gpg
    # Add the Tailscale apt repository
    echo "deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] [https://pkgs.tailscale.com/stable/ubuntu](https://pkgs.tailscale.com/stable/ubuntu) noble main" | sudo tee /etc/apt/sources.list.d/tailscale.list
    ```
    **Note:** `noble` is the codename for Ubuntu 24.04. If your VM is running a different Ubuntu version, please replace `noble` with your version's codename (e.g., `jammy` for Ubuntu 22.04, `focal` for Ubuntu 20.04). You can check your Ubuntu version with `lsb_release -cs`.
3.  Update package lists and install Tailscale:
    ```bash
    sudo apt update
    sudo apt install tailscale -y
    ```
4.  Connect your VM to your Tailscale network:
    ```bash
    sudo tailscale up
    ```
    This command will output a unique authentication URL in your terminal. Copy this entire URL, paste it into your local computer's web browser, log in to your Tailscale account, and authorize the new device (your VM).
5.  Verify Tailscale connection:
    ```bash
    tailscale status
    ```
    You should see your VM listed with a private Tailscale IP address (e.g., `YOUR_VM_TAILSCALE_IP`) and its hostname (e.g., `your-vm-hostname`), with an "active; logged-in" status.

#### 9.3. Enable Tailscale HTTPS (MagicDNS)

1.  Open your web browser and go to your Tailscale admin console: `https://login.tailscale.com/admin/dns`
2.  On the DNS page, ensure **"MagicDNS" is enabled**.
3.  Also, ensure **"Override local DNS" is checked** under MagicDNS settings.
4.  Ensure **"Enable Https"** under **"HTTPS Certificates"** is enabled for your tailnet (e.g., `your-tailnet-name.ts.net`). This tells Tailscale to issue and manage SSL certificates for your private Tailscale hostnames.

#### 9.4. Configure Tailscale to Serve HTTPS Directly

This step configures Vaultwarden to work directly with `tailscale serve`, bypassing Nginx Proxy Manager for Tailscale access.

1.  **Important:** Update Vaultwarden's `DOMAIN` via Admin Panel:
    * Access your Vaultwarden Admin Panel via `http://127.0.0.1:8080/admin` (you may need to temporarily bind Vaultwarden to a public IP to access its admin panel if you've already configured it for local binding).
    * Log in using your `ADMIN_TOKEN`.
    * Navigate to **Settings**.
    * Change the `DOMAIN` field to: `http://127.0.0.1:8080`. Ensure it is exactly `http://127.0.0.1:8080`.
    * Click **Save**.
    * Restart Vaultwarden again for this `config.json` change to take effect:
        ```bash
        cd ~/vaultwarden-data
        docker compose down
        docker compose up -d
        ```
2.  Go to your SSH terminal on the VM.
3.  Run this command:
    ```bash
    sudo tailscale serve --bg --https=443 127.0.0.1:8080
    ```
    * `--bg`: Runs the service persistently in the background. It will automatically resume after reboots or `tailscale up`.
    * `--https=443`: Explicitly tells Tailscale to serve HTTPS on port 443.
    * `127.0.0.1:8080`: This is the target local address where your Vaultwarden instance is listening internally on the VM.
    * **Note:** If `jq` is not installed, you might need to install it first: `sudo apt install jq -y`.
4.  Verify Tailscale serve status:
    ```bash
    tailscale serve status
    ```
    You should see an entry similar to:
    `https://your-vm-hostname.your-tailnet-name.ts.net/`
    `|-- proxy http://127.0.0.1:8080`

#### 9.5. Configure Bitwarden Clients for Tailscale HTTPS Access

1.  Ensure Tailscale is active and connected on your local device (computer/mobile).
2.  Open your Bitwarden client.
3.  Go to the "Self-host" or "Custom URL" setting.
4.  Change the server URL to your full Tailscale hostname with HTTPS: `https://your-vm-hostname.your-tailnet-name.ts.net`
5.  Attempt to log in. You should now see a trusted connection and the login page should load correctly.

### 10. Harden VM Security (Close Public Ports)

Since you are now accessing Vaultwarden securely over your private Tailscale network, you no longer need any public ports open to the internet except for SSH (Port 22) if you still require it. Closing these ports significantly reduces your VM's attack surface and makes your Vaultwarden instance invisible to public internet scanners.

1.  Go to your SSH terminal on the VM.
2.  Close public firewall ports (UFW):
    ```bash
    sudo ufw delete allow http >/dev/null 2>&1 # Close port 80
    sudo ufw delete allow https >/dev/null 2>&1 # Close port 443
    sudo ufw delete allow 81/tcp >/dev/null 2>&1 # Close port 81 (NPM Admin UI)
    sudo ufw status | grep -E '80|443|81' # Check UFW status
    ```
3.  **Crucial: Close Public Ports in Oracle Cloud Infrastructure (OCI) Security List:**
    * Log in to your Oracle Cloud Infrastructure (OCI) Console.
    * Navigate to **Networking > Virtual Cloud Networks**.
    * Click on your VCN associated with your VM.
    * Click on "Security Lists" (usually the default one).
    * Find the Ingress Rules for your VM.
    * **Delete** the Ingress Rules for Port 80 (HTTP), Port 443 (HTTPS), and Port 81 (NPM Admin UI) that have Source CIDR: `0.0.0.0/0`.
    * Do **NOT** delete the rule for Port 22 (SSH) unless you are fully comfortable relying solely on Tailscale for SSH access.

Once you've closed these ports in OCI, your Vaultwarden instance will effectively disappear from the public internet, and only devices on your Tailscale network will be able to access it.

### 11. Post-Installation & Maintenance

* **Updates:** Vaultwarden (when self-hosted via Docker Compose) does not auto-update by default. You need to manually pull the new Docker image and recreate the container to apply updates. This gives you control over when updates happen and allows you to check for breaking changes.
    * To update Vaultwarden, navigate to `~/vaultwarden-data` and run:
        ```bash
        docker compose pull vaultwarden
        docker compose up -d
        ```
* **Backup Restore:** The `rclone` backup is intended for restoring your Vaultwarden instance to a previous state or migrating it to a new server.
    * **Restoring from `.bin` files:** The `.bin` extension simply indicates that the file's content is encrypted by `rclone crypt`. `rclone` handles the decryption process automatically when you copy files from a crypt remote. You will need the encryption password you set up for the `vaultwarden_encrypted` remote.
    * **Restoring to Official Bitwarden:** You cannot directly restore a raw Vaultwarden database backup to an official Bitwarden server. Vaultwarden uses SQLite, while official Bitwarden uses Microsoft SQL Server. To migrate, use the client-side export/import feature (Tools > Export Vault in your Bitwarden client).
    * **VM Decommission/Migration:** If your Oracle VM is decommissioned and you receive a new one, you can set up Vaultwarden on the new VM exactly as described in this guide and restore your data from your `rclone` encrypted backups.
* **Vaultwarden Database:** Vaultwarden uses SQLite as its default database, which is file-based and simplifies setup. The official Bitwarden server uses Microsoft SQL Server. It is possible to configure Vaultwarden to use PostgreSQL or MySQL/MariaDB by setting the `DATABASE_URL` environment variable in `docker-compose.yml` and running the database as another Docker container on your VM. However, for most personal or small family setups, SQLite is sufficient and more resource-efficient.
