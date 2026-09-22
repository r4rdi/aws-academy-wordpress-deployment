<div align="center">

```text
 █████╗ ██╗    ██╗███████╗    ██████╗ ███████╗██████╗ ██╗      ██████╗ ██╗   ██╗███╗   ███╗███████╗███╗   ██╗████████╗
██╔══██╗██║    ██║██╔════╝    ██╔══██╗██╔════╝██╔══██╗██║     ██╔═══██╗╚██╗ ██╔╝████╗ ████║██╔════╝████╗  ██║╚══██╔══╝
███████║██║ █╗ ██║███████╗    ██║  ██║█████╗  ██████╔╝██║     ██║   ██║ ╚████╔╝ ██╔████╔██║█████╗  ██╔██╗ ██║   ██║   
██╔══██║██║███╗██║╚════██║    ██║  ██║██╔══╝  ██╔═══╝ ██║     ██║   ██║  ╚██╔╝  ██║╚██╔╝██║██╔══╝  ██║╚██╗██║   ██║   
██║  ██║╚███╔███╔╝███████║    ██████╔╝███████╗██║     ███████╗╚██████╔╝   ██║   ██║ ╚═╝ ██║███████╗██║ ╚████║   ██║   
╚═╝  ╚═╝ ╚══╝╚══╝ ╚══════╝    ╚═════╝ ╚══════╝╚═╝     ╚══════╝ ╚═════╝    ╚═╝   ╚═╝     ╚═╝╚══════╝╚═╝  ╚═══╝   ╚═╝   
```

# 🌐 AWS Cloud Infrastructure & WordPress Non-Docker Deployment <br>
**Panduan End-to-End: VPC, EC2 Ubuntu Server, PuTTY (.ppk), LAMP, dan Nginx Reverse Proxy (HTTPS)**

</div>

---

## 📌 Spesifikasi Arsitektur Sistem

| Komponen | Detail Konfigurasi |
| :--- | :--- |
| **Cloud Provider** | Amazon Web Services (AWS Academy / AWS Labs) |
| **VPC / CIDR** | `wordpress-vpc` / `10.0.0.0/16` |
| **Subnets** | Public: `10.0.0.0/20`, `10.0.16.0/20` \| Private: `10.0.128.0/20`, `10.0.144.0/20` |
| **Compute Instance** | AWS EC2 Instance (`t3.micro`) |
| **Operating System** | Ubuntu Server (26.04 LTS / Modern LTS) |
| **Authentication** | Custom RSA Private Key (`.ppk` for PuTTY) |
| **Web Stack (Backend)** | Apache2 (Port `8080`), MariaDB, PHP 8.x (LAMP) |
| **Reverse Proxy (Frontend)** | Nginx (Port `80` redirect to `443`), Self-Signed SSL/TLS |
| **Public IP Target** | `98.93.6.165` (Contoh Aktif) |

---

## 🛠️ Tahapan Konfigurasi Lengkap

### Langkah 1: Pembangunan AWS VPC & Networking
1. Masuk ke **AWS Management Console > VPC**.
2. Klik **Create VPC** -> Pilih **VPC and more** (`wordpress-prefix`).
3. Set IPv4 CIDR: `10.0.0.0/16`, 2 Availability Zones (`us-east-1a`, `us-east-1b`), 2 Public Subnets, 2 Private Subnets, NAT Gateways Zonal.
4. Buat **Security Group** (`wordpress-sg`) dengan Inbound Rules:
   * SSH (TCP 22) -> `0.0.0.0/0`
   * HTTP (TCP 80) -> `0.0.0.0/0`
   * HTTPS (TCP 443) -> `0.0.0.0/0`

### Langkah 2: Peluncuran EC2 Instance (Ubuntu Server)
1. Buka **EC2 Console > Launch instances**.
2. Name: `wordpress-server`, AMI: **Ubuntu Server (26.04 LTS)**.
3. Instance Type: `t3.micro`.
4. Key Pair: Pilih custom RSA key pair (yang dikonversi/dipersiapkan format `.ppk`).
5. Network: Pilih `wordpress-vpc`, Public Subnet, Auto-assign public IP: `Enable`, Security Group: `wordpress-sg`.
6. Launch instance dan catat Public IP (misal: `98.93.6.165`).

### Langkah 3: Koneksi SSH via PuTTY (.ppk)
1. Buka aplikasi **PuTTY**.
2. Masukkan Host Name: `ubuntu@98.93.6.165`.
3. Navigasi ke **Connection > SSH > Auth > Credentials**.
4. Pilih file kunci privat `.ppk` kustom Anda.
5. Klik **Open** dan terima *Security Alert*.

### Langkah 4: Instalasi LAMP Stack (Apache, MariaDB, PHP)
```bash
# Update sistem
sudo apt update -y

# Install Apache, MariaDB, dan ekstensi PHP
sudo apt install apache2 mariadb-server php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y

# Aktifkan service
sudo systemctl start apache2 && sudo systemctl enable apache2
sudo systemctl start mariadb && sudo systemctl enable mariadb
```

### Langkah 5: Konfigurasi Database WordPress
```bash
sudo mysql -u root
```
Di dalam prompt MySQL:
```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'PasswordKuat123!';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Langkah 6: Deployment File WordPress & Virtual Host Awal
```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xvzf latest.tar.gz
sudo mv wordpress/* /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```
Buat file virtual host awal (`/etc/apache2/sites-available/wordpress.conf`):
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ServerName _
    <Directory /var/www/html/>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Aktifkan konfigurasi:
```bash
sudo a2ensite wordpress.conf
sudo a2dissite 000-default.conf
sudo rm -f /var/www/html/index.html
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### Langkah 7: Migrasi Apache ke Port 8080 (Persiapan Reverse Proxy Nginx)
Agar port 80/443 bisa dikelola Nginx:
1. Edit `/etc/apache2/ports.conf`, ubah `Listen 80` menjadi `Listen 8080`.
2. Edit `/etc/apache2/sites-available/wordpress.conf`, ubah `<VirtualHost *:80>` menjadi `<VirtualHost *:8080>`.
3. Restart Apache & verifikasi:
```bash
sudo systemctl restart apache2
sudo ss -tulpn | grep 8080
```

### Langkah 8: Instalasi Nginx & Self-Signed SSL Certificate
```bash
sudo apt install nginx openssl -y
sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx.key \
  -out /etc/nginx/ssl/nginx.crt \
  -subj "/C=ID/ST=Jakarta/L=Jakarta/O=AWSLab/CN=98.93.6.165"
```

### Langkah 9: Konfigurasi Nginx Reverse Proxy (`/etc/nginx/conf.d/wordpress-https.conf`)
```nginx
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
    }
}
```
Uji dan muat ulang Nginx:
```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
```

### Langkah 10: Hardening `wp-config.php` untuk Mengatasi Redirect Loop / SSL Termination
Buka `/var/www/html/wp-config.php` dan tambahkan sebelum `That's all, stop editing!`:
```php
define('WP_HOME', 'https://98.93.6.165');
define('WP_SITEURL', 'https://98.93.6.165');
define('FORCE_SSL_ADMIN', true);
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}
```

---

## ✅ Verifikasi & Hasil Akhir
* **Akses Situs Publik**: `https://98.93.6.165` (Aman dengan HTTPS / Self-signed certificate warning normal).
* **Akses Admin Dasbor**: `https://98.93.6.165/wp-admin/` (Berfungsi tanpa infinite redirect loop `ERR_TOO_MANY_REDIRECTS`).
* **Backend Stack Sync**: Nginx (443 SSL termination) -> Apache (8080 backend PHP rendering) -> MariaDB.

---
*Dokumentasi disusun untuk keperluan praktikum & evaluasi AWS Cloud Infrastructure.*
