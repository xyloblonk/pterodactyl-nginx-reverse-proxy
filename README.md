# Hosting Pterodactyl Panel and Multiple Websites on a Single VPS with nginx and Isolated SSH Users

Running a **Pterodactyl panel** alongside multiple websites on the same VPS is not only possible but also manageable with the right setup. This guide walks you through the entire process — from choosing the right VPS, securing it, setting up **nginx with vhosts**, configuring **per-site SSH/SFTP isolation**, and finally deploying **Pterodactyl Panel + Wings**.

## 🖥️ 1. Choosing the Right VPS & Operating System

- **OS**: Ubuntu 22.04 LTS (recommended by Pterodactyl).  
- **Virtualization**: **KVM or QEMU**. Avoid OpenVZ/LXC; Docker and Wings require proper kernel support.  
- **Specs** (minimum):  
  - Panel: 2 vCPU, 2GB RAM, 20GB SSD, 50Mbps Internet
  - Node/Wings: depends on game server requirements (4 vCPU / 8GB RAM is a common baseline)
  - Overall Recommendation: 6 vCPU (3 Physical Cores), 8GB RAM, 150GB NVMe, 200Mbps Internet
- **DNS**: Ensure you can create A records for `panel.example.com`, `site1.example.com`, etc.  

## 🔐 2. Securing the Base System

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Create admin user
sudo adduser admin
sudo usermod -aG sudo admin

# Install security tools
sudo apt install -y ufw fail2ban unattended-upgrades

# Firewall rules
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable

# Harden SSH
sudo nano /etc/ssh/sshd_config
# Set:
# PermitRootLogin no
# PasswordAuthentication no
# Subsystem sftp internal-sftp
sudo systemctl restart ssh
````

Enable unattended upgrades:

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## 🌐 3. Installing the Web Stack

```bash
# Add PHP repo
sudo add-apt-repository -y ppa:ondrej/php
sudo apt update

# Install essentials
sudo apt install -y nginx mariadb-server redis-server unzip git curl software-properties-common

# Install PHP 8.3 and extensions
sudo apt install -y php8.3 php8.3-fpm php8.3-cli php8.3-mbstring php8.3-zip \
php8.3-mysql php8.3-curl php8.3-gd php8.3-xml php8.3-bcmath
```

Install Composer:

```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

## 🗄️ 4. Database Setup

```bash
sudo mysql_secure_installation

# In MariaDB shell:
CREATE DATABASE panel;
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY 'SuperSecurePass!';
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1';
FLUSH PRIVILEGES;
```

## 📂 5. Directory Structure

```
/var/www/
  ├── pterodactyl/     # Panel files
  ├── site1/           # User site1 (chroot root)
  │    └── html/       # Site content
  ├── site2/
  │    └── html/
  └── siteN/
       └── html/
```

## 👤 6. Isolated SSH/SFTP Users

Create a restricted group:

```bash
sudo groupadd sftpusers
```

For each site (example: `site1`):

```bash
sudo useradd -g sftpusers -d /html -s /usr/sbin/nologin -M site1
sudo passwd site1

sudo mkdir -p /var/www/site1/html
sudo chown root:root /var/www/site1
sudo chmod 755 /var/www/site1
sudo chown site1:sftpusers /var/www/site1/html
```

Update SSH config (`/etc/ssh/sshd_config`):

```
Match Group sftpusers
    ChrootDirectory /var/www/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

Restart:

```bash
sudo systemctl restart ssh
```

✅ Each user is jailed into their own `/var/www/<user>`.

## 📝 7. nginx Configuration

### Panel vhost

`/etc/nginx/sites-available/pterodactyl.conf`:

```nginx
server {
    listen 80;
    server_name panel.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name panel.example.com;

    root /var/www/pterodactyl/public;
    index index.php;

    ssl_certificate /etc/letsencrypt/live/panel.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/panel.example.com/privkey.pem;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        include snippets/fastcgi-php.conf;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

### Website vhost

`/etc/nginx/sites-available/site1.conf`:

```nginx
server {
    listen 80;
    server_name site1.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name site1.example.com;

    root /var/www/site1/html;
    index index.php index.html;

    ssl_certificate /etc/letsencrypt/live/site1.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/site1.example.com/privkey.pem;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        include snippets/fastcgi-php.conf;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/pterodactyl.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/site1.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 🔒 8. HTTPS with Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d panel.example.com -d site1.example.com -d site2.example.com
```

## 🖥️ 9. Installing Pterodactyl Panel

```bash
cd /var/www/pterodactyl
curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
tar -xzvf panel.tar.gz
cp .env.example .env

# Install dependencies
COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader

# Setup environment
php artisan key:generate --force
php artisan p:environment:setup
php artisan p:environment:database
php artisan migrate --seed --force

# Create admin
php artisan p:user:make

# Permissions
sudo chown -R www-data:www-data /var/www/pterodactyl/*
```

Enable queue worker (`/etc/systemd/system/pteroq.service`):

```ini
[Unit]
Description=Pterodactyl Queue Worker
After=redis-server.service

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/pterodactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now pteroq
```

## 🐳 10. Installing Pterodactyl Wings

```bash
# Docker
curl -sSL https://get.docker.com/ | CHANNEL=stable bash
sudo systemctl enable --now docker

# Wings binary
sudo mkdir -p /etc/pterodactyl
sudo curl -L -o /usr/local/bin/wings \
  "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_amd64"
sudo chmod +x /usr/local/bin/wings
```

Get config from **Panel → Nodes → Configuration** and save as `/etc/pterodactyl/config.yml`.

Systemd unit (`/etc/systemd/system/wings.service`):

```ini
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
ExecStart=/usr/local/bin/wings
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wings
```

## 🌍 11. Reverse Proxying Wings (Optional)

`/etc/nginx/sites-available/wings.conf`:

```nginx
server {
    listen 80;
    server_name wings.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name wings.example.com;

    ssl_certificate /etc/letsencrypt/live/wings.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wings.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}
```

## ✅ 12. Final Checklist

* [x] VPS is KVM-based, secured, updated
* [x] nginx vhosts for `panel` and all sites
* [x] Let’s Encrypt certs installed
* [x] Pterodactyl Panel running with working `.env` and queue worker
* [x] Wings running and registered in Panel
* [x] SSH/SFTP users can only access their site’s `/html` folder

## 🔎 Common Issues

* **Wrong virtualization**: OpenVZ → Docker/Wings fail.
* **Chroot ownership**: must be `root:root`, not the site user.
* **Proxy headers**: missing `X-Forwarded-Proto` → login/session issues.
* **Firewall**: ensure ports 8080/2022 open if not proxied.
* **DNS**: A records must resolve before certbot will issue SSL.

## 📚 References

* [Pterodactyl Panel Docs](https://pterodactyl.io/panel/1.0/getting_started.html)
* [Pterodactyl Wings Docs](https://pterodactyl.io/wings/1.0/installing.html)
* [nginx Reverse Proxy Guide](https://nginx.org/en/docs/)
* [OpenSSH Chroot SFTP](https://man.openbsd.org/sshd_config#ChrootDirectory)
