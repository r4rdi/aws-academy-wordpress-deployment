<div align="center">

```text
 █████╗ ██╗    ██╗███████╗    ██████╗ ███████╗██████╗ ██╗      ██████╗ ██╗   ██╗
██╔══██╗██║    ██║██╔════╝    ██╔══██╗██╔════╝██╔══██╗██║     ██╔═══██╗╚██╗ ██╔╝
███████║██║ █╗ ██║███████╗    ██║  ██║█████╗  ██████╔╝██║     ██║   ██║ ╚████╔╝ 
██╔══██║██║███╗██║╚════██║    ██║  ██║██╔══╝  ██╔═══╝ ██║     ██║   ██║  ╚██╔╝  
██║  ██║╚███╔███╔╝███████║    ██████╔╝███████╗██║     ███████╗╚██████╔╝   ██║   
╚═╝  ╚═╝ ╚══╝╚══╝ ╚══════╝    ╚═════╝ ╚══════╝╚═╝     ╚══════╝ ╚═════╝    ╚═╝   
                                                                                                              
```

# AWS Cloud Infrastructure & WordPress Non-Docker Deployment 



**Panduan End-to-End: VPC, EC2 Ubuntu Server, PuTTY (.ppk), LAMP, dan Nginx Reverse Proxy (HTTPS)**

---

## 📌 Spesifikasi Arsitektur Sistem



| Komponen | Detail Konfigurasi |
| --- | --- |
| **Cloud Provider** | Amazon Web Services (AWS Academy / AWS Labs)

 |
| **VPC / CIDR** | `wordpress-vpc` / `10.0.0.0/16`<br> |
| **Subnets** | Public: `10.0.0.0/20`, `10.0.16.0/20` | Private: `10.0.128.0/20`, `10.0.144.0/20`<br> |
| **Compute Instance** | AWS EC2 Instance (`t3.micro`)

 |
| **Operating System** | Ubuntu Server (26.04 LTS / Modern LTS)

 |
| **Authentication** | Custom RSA Private Key (`.ppk` for PuTTY)

 |
| **Web Stack (Backend)** | Apache2 (Port `8080`), MariaDB, PHP 8.x (LAMP)

 |
| **Reverse Proxy (Frontend)** | Nginx (Port `80` redirect to `443`), Self-Signed SSL/TLS

 |
| **Public IP Target** | `98.93.6.165` (Contoh Aktif)

 |

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


6. Klik **Launch instance** dan catat Public IP yang didapatkan (misal: `98.93.6.165`).



### Langkah 3: Koneksi SSH via PuTTY (.ppk)



1. Buka aplikasi **PuTTY** di komputer lokal.


2. Masukkan Host Name: `ubuntu@98.93.6.165`.


3. Navigasi menu sebelah kiri ke **Connection > SSH > Auth > Credentials**.


4. Pada kolom *Private key file for authentication*, klik **Browse** dan pilih file `.ppk` Anda.


5. Klik **Open** dan terima prompt *PuTTY Security Alert*.



---

### Langkah 4: Instalasi LAMP Stack (Apache, MariaDB, PHP)



Jalankan perintah berikut di terminal SSH untuk memperbarui paket dan memasang dependensi LAMP Stack:

```bash
# Update direktori paket sistem
sudo apt update -y && sudo apt upgrade -y

# Install Apache2, MariaDB Server, dan pustaka PHP yang dibutuhkan WordPress
sudo apt install apache2 mariadb-server php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip nano -y

# Aktifkan dan pastikan layanan Apache & MariaDB berjalan otomatis saat booting
sudo systemctl start apache2
sudo systemctl enable apache2
sudo systemctl start mariadb
sudo systemctl enable mariadb

```

---

### Langkah 5: Konfigurasi Database MariaDB untuk WordPress



Masuk ke CLI MariaDB sebagai root:

```bash
sudo mysql -u root

```

Di dalam prompt MySQL (`MariaDB [(none)]>`), jalankan perintah SQL berikut baris demi baris:

```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'PasswordKuat123!';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;

```

---

### Langkah 6: Unduh File WordPress & Buat Virtual Host Apache



1. **Unduh dan Ekstrak WordPress:**


```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xvzf latest.tar.gz
sudo mv wordpress/* /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/

```

2. **Buat file konfigurasi Virtual Host Apache:**


Jalankan perintah `nano` untuk membuat file baru:



```bash
sudo nano /etc/apache2/sites-available/wordpress.conf

```

Tempelkan (*paste*) kode konfigurasi berikut:

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

> **Petunjuk Simpan di Nano:** Tekan `Ctrl + O`, lalu `Enter` untuk menyimpan. Tekan `Ctrl + X` untuk keluar dari editor.

3. **Aktifkan Situs dan Modul Apache:**


```bash
# Aktifkan konfigurasi wordpress.conf
sudo a2ensite wordpress.conf

# Nonaktifkan file default bawaan Apache
sudo a2dissite 000-default.conf

# Hapus file index.html bawaan jika ada
sudo rm -f /var/www/html/index.html

# Aktifkan modul rewrite Apache untuk mendukung URL Permalink WordPress
sudo a2enmod rewrite

# Restart layanan Apache
sudo systemctl restart apache2

```

---

### Langkah 7: Migrasi Apache ke Port 8080 (Persiapan Reverse Proxy)



Agar port 80 dan 443 dapat digunakan oleh Nginx sebagai Reverse Proxy, ubah port Apache ke `8080`.

1. **Ubah file `ports.conf` Apache:**


```bash
sudo nano /etc/apache2/ports.conf

```

Cari baris `Listen 80` dan ubah nilainya menjadi:

```apache
Listen 8080

```

*(Tekan `Ctrl + O` -> `Enter` -> `Ctrl + X`)*

2. **Ubah file Virtual Host `wordpress.conf`:**


```bash
sudo nano /etc/apache2/sites-available/wordpress.conf

```

Ubah baris pertama dari `<VirtualHost *:80>` menjadi:

```apache
<VirtualHost *:8080>

```

*(Tekan `Ctrl + O` -> `Enter` -> `Ctrl + X`)*

3. **Restart dan Verifikasi Port Apache:**


```bash
sudo systemctl restart apache2
sudo ss -tulpn | grep 8080

```

---

### Langkah 8: Instalasi Nginx & Pembuatan Certificate SSL Self-Signed



Jalankan perintah berikut untuk menginstal Nginx, OpenSSL, dan membuat direktori serta sertifikat SSL:

```bash
# Install Nginx dan OpenSSL
sudo apt install nginx openssl -y

# Buat folder khusus SSL Nginx
sudo mkdir -p /etc/nginx/ssl

# Generate sertifikat SSL Self-Signed (Berlaku 365 hari)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx.key \
  -out /etc/nginx/ssl/nginx.crt \
  -subj "/C=ID/ST=Jakarta/L=Jakarta/O=AWSLab/CN=98.93.6.165"

```

---

### Langkah 9: Konfigurasi Nginx Reverse Proxy (`/etc/nginx/conf.d/wordpress-https.conf`)



1. **Buat file konfigurasi Nginx Reverse Proxy:**


```bash
sudo nano /etc/nginx/conf.d/wordpress-https.conf

```

2. **Isikan konfigurasi berikut ke dalam editor:**


```nginx
server {
    listen 80;
    server_name _;
    # Redirect otomatis semua lalu lintas HTTP (80) ke HTTPS (443)
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;

    # Jalur sertifikat SSL yang dibuat pada Langkah 8
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        # Meneruskan traffic ke Apache backend di port 8080
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
    }
}

```

*(Tekan `Ctrl + O` -> `Enter` -> `Ctrl + X`)*

3. **Uji Konfigurasi dan Jalankan Nginx:**


```bash
# Hapus symlink server default Nginx
sudo rm -f /etc/nginx/sites-enabled/default

# Tes sintaks konfigurasi Nginx
sudo nginx -t

# Jika output 'syntax is ok', restart dan aktifkan Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx

```

---

### Langkah 10: Pembuatan & Konfigurasi File `wp-config.php` (Mencegah Loop SSL)



1. **Buat file `wp-config.php` dari file sample:**

```bash
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo nano /var/www/html/wp-config.php

```

2. **Sesuaikan kredensial Database:**
Cari baris berikut dan sesuaikan nama DB, Username, serta Password sesuai Langkah 5:



```php
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'PasswordKuat123!' );
define( 'DB_HOST', 'localhost' );

```

3. **Tambahkan Parameter SSL Termination:**
Gulir ke bagian paling bawah file, temukan baris `/* That's all, stop editing! Happy publishing. */`. **Tepat di atas baris tersebut**, tambahkan kode berikut:



```php
define('WP_HOME', 'https://98.93.6.165');
define('WP_SITEURL', 'https://98.93.6.165');
define('FORCE_SSL_ADMIN', true);
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}

```

*(Tekan `Ctrl + O` -> `Enter` -> `Ctrl + X`)*

4. **Atur Ulang Perizinan File:**


```bash
sudo chown -R www-data:www-data /var/www/html/

```

---

## ✅ Verifikasi & Hasil Akhir



1. **Akses Situs Publik:**


Buka browser dan akses URL `[https://98.93.6.165](https://98.93.6.165)`. Browser akan menampilkan peringatan *SSL Self-Signed Warning* (hal ini normal karena menggunakan SSL buatan sendiri). Klik *Advanced > Proceed* untuk melanjutkan ke halaman pemasangan WordPress.


2. **Pengisian Instalasi Web:**
Isi judul situs, username admin, password admin, dan email untuk menyelesaikan pemasangan WordPress.
3. **Akses Dashboard Admin:**


Buka `[https://98.93.6.165/wp-admin/](https://98.93.6.165/wp-admin/)`. Pastikan Anda dapat masuk tanpa mengalami error `ERR_TOO_MANY_REDIRECTS`.


4. **Validasi Alur Kerja Stack:**

* Client HTTP (80) -> Auto Redirect ke HTTPS (443) Nginx.


* Nginx (443 SSL Termination) -> Proxy Pass ke Apache Backend (`127.0.0.1:8080`).


* Apache (8080 PHP Processing) -> Query Database MariaDB.





---

*Dokumentasi disempurnakan untuk keperluan praktikum & evaluasi AWS Cloud Infrastructure.*
