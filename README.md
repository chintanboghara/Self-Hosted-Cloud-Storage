# Linux-Based Self-Hosted Cloud Storage with Nextcloud

This project sets up a private cloud storage solution similar to Google Drive or Dropbox using Nextcloud on an Ubuntu server. It covers installation, configuration, and security measures including user management, firewall rules, SSL encryption, and automated backups.

## Installation & Configuration

### 1. Update System

Ensure your Ubuntu system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Packages

Install Apache, MariaDB, PHP, and necessary PHP modules:

```bash
sudo apt install apache2 mariadb-server php php-mysql libapache2-mod-php php-xml php-zip php-mbstring php-curl php-gd php-intl -y
```

### 3. Secure the MariaDB Database

Run the secure installation script to harden your MariaDB setup:

```bash
sudo mysql_secure_installation
```

Follow the prompts to:
- Set a strong root password.
- Remove anonymous users.
- Disable remote root login.
- Remove test databases.

### 4. Create Nextcloud Database & User

Log into MariaDB and execute the following commands:

```sql
sudo mysql -u root -p
```

Then, within the MariaDB prompt:

```sql
CREATE DATABASE nextcloud;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'YourStrongPassword';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 5. Download & Install Nextcloud

Change to the web directory, download, extract, and set appropriate permissions:

```bash
cd /var/www/
sudo wget https://download.nextcloud.com/server/releases/latest.tar.bz2
sudo apt update
sudo apt install bzip2 -y
sudo tar -xvf latest.tar.bz2
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 755 /var/www/nextcloud
```

### 6. Configure Apache for Nextcloud

Create an Apache configuration file for Nextcloud:

```bash
sudo vi /etc/apache2/sites-available/nextcloud.conf
```

Insert the following configuration (adjust the email and domain accordingly):

```apacheconf
<VirtualHost *:80>
    ServerAdmin your-email@example.com
    DocumentRoot /var/www/nextcloud
    ServerName yourdomain.com

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

Enable the site and necessary Apache modules, then restart Apache:

```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite headers env dir mime
sudo systemctl restart apache2
```

### 7. Secure Nextcloud with SSL (Let's Encrypt)

Install Certbot and secure your site with a free SSL certificate:

```bash
sudo apt install certbot python3-certbot-apache -y
sudo certbot --apache -d yourdomain.com
```

Set up automatic SSL renewal by editing the root cron table:

```bash
sudo crontab -e
```

Add the following line:

```cron
0 3 * * * certbot renew --quiet
```

### 8. Configure Firewall & Security

Allow HTTP and HTTPS traffic, then enable UFW:

```bash
sudo ufw allow 80,443/tcp
sudo ufw enable
```

Install Fail2Ban to help prevent brute-force attacks:

```bash
sudo apt install fail2ban -y
```

### 9. Configure Automatic Backups

Set up daily backups using rsync. Edit the root cron table:

```bash
sudo crontab -e
```

Add the following line to create daily backups (adjust the backup directory as needed):

```cron
0 2 * * * rsync -a /var/www/nextcloud /backup/nextcloud_$(date +\%F)
```

## Final Setup

1. Open your web browser and navigate to:
   ```
   http://yourdomain.com
   ```
2. Follow the Nextcloud setup wizard:
   - When prompted for the database configuration, use:
     - **Database user:** nextclouduser
     - **Database password:** YourStrongPassword
     - **Database name:** nextcloud

3. Complete the configuration and begin using your self-hosted cloud storage.
