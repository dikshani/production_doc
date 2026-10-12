# Deploying WordPress on AWS EC2 + RDS (Apache, PHP-FPM, MySQL)

A practical, step-by-step guide to running WordPress on an Ubuntu EC2 instance with the database on Amazon RDS (MySQL), plus how to migrate a site between environments (staging → preprod → prod).

> All names, IPs, domains and passwords below are placeholders. Replace them with your own values and never commit real credentials.

---

## 1. Architecture

```
User ──HTTPS──> DNS ──> (ALB / EC2 public IP)
                              │
                     EC2 (Ubuntu + Apache + PHP-FPM + WordPress)
                              │  port 3306 (private)
                      RDS MySQL (private subnet)
```

**Why RDS instead of MySQL on the same server?**
Managed backups, automated patching, Multi-AZ failover and independent scaling. The web server stays stateless and easy to rebuild.

## 2. Prerequisites

- AWS account with permissions for EC2, RDS, VPC and security groups
- EC2 instance running Ubuntu 22.04 LTS (or 20.04)
- RDS MySQL instance in a **private subnet**
- Domain name with DNS access (and a TLS certificate)
- SSH key pair (`.pem`)

### Security groups (least privilege)

| Group | Inbound rule | Source |
|---|---|---|
| `web-sg` (EC2) | 80, 443 | Internet (or ALB) |
| `web-sg` (EC2) | 22 | Your office/VPN IP only |
| `db-sg` (RDS) | 3306 | `web-sg` only |

## 3. Connect to the server

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ubuntu@<EC2-PUBLIC-IP>
```

## 4. Install Apache and PHP

Recommended: **PHP-FPM** with `mpm_event`, instead of `mod_php`.
PHP-FPM lets you run different PHP versions per site, and `mod_php` allows only one version loaded in Apache at a time.

```bash
sudo apt update
sudo apt install -y apache2 \
  php-fpm php-mysql php-curl php-gd php-mbstring \
  php-xml php-xmlrpc php-soap php-intl php-zip

# Switch Apache from mod_php model to PHP-FPM
sudo a2dismod php* mpm_prefork 2>/dev/null
sudo a2enmod mpm_event proxy_fcgi setenvif rewrite headers
sudo a2enconf php*-fpm          # e.g. php8.1-fpm
sudo systemctl restart apache2
```

Check versions:

```bash
php -v
apache2 -v
```

> PHP 7.4 reached end of life in Nov 2022. Prefer PHP 8.1+ for security updates, and confirm plugin/theme compatibility first.

## 5. Create the database on RDS

1. RDS → Create database → MySQL (or MariaDB), pick instance size, enable automated backups, place it in a private subnet, attach `db-sg`.
2. From the EC2 server, connect and create the DB and user:

```bash
sudo apt install -y mysql-client
mysql -h <RDS-ENDPOINT> -u admin -p
```

```sql
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wp_user'@'%' IDENTIFIED BY '<STRONG_PASSWORD>';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'%';
FLUSH PRIVILEGES;
```

Test connectivity (if this fails, check the security group first):

```bash
mysql -h <RDS-ENDPOINT> -u wp_user -p wordpress -e "SELECT 1;"
```

## 6. Download WordPress

```bash
cd /tmp
curl -O https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mv wordpress /var/www/html/wordpress
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo find /var/www/html/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/html/wordpress -type f -exec chmod 644 {} \;
```

## 7. Configure WordPress

```bash
cd /var/www/html/wordpress
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

```php
define('DB_NAME',     'wordpress');
define('DB_USER',     'wp_user');
define('DB_PASSWORD', '<STRONG_PASSWORD>');
define('DB_HOST',     '<RDS-ENDPOINT>');
```

Also replace the authentication keys and salts with fresh values from
`https://api.wordpress.org/secret-key/1.1/salt/`.

**Better practice:** do not keep the DB password in plain text in the repo or AMI. Load it from AWS Secrets Manager / SSM Parameter Store at deploy time, and keep `wp-config.php` out of Git.

## 8. Configure the Apache virtual host

```bash
sudo nano /etc/apache2/sites-available/wordpress.conf
```

```apache
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/html/wordpress

    <Directory /var/www/html/wordpress>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog  ${APACHE_LOG_DIR}/wordpress-error.log
    CustomLog ${APACHE_LOG_DIR}/wordpress-access.log combined
</VirtualHost>
```

Enable it:

```bash
sudo a2dissite 000-default.conf
sudo a2ensite wordpress.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

`AllowOverride All` + `a2enmod rewrite` (done in step 4) are required for `.htaccess` rules and pretty permalinks.

## 9. HTTPS and DNS

- Point the domain's DNS record to the server (or load balancer).
- Terminate TLS either on an **ALB with an ACM certificate** or on Apache using Let's Encrypt:

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d example.com -d www.example.com
```

- Redirect HTTP → HTTPS and set `siteurl` / `home` to the `https://` URL.

Set up DNS and TLS **before** importing a migrated site, so URLs resolve correctly.

## 10. Verify

Open `http://<EC2-PUBLIC-IP>` (or the domain) and finish the WordPress install screen. Then check:

```bash
sudo systemctl status apache2
sudo systemctl status php8.1-fpm
sudo tail -f /var/log/apache2/wordpress-error.log
```

## 11. Migrating a site between environments

Example: staging → preprod → prod.

### Option A: All-in-One WP Migration plugin (simplest)

1. **Source site:** install and activate the plugin → *All-in-One WP Migration → Export → Export To → File*.
2. **Target site:** install the same plugin → *Import → File* → upload the export → *Proceed*.
3. Re-save Permalinks (*Settings → Permalinks → Save*).

The export includes the database, themes, plugins and media. The free version has an upload size limit, which you can raise in `php.ini` (`upload_max_filesize`, `post_max_size`) or avoid with Option B.

### Option B: WP-CLI + rsync (better for large or repeatable moves)

```bash
# On source
wp db export site.sql
rsync -avz wp-content/ user@target:/var/www/html/wordpress/wp-content/

# On target
wp db import site.sql
wp search-replace 'https://staging.example.com' 'https://www.example.com' --all-tables
wp cache flush
```

### Migration checklist

- [ ] Take a backup / RDS snapshot of the **target** first
- [ ] DNS and TLS ready on the target
- [ ] Update site URL (`search-replace`)
- [ ] Re-save permalinks, check `.htaccess`
- [ ] Clear cache / CDN
- [ ] Test login, forms, media, and key pages
- [ ] Have a rollback plan (snapshot restore)

## 12. Common issues

| Symptom | Likely cause | Fix |
|---|---|---|
| "Error establishing a database connection" | Wrong credentials, or RDS security group blocks EC2 | Test with `mysql -h`; check `db-sg` allows `web-sg` on 3306 |
| Pretty permalinks return 404 | `mod_rewrite` off or `AllowOverride None` | `a2enmod rewrite`; set `AllowOverride All`; reload Apache |
| White screen / 500 error | PHP error or plugin conflict | Check Apache/PHP-FPM error logs; disable plugins via WP-CLI |
| Cannot upload media | File ownership/permissions | `chown -R www-data:www-data wp-content` |
| Import fails with size error | Upload limit | Raise `upload_max_filesize` / `post_max_size`, or use WP-CLI |
| Mixed content after HTTPS | Old `http://` URLs in DB | `wp search-replace` |

## 13. Hardening and improvements

- Store secrets in Secrets Manager / Parameter Store
- Use HTTPS only, plus security headers
- Keep RDS private; restrict SSH to VPN/office IPs (or use SSM Session Manager)
- Enable automated RDS backups and test restores
- Keep WordPress core, plugins and PHP patched
- Ship logs to CloudWatch and set alarms (CPU, disk, 5xx, DB connections)
- Automate the whole setup with Terraform (infra) and Ansible (configuration) so every environment is identical
- Deploy themes through a Git workflow rather than manual file edits

## 14. Quick reference

```bash
sudo systemctl restart apache2          # restart web server
sudo systemctl restart php8.1-fpm       # restart PHP
sudo apache2ctl configtest              # validate Apache config
sudo tail -f /var/log/apache2/*.log     # watch logs
sudo a2ensite|a2dissite <site>          # enable/disable a vhost
sudo a2enmod|a2dismod <module>          # enable/disable a module
```
