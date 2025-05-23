
# Full WordPress HA Setup with Internal CDN and Disaster Recovery

This guide describes how to deploy a 6-node highly available WordPress cluster on Linux with NGINX, using an internal CDN and a built-in disaster recovery plan.

---

## 🔧 Node Architecture

| Node | Hostname       | Purpose                         |
|------|----------------|---------------------------------|
| 1    | web1.local     | WordPress Web Node              |
| 2    | web2.local     | WordPress Web Node              |
| 3    | cdn1.local     | Internal CDN (Primary)          |
| 4    | cdn2.local     | Internal CDN (Replica)          |
| 5    | db1.local      | MySQL/MariaDB Primary           |
| 6    | db2.local      | MySQL/MariaDB Replica/Backup    |

---

## 📦 Installation (All Nodes)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx php-fpm php-mysql mariadb-client rsync curl unzip git ufw
```

---

## 🌐 Web Node Setup (web1 + web2)

### Install WordPress
```bash
cd /var/www/
sudo curl -O https://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo mv wordpress mysite
sudo chown -R www-data:www-data mysite
```

### Configure NGINX (example for web1)
```nginx
server {
    listen 80;
    server_name web1.local;

    root /var/www/mysite;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }
}
```

---

## 🗂 Internal CDN Setup

### On cdn1.local:
```bash
sudo mkdir -p /var/cdn/uploads
sudo chown www-data:www-data /var/cdn/uploads
echo "/var/cdn/uploads  *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

### On web1.local and web2.local:
```bash
sudo apt install nfs-common
sudo mkdir -p /var/www/mysite/wp-content/uploads
sudo mount cdn1.local:/var/cdn/uploads /var/www/mysite/wp-content/uploads
echo "cdn1.local:/var/cdn/uploads /var/www/mysite/wp-content/uploads nfs defaults 0 0" | sudo tee -a /etc/fstab
```

---

## 🧩 Serve Media via CDN

In `functions.php` of your theme:
```php
add_filter('upload_dir', 'force_cdn_upload_url');
function force_cdn_upload_url($upload) {
    $upload['baseurl'] = 'http://cdn1.local/wp-content/uploads';
    return $upload;
}
```

---

## 🧪 OPTIONAL: Remove /uploads on Web Nodes (Safe Version)

Once mounted via NFS:
```bash
sudo umount /var/www/mysite/wp-content/uploads
sudo rm -rf /var/www/mysite/wp-content/uploads
sudo mkdir /var/www/mysite/wp-content/uploads
sudo mount cdn1.local:/var/cdn/uploads /var/www/mysite/wp-content/uploads
```
Now uploads do not touch the local disk — they’re fully offloaded to CDN.

---

## 🛢 Database Setup

### On db1.local (Primary):
```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```

Edit config:
```
bind-address = 0.0.0.0
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
```

Create replication user:
```sql
CREATE USER 'replica'@'%' IDENTIFIED BY 'yourpassword';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';
```

Get status:
```sql
SHOW MASTER STATUS;
```

---

### On db2.local (Replica):
```bash
sudo apt install mariadb-server -y
```

Edit config:
```
server-id = 2
relay-log = /var/log/mysql/mysql-relay-bin
```

Start replication:
```sql
CHANGE MASTER TO
  MASTER_HOST='db1.local',
  MASTER_USER='replica',
  MASTER_PASSWORD='yourpassword',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS= 123;
START SLAVE;
```

---

## 🔄 CDN Sync (cdn1 → cdn2)

Add to cron on cdn1:
```bash
crontab -e
```

```bash
0 * * * * rsync -avz /var/cdn/uploads/ user@cdn2.local:/var/cdn/uploads/
```

---

## 🔥 Disaster Recovery Steps

### Web Node Failure
- Load balancer reroutes to surviving node
- Rebuild using Git:
```bash
git clone https://your.repo.git /var/www/mysite
```

---

### Internal CDN Failure
- Update WordPress to use `cdn2.local` instead of `cdn1.local`:
```php
$upload['baseurl'] = 'http://cdn2.local/wp-content/uploads';
```

- Update DB links:
```sql
UPDATE wp_options SET option_value = REPLACE(option_value, 'cdn1.local', 'cdn2.local') WHERE option_name IN ('siteurl', 'home');
```

---

### Database Failure (db1.local)
- Promote db2 to primary:
```sql
STOP SLAVE;
RESET SLAVE ALL;
```

- Update `wp-config.php`:
```php
define('DB_HOST', 'db2.local');
```

---

## 🧰 Backup and Monitoring

|  Component | Method               | Frequency       |
|------------|----------------------|-----------------|
| Database   | `mysqldump`          | Daily           |
| Binlogs    | Binary logging       | Continuous      |
| CDN        | `rsync`              | Hourly          |
| WP Files   | Git                  | On Change       |
| Monitoring | Prometheus + Grafana | 24/7            |
