# Linux-Based Self-Hosted Cloud Storage with Nextcloud

This guide details the process of setting up a private, self-hosted cloud storage solution, similar to Google Drive or Dropbox, using Nextcloud on an Ubuntu server. It covers installation, configuration, essential security measures (user management, firewall, SSL encryption), and automated backups.

**Features:**

*   Full control over your data.
*   Secure file storage, syncing, and sharing.
*   Expandable with Nextcloud Apps (Calendar, Contacts, Notes, etc.).
*   Accessible via web interface and desktop/mobile clients.

## Prerequisites

*   An Ubuntu server (LTS version recommended, e.g., 20.04, 22.04).
*   SSH access to the server with `sudo` privileges.
*   A registered domain name (`yourdomain.com`) pointing to your server's public IP address.
*   **(Optional but Recommended)** A static IP address for your server.

**Important:** Throughout this guide, replace placeholders like `yourdomain.com`, `your-email@example.com`, and `YourStrongPassword` with your actual values. Choose a **very strong and unique** password for the database user.

## 1. System Preparation

Ensure your Ubuntu system packages are up to date.

```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Install Core Components (Apache, MariaDB, PHP)

Install the Apache web server, MariaDB database server, PHP, and necessary PHP modules for Nextcloud.

```bash
# Install core packages
sudo apt install apache2 mariadb-server php libapache2-mod-php -y

# Install required PHP modules for Nextcloud
# xml: File format processing
# zip: Compression support
# mbstring: Multibyte string handling (for international characters)
# curl: Network requests (updates, apps)
# gd: Image processing (thumbnails)
# intl: Internationalization support
# mysql: Database connectivity
sudo apt install php-xml php-zip php-mbstring php-curl php-gd php-intl php-mysql -y

# (Optional) Install bzip2 if needed for extraction later
sudo apt install bzip2 -y
```

## 3. Database Setup

### Secure MariaDB Installation

Run the interactive script to improve the security of your MariaDB installation.

```bash
sudo mysql_secure_installation
```

Follow the prompts:
*   Set a strong root password (or use socket authentication if configured).
*   Remove anonymous users? **Yes**
*   Disallow root login remotely? **Yes**
*   Remove test database and access to it? **Yes**
*   Reload privilege tables now? **Yes**

### Create Nextcloud Database and User

Log in to the MariaDB server as root. You might be prompted for the root password you just set, or if using socket authentication, `sudo` is sufficient.

```bash
sudo mysql -u root -p
```

Execute the following SQL commands within the MariaDB prompt. **Remember to replace `YourStrongPassword` with a secure password you generate.**

```sql
-- Create the database for Nextcloud
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

-- Create a dedicated database user for Nextcloud
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'YourStrongPassword';

-- Grant all necessary privileges on the Nextcloud database to the new user
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';

-- Apply the changes
FLUSH PRIVILEGES;

-- Exit the MariaDB prompt
EXIT;
```

## 4. Download and Prepare Nextcloud

Download the latest stable version of Nextcloud, extract it to the web server directory, and set the correct ownership and permissions.

```bash
# Navigate to the parent directory for web files
cd /var/www/

# Download the latest Nextcloud release (check Nextcloud website for the current URL if needed)
sudo wget https://download.nextcloud.com/server/releases/latest.tar.bz2

# Extract the archive
sudo tar -xvf latest.tar.bz2

# Set ownership to the web server user (typically www-data on Ubuntu/Debian)
sudo chown -R www-data:www-data /var/www/nextcloud

# Set appropriate permissions for Nextcloud files and directories
sudo chmod -R 755 /var/www/nextcloud

# Clean up the downloaded archive
sudo rm latest.tar.bz2
```

## 5. Configure Apache Web Server

Create a dedicated Apache virtual host configuration file for your Nextcloud instance.

```bash
sudo vi /etc/apache2/sites-available/nextcloud.conf
```

Paste the following configuration into the file. **Adjust `ServerAdmin` and `ServerName`**.

```apacheconf
<VirtualHost *:80>
    ServerAdmin your-email@example.com
    DocumentRoot /var/www/nextcloud
    ServerName yourdomain.com

    <Directory /var/www/nextcloud/>
        # Allow .htaccess overrides for Nextcloud's rewrite rules
        AllowOverride All
        # Security: Ensure requests are granted
        Require all granted
        # Recommended options for Nextcloud
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>

    # Define custom log files
    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

Enable the new site configuration, required Apache modules, and restart Apache to apply changes.

```bash
# Enable the Nextcloud site
sudo a2ensite nextcloud.conf

# Enable necessary Apache modules (rewrite for pretty URLs, headers for security, etc.)
sudo a2enmod rewrite headers env dir mime

# Restart Apache
sudo systemctl restart apache2
```

## 6. Secure with HTTPS (Let's Encrypt SSL)

Install Certbot, the Let's Encrypt client, and obtain a free SSL certificate for your domain. Certbot will automatically modify your Apache configuration to handle HTTPS.

```bash
# Install Certbot and the Apache plugin
sudo apt install certbot python3-certbot-apache -y

# Obtain and install the SSL certificate (follow the prompts)
# Choose the redirect option when asked (usually option 2) to enforce HTTPS.
sudo certbot --apache -d yourdomain.com
```

Set up automatic renewal for the certificate using a cron job.

```bash
# Edit the root user's crontab
sudo crontab -e
```

Add the following line to check for renewal twice daily (at a random minute within the hour):

```cron
0 3,15 * * * certbot renew --quiet
```
*(The `--quiet` flag suppresses output unless there's an error.)*

## 7. Firewall Configuration

Configure the Uncomplicated Firewall (UFW) to allow web traffic on standard HTTP (80) and HTTPS (443) ports.

```bash
# Allow HTTP and HTTPS traffic
sudo ufw allow 80,443/tcp

# Allow SSH traffic (if not already enabled)
sudo ufw allow ssh

# Enable the firewall
sudo ufw enable

# Check the status
sudo ufw status
```

## 8. Install Fail2Ban (Optional but Recommended)

Fail2Ban helps protect your server from brute-force attacks by monitoring log files and banning IPs that show malicious signs.

```bash
sudo apt install fail2ban -y

# Fail2Ban starts automatically and provides basic protection for SSH out-of-the-box.
# For Nextcloud-specific rules, you might need further configuration (see Nextcloud documentation).
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

## 9. Initial Nextcloud Setup (Web Interface)

Now, complete the Nextcloud installation through your web browser.

1.  Open your web browser and navigate to `https://yourdomain.com` (use HTTPS).
2.  You will be greeted by the Nextcloud setup wizard.
3.  Create an **admin account** (choose a username and a strong password).
4.  Expand the **Storage & database** section.
5.  Select **MariaDB/MySQL** as the database type.
6.  Enter the database details you created earlier:
    *   **Database user:** `nextclouduser`
    *   **Database password:** `YourStrongPassword` (the one you set)
    *   **Database name:** `nextcloud`
    *   **Database host:** `localhost` (usually correct)
7.  Leave the **Data folder** path as the default unless you have specific reasons to change it.
8.  Click **Finish setup**.

Nextcloud will install the necessary components and configure the database. This might take a few minutes. Once done, you'll be logged into your new Nextcloud instance.

## 10. Configure Automated Backups

**Crucially, you must back up both the Nextcloud files AND the database.**

Create a directory to store backups:

```bash
sudo mkdir -p /backup
sudo chmod 700 /backup # Restrict access to root
```

Edit the root cron table to schedule daily backups:

```bash
sudo crontab -e
```

Add the following lines. **Adjust the backup directory (`/backup`) and database credentials (`nextclouduser`, `YourStrongPassword`) as needed.**

```cron
# Backup script runs daily at 2:00 AM

# Define backup directory and timestamp
BACKUP_DIR="/backup/nextcloud_$(date +\%F)"
DB_USER="nextclouduser"
DB_PASS="YourStrongPassword" # Consider using a .my.cnf file for credentials instead of plaintext
DB_NAME="nextcloud"
NC_DIR="/var/www/nextcloud"
NC_CONFIG_DIR="/var/www/nextcloud/config"

# Create timestamped directory
0 2 * * * mkdir -p $BACKUP_DIR

# Backup Nextcloud files using rsync (-a preserves permissions, ownership, etc.)
0 2 * * * rsync -a --delete $NC_DIR/ $BACKUP_DIR/files/

# Backup Nextcloud config.php separately (contains sensitive info)
0 2 * * * cp $NC_CONFIG_DIR/config.php $BACKUP_DIR/config.php

# Backup the Nextcloud database using mysqldump
0 2 * * * mysqldump --single-transaction -h localhost -u $DB_USER -p"$DB_PASS" $DB_NAME > $BACKUP_DIR/nextcloud-db.sql

# Optional: Set permissions on the backup directory (adjust as needed)
0 2 * * * chmod -R 600 $BACKUP_DIR

# Optional: Remove backups older than 7 days
0 3 * * * find /backup -type d -name 'nextcloud_*' -mtime +7 -exec rm -rf {} \;
```

**Important Backup Considerations:**

*   **Security:** Storing the database password directly in cron is not ideal. Research using a `.my.cnf` credentials file for `mysqldump` for better security.
*   **Storage:** Ensure the `/backup` location has enough disk space.
*   **Off-site Backups:** Regularly copy these backups to a separate physical location or cloud storage service (e.g., using `rclone`) for disaster recovery.
*   **Testing:** Periodically test restoring your backups to ensure they are valid and the process works. A backup that hasn't been tested is unreliable.
*   **Maintenance Mode:** For critical consistency, consider putting Nextcloud into maintenance mode *before* starting the backup and disabling it *afterwards*. This can be scripted using `occ maintenance:mode --on` and `occ maintenance:mode --off`.

## Post-Installation Recommendations

*   **Performance Tuning:**
    *   Configure PHP OPcache for better PHP performance.
    *   Set up Memory Caching (APCu, Redis, or Memcached) via Nextcloud's `config.php`. Redis is generally recommended for multi-user instances. Consult the Nextcloud documentation for details.
    *   Adjust PHP settings (e.g., `memory_limit`, `upload_max_filesize`) in `/etc/php/<version>/apache2/php.ini` as needed. Remember to restart Apache after changes.
*   **Nextcloud Apps:** Explore and install apps from the Nextcloud App Store via the web interface (click your user icon -> Apps). Popular apps include Calendar, Contacts, Notes, Collabora Online/OnlyOffice (for document editing), etc.
*   **User Management:** Create user accounts via the Nextcloud web interface (click your user icon -> Users).
*   **Security Scan:** Run the built-in security scan in Nextcloud (Settings -> Administration -> Overview) and address any warnings.
*   **Email Server Configuration:** Configure email settings (Settings -> Administration -> Basic settings) so Nextcloud can send notifications (password resets, share notifications, etc.).

## Troubleshooting

*   **Check Logs:** If you encounter issues, check the relevant log files:
    *   Nextcloud Log: Accessible via the web interface (Settings -> Administration -> Logging) or in the data directory (e.g., `/var/www/nextcloud/data/nextcloud.log`).
    *   Apache Error Log: `/var/log/apache2/nextcloud_error.log` (or `/var/log/apache2/error.log`).
    *   Apache Access Log: `/var/log/apache2/nextcloud_access.log` (or `/var/log/apache2/access.log`).
    *   System Log: `sudo journalctl -xe` or `/var/log/syslog`.
*   **Permissions:** Incorrect file/directory permissions are a common source of problems. Double-check ownership (`www-data:www-data`) and permissions (`755`) for `/var/www/nextcloud`.
*   **PHP Modules:** Ensure all required PHP modules are installed and enabled.
